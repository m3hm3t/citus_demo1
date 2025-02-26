# How to use Citus 13’s expanded functionality on distributed partitioned tables:

1. Specifying an **access method** on a partitioned table.  
2. Creating an **identity column** in the partitioned table.  
3. Adding an **exclusion constraint**.  
4. Demonstrating how these changes propagate when you **distribute** and **attach** new partitions.  
5. Seeing how an insert might fail because of the new exclusion constraint.  

In this demo, we’ll create a table for “orders,” partitioned by a date column, then distribute it, attach more partitions, and see these new features in action.

---


## Step 1: Create a Partitioned Table with a Custom Access Method and Identity Column

Let’s create a **partitioned table** named `orders_partitioned` that is partitioned by range on `order_date`. We’ll also:

- Use `USING columnar` to specify the **columnar** access method for the table.  
- Define an **identity column** `order_id` with a custom start and increment.  

> Note: The syntax `PARTITION BY RANGE (order_date) USING columnar` is the Postgres 17 way to specify an access method for partitioned tables. Citus supports propagating this to worker nodes once the table is distributed.

```sql
CREATE TABLE orders_partitioned
(
    order_id   bigint GENERATED BY DEFAULT AS IDENTITY (START WITH 100 INCREMENT BY 100),
    order_date date   NOT NULL,
    amount     numeric(10,2),
    customer   text
)
PARTITION BY RANGE (order_date)
USING columnar;
```

---

## Step 2: Create an Initial Partition

We’ll create our first partition covering January 2025:

```sql
CREATE TABLE orders_jan_2025
    PARTITION OF orders_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

---

## Step 3: Distribute the Table

Now that `orders_partitioned` is a partitioned table, we turn it into a **distributed partitioned table** by calling Citus’s `create_distributed_table` on the **parent**:

```sql
SELECT create_distributed_table('orders_partitioned', 'order_date');
```

> **Note**: We used `order_date` as the distribution column because that’s the column we are partitioning on.

---

## Step 4: Create Another Partition

Let’s create a second partition for February 2025. Since the parent is now a **distributed partitioned table**, this child will automatically be distributed across the worker nodes:

```sql
CREATE TABLE orders_feb_2025
    PARTITION OF orders_partitioned
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

---

## Step 5: Alter the Access Method on the Parent

We can change the access method for the parent partitioned table at any time. Existing partitions won’t be changed automatically, but **future partitions** will inherit the new access method.

```sql
ALTER TABLE orders_partitioned SET ACCESS METHOD heap;
```

So now:

- `orders_partitioned` itself is `heap`  
- `orders_jan_2025` (old partition) remains `columnar`  
- `orders_feb_2025` (old partition) remains `columnar`  
- Any newly created partitions will become `heap`.

---

## Step 6: Add an Exclusion Constraint

Let’s say we want to forbid inserting two rows that have the exact same `order_date` and the same `order_id`. We can do this by adding an **exclusion constraint**:

```sql
ALTER TABLE orders_partitioned 
ADD CONSTRAINT orders_excl 
EXCLUDE USING btree (order_date WITH =, order_id WITH =);
```

This constraint is added to:

- The **parent** table (`orders_partitioned`),  
- All **existing** partitions (`orders_jan_2025`, `orders_feb_2025`),  
- All **future** partitions you attach.

---

## Step 7: Attach Another Partition (Inherits Identity Column & Exclusion Constraint)

Let’s add a partition for March 2025. We’ll create the table first and then attach it to the parent. Because the parent has an identity column and an exclusion constraint, the new child will automatically **inherit** both.

```sql
CREATE TABLE orders_mar_2025 (
    order_id   bigint NOT NULL,
    order_date date,
    amount     numeric(10,2),
    customer   text
);

ALTER TABLE orders_partitioned
    ATTACH PARTITION orders_mar_2025 
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
```

---

## Step 8: Verify Inheritance of Identity & Exclusion Constraint

### 8a. Identity Column

Check `pg_attribute` to see that each child partition inherits the generated identity:

```sql
SELECT attrelid::regclass AS table_name, attname, attidentity
FROM pg_attribute
WHERE attname = 'order_id' AND attidentity = 'd'
ORDER BY table_name;
```

You should see something like:

```
     table_name       | attname  | attidentity
----------------------+----------+-------------
 orders_partitioned   | order_id | d
 orders_jan_2025      | order_id | d
 orders_feb_2025      | order_id | d
 orders_mar_2025      | order_id | d
(4 rows)
```

### 8b. Access Methods

Check each table’s `amname`:

```sql
SELECT relname, amname
FROM pg_class c
LEFT JOIN pg_am am ON (c.relam = am.oid)
WHERE relname IN ('orders_partitioned', 'orders_jan_2025', 'orders_feb_2025', 'orders_mar_2025')
ORDER BY relname;
```

You’ll see that the original parent and newly attached partition use `heap`:

```
       relname             |  amname
--------------------------+----------
 orders_partitioned       | heap
 orders_jan_2025          | columnar
 orders_feb_2025          | columnar
 orders_mar_2025          | heap
(4 rows)
```

### 8c. Exclusion Constraints

Check the constraints:

```sql
SELECT conname
FROM pg_constraint
WHERE conname LIKE '%excl%'
ORDER BY conname;
```

Example output:

```
         conname
-------------------------
 orders_excl
 orders_jan_2025_excl
 orders_feb_2025_excl
 orders_mar_2025_excl
(4 rows)
```

---

## Step 9: Test an Insert that Violates the Exclusion Constraint

Now, let’s see the new exclusion constraint in action. If you try to insert **two identical** `(order_date, order_id)` combos, the second one will fail.

```sql
-- First insert succeeds
INSERT INTO orders_partitioned (order_date, amount, customer) 
VALUES ('2025-01-10', 123.45, 'Alice');

-- Try a second insert with the same date and identity (order_id 100) 
INSERT INTO orders_partitioned (order_id, order_date, amount, customer) 
VALUES (100, '2025-01-10', 200.00, 'Bob');
```

- The **first** insert uses the default generated identity (which starts at 100).  
- The **second** insert tries to explicitly use `order_id = 100` on the **same** `order_date`.  
- This will violate the `EXCLUDE (order_date WITH =, order_id WITH =)`, causing an **error** like:
  
  ```
  ERROR:  conflicting key value violates exclusion constraint "orders_jan_2025_excl"
  DETAIL:  Key (order_date, order_id)=(2025-01-10, 100) conflicts with existing row.
  ```

---


# Putting It All Together

1. We created a **partitioned** table (`orders_partitioned`) with a **columnar** access method and an **identity** column.  
2. We **distributed** the partitioned table with Citus.  
3. We **altered** the access method to `heap`, affecting future partitions.  
4. We added an **exclusion constraint** to the parent (and all partitions).  
5. We **attached** a new partition (`orders_mar_2025`), which automatically inherited the identity column, the exclusion constraint, and the new `heap` access method.  
6. We **tested** that the constraint indeed prevents inserts with conflicting values.  
