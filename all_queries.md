# SQL Notes: DML, DQL, Joins, Aggregation, Subqueries, CTEs, Window Functions

Practice schema used throughout: `customers`, `categories`, `products`, `orders`, `order_items`, `employees`, `departments`, `payments`.

---

## 1. DML: Data Manipulation Language

`INSERT`, `UPDATE`, `DELETE` are DML commands.

### INSERT

```sql
INSERT INTO customers (first_name, last_name, email, phone, city, country)
VALUES ('John', 'Doe', 'john.doe@gmail.com', '9999999999', 'Mumbai', 'India');

-- Multiple rows in one statement
INSERT INTO categories (category_name, description)
VALUES
  ('Dairy', 'Dairy products'),
  ('Gym', 'Gym products'),
  ('Cold drinks', 'cold drinks description');
```

### UPDATE

```sql
UPDATE products SET stock_quantity = 50 WHERE product_id = 1;   -- one row
UPDATE products SET is_active = FALSE;                          -- ALL rows (no WHERE!)
UPDATE products SET price = price + 100;                        -- ALL rows
UPDATE products SET price = price + 200 WHERE product_id = 3;   -- one row
```

> **Warning:** an `UPDATE` or `DELETE` without `WHERE` affects every row in the table.

### DELETE

```sql
DELETE FROM orders WHERE order_id = 1;
```

> If `order_items` references the order, the delete may fail (foreign key) or cascade, depending on how the FK is defined.

### Checking for NULL

```sql
SELECT customer_id, first_name, phone
FROM customers
WHERE phone IS NULL;
```

---

## 2. DQL: Data Query Language

### Selecting columns, aliases, DISTINCT

```sql
SELECT product_id, product_name AS "Product Name", stock_quantity
FROM products;

SELECT DISTINCT country FROM customers;
```

### Filtering with WHERE

```sql
-- Comparison
SELECT * FROM products WHERE price > 200;

-- AND: in stock and price > 150
SELECT * FROM products WHERE stock_quantity > 0 AND price > 150;

-- OR: out of stock or price < 150
SELECT * FROM products WHERE stock_quantity = 0 OR price < 150;

-- NOT: inactive products
SELECT * FROM products WHERE NOT is_active;
```

### BETWEEN (inclusive on both ends)

```sql
-- These two are equivalent
SELECT * FROM products WHERE price >= 120 AND price <= 200;
SELECT * FROM products WHERE price BETWEEN 120 AND 200;
```

### IN

```sql
-- Instead of chaining OR
SELECT * FROM products WHERE category_id IN (1, 3, 4, 5);
```

### LIKE (pattern matching)

| Pattern       | Meaning                |
| ------------- | ---------------------- |
| `'Wireless%'` | starts with "Wireless" |
| `'%less'`     | ends with "less"       |
| `'%less%'`    | contains "less"        |

```sql
SELECT * FROM products WHERE product_name LIKE 'Wireless%';
```

### NULL handling: IS NULL, IS NOT NULL, COALESCE

```sql
SELECT * FROM customers WHERE phone IS NULL;       -- no phone number
SELECT * FROM customers WHERE phone IS NOT NULL;   -- has phone number

-- COALESCE returns the first non-NULL value
SELECT
    first_name,
    last_name,
    COALESCE(phone, 'Phone number unavailable') AS phone_num,
    COALESCE(city, 'No city') AS city_name
FROM customers;
```

### ORDER BY

```sql
-- ASC = ascending (default), DESC = descending
SELECT * FROM products ORDER BY price ASC;    -- cheapest to most expensive
SELECT * FROM products ORDER BY price DESC;   -- most expensive first

-- Multiple sort keys
SELECT * FROM products ORDER BY category_id DESC, price ASC;
```

### LIMIT and OFFSET

```sql
-- Top 10 most expensive products
SELECT * FROM products ORDER BY price DESC LIMIT 10;
```

### Pagination

```sql
SELECT * FROM products
ORDER BY product_id
LIMIT 10
OFFSET 10;   -- page 2 with page size 10
```

Formula:

```
offset = (page - 1) * limit
```

| Page | Limit | Offset |
| ---- | ----- | ------ |
| 1    | 10    | 0      |
| 2    | 10    | 10     |
| 3    | 10    | 20     |

> Always pair `LIMIT`/`OFFSET` with `ORDER BY`, otherwise row order is not guaranteed.

---

## 3. Joins

| Join              | Returns                                                                  |
| ----------------- | ------------------------------------------------------------------------ |
| `INNER JOIN`      | only rows with a match in both tables                                    |
| `LEFT JOIN`       | all rows from the left table, plus matches from the right (NULL if none) |
| `RIGHT JOIN`      | all rows from the right table, plus matches from the left                |
| `FULL OUTER JOIN` | all rows from both tables, matched where possible                        |
| `CROSS JOIN`      | every combination of rows (Cartesian product)                            |

### INNER JOIN: only orders punched by employees

```sql
SELECT o.status, o.order_id, e.email, e.first_name
FROM orders AS o
INNER JOIN employees AS e ON o.employee_id = e.employee_id;
```

### LEFT JOIN: all orders, employee details empty if the order came directly from a customer

```sql
SELECT o.status, o.order_id, e.email, e.first_name
FROM orders AS o
LEFT JOIN employees AS e ON o.employee_id = e.employee_id;
```

### RIGHT and FULL OUTER JOIN

```sql
-- All employees, even those who processed no orders
SELECT o.status, o.order_id, e.email, e.first_name
FROM orders AS o
RIGHT JOIN employees AS e ON o.employee_id = e.employee_id;

-- Everything from both sides
SELECT o.status, o.order_id, e.email, e.first_name
FROM orders AS o
FULL OUTER JOIN employees AS e ON o.employee_id = e.employee_id;
```

### CROSS JOIN

```sql
SELECT * FROM departments CROSS JOIN categories;
```

### Anti-join pattern: find rows with NO match

```sql
-- Customers who never placed an order
SELECT c.first_name, c.email, o.status, o.order_id
FROM customers AS c
LEFT JOIN orders AS o ON o.customer_id = c.customer_id
WHERE o.status IS NULL;

-- Orders that have not been paid
SELECT *
FROM orders AS o
LEFT JOIN payments AS p ON p.order_id = o.order_id
WHERE p.payment_id IS NULL;
```

### Multi-table join

```sql
-- Full order info: items, product, category, customer
SELECT
    o.order_id, o.status,
    oi.product_id, oi.unit_price,
    p.product_name, p.price,
    c.category_name,
    cu.first_name, cu.last_name
FROM orders AS o
LEFT JOIN order_items AS oi ON oi.order_id = o.order_id
LEFT JOIN products    AS p  ON p.product_id = oi.product_id
LEFT JOIN categories  AS c  ON c.category_id = p.category_id
LEFT JOIN customers   AS cu ON cu.customer_id = o.customer_id;
```

### Self join: employees with their manager

```sql
SELECT
    emp.first_name AS "Employee First Name",
    emp.last_name  AS "Employee Last Name",
    manager_tbl.first_name AS "Manager First Name",
    manager_tbl.last_name  AS "Manager Last Name"
FROM employees AS emp
LEFT JOIN employees AS manager_tbl ON emp.manager_id = manager_tbl.employee_id;
```

---

## 4. Aggregation

Common aggregate functions: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

### COUNT

```sql
SELECT
    COUNT(*)     AS cust_count,              -- all rows
    COUNT(phone) AS cust_with_phone_number   -- non-NULL phones only
FROM customers;

SELECT COUNT(DISTINCT country) FROM customers;
```

### SUM: total revenue

```sql
SELECT SUM(quantity * unit_price) AS total_revenue FROM order_items;
```

### AVG, MIN, MAX

```sql
SELECT ROUND(AVG(price), 2) AS avg_price FROM products;

SELECT MIN(price) AS minimum_price, MAX(price) AS max_price FROM products;

SELECT
    COUNT(*)              AS total_product,
    ROUND(AVG(price), 2)  AS average_price,
    MIN(price)            AS min_price,
    MAX(price)            AS max_price,
    SUM(stock_quantity)   AS total_stock
FROM products;
```

### GROUP BY

```sql
-- Products per category
SELECT category_id, COUNT(*)
FROM products
GROUP BY category_id;

-- Active/inactive products per category
SELECT category_id, is_active, COUNT(*)
FROM products
GROUP BY category_id, is_active;

-- Include category name (every non-aggregated column must be in GROUP BY)
SELECT p.category_id, c.category_name, COUNT(*)
FROM products AS p
LEFT JOIN categories AS c ON c.category_id = p.category_id
GROUP BY p.category_id, c.category_name;
```

### HAVING: filter after grouping

```sql
-- Categories with more than 5 products
SELECT category_id, COUNT(*)
FROM products
GROUP BY category_id
HAVING COUNT(*) > 5;
```

```sql
-- Departments with more than one managed employee
-- AND combined salary above 200000
SELECT department_id, COUNT(*), SUM(salary) AS total_dep_salary
FROM employees
WHERE manager_id IS NOT NULL
GROUP BY department_id
HAVING COUNT(*) > 1 AND SUM(salary) > 200000;
```

> **`WHERE` vs `HAVING`:** `WHERE` filters rows _before_ grouping; `HAVING` filters groups _after_ aggregation.

---

## 5. Subqueries

### Subquery in WHERE with IN

```sql
-- Customers with at least one cancelled order
SELECT *
FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM orders WHERE status = 'cancelled'
);
```

### Scalar subquery

```sql
-- Products priced above the average price
SELECT *
FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

Example: prices 100, 200, 1000, 200, 900 give an average of 480, so only 1000 and 900 are returned.

### Derived table (subquery in FROM)

```sql
-- Categories with sales > 500, including category name
SELECT c.category_name, category_stats.category_sales
FROM (
    SELECT p.category_id AS stat_id,
           SUM(oi.quantity * oi.unit_price) AS category_sales
    FROM order_items AS oi
    LEFT JOIN products AS p ON p.product_id = oi.product_id
    GROUP BY p.category_id
) AS category_stats
LEFT JOIN categories AS c ON c.category_id = category_stats.stat_id
WHERE category_sales > 500;
```

---

## 6. CTEs: Common Table Expressions

A CTE is a named, temporary result set defined with `WITH`. It makes queries more readable than nested subqueries.

```sql
WITH var_name AS (
    -- SQL query
)
SELECT ... FROM var_name;
```

### Same category-sales query, rewritten as a CTE

```sql
WITH category_stats AS (
    SELECT p.category_id,
           SUM(oi.quantity * oi.unit_price) AS category_sales
    FROM order_items AS oi
    LEFT JOIN products AS p ON p.product_id = oi.product_id
    GROUP BY p.category_id
)
SELECT c.category_name, category_stats.category_sales
FROM category_stats
LEFT JOIN categories AS c ON c.category_id = category_stats.category_id;
```

### Multiple CTEs (chained)

```sql
-- Category with the highest sales
WITH category_stats AS (
    SELECT p.category_id,
           SUM(oi.quantity * oi.unit_price) AS category_sales
    FROM order_items AS oi
    LEFT JOIN products AS p ON p.product_id = oi.product_id
    GROUP BY p.category_id
),
hsc AS (
    SELECT * FROM category_stats
    ORDER BY category_sales DESC
    LIMIT 1
)
SELECT c.category_name, hsc.category_sales
FROM hsc
LEFT JOIN categories AS c ON c.category_id = hsc.category_id;
```

### Reusability: define once, use many times

Goal: each customer's total spend, the average spend across customers, and a flag for above-average spenders.

```sql
WITH customer_total AS (
    SELECT o.customer_id,
           SUM(oi.quantity * oi.unit_price) AS customer_spend
    FROM orders AS o
    LEFT JOIN order_items AS oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.customer_id
)
SELECT
    ct.customer_id,
    ct.customer_spend,
    ROUND((SELECT AVG(customer_spend) FROM customer_total), 2) AS avg_spend,
    ct.customer_spend > (SELECT AVG(customer_spend) FROM customer_total) AS is_above_avg
FROM customer_total AS ct;
```

The equivalent without a CTE has to repeat the whole inner query three times (for `ct`, for `avg_spend`, and for `is_above_avg`). That is the main advantage of CTEs: write the logic once and reference it by name.

---

## 7. Window Functions

Window functions compute values across a set of rows **without collapsing them** (unlike `GROUP BY`).

### GROUP BY vs window function

```sql
-- GROUP BY: one row per category
SELECT category_id, SUM(price) AS product_sum
FROM products
GROUP BY category_id;

-- Window: keeps every product row, adds the category total
SELECT *, SUM(price) OVER (PARTITION BY category_id) AS category_total_price
FROM products;
```

### OVER () with no partition: whole table

```sql
SELECT *, AVG(price) OVER () AS avg_price
FROM products;
```

### Ranking functions

```sql
SELECT
    product_name,
    price,
    ROW_NUMBER() OVER (ORDER BY price DESC) AS prod_row_num,
    RANK()       OVER (ORDER BY price DESC) AS prod_rank,
    DENSE_RANK() OVER (ORDER BY price DESC) AS prod_dense_rank
FROM products;
```

| Function       | Behavior on ties (e.g. prices 300, 200, 200, 100) |
| -------------- | ------------------------------------------------- |
| `ROW_NUMBER()` | always unique: 1, 2, 3, 4                         |
| `RANK()`       | ties share a rank, then skips: 1, 2, 2, 4         |
| `DENSE_RANK()` | ties share a rank, no skip: 1, 2, 2, 3            |

---

## 8. UNION and UNION ALL

Combine the results of multiple queries. Each query must return the same number of columns with compatible types.

```sql
-- List of all people in the community
SELECT first_name, last_name, 'employee' AS role FROM employees
UNION
SELECT first_name, last_name, 'customer' AS role FROM customers;
```

| Operator    | Duplicates                            |
| ----------- | ------------------------------------- |
| `UNION`     | removed (slower, needs a dedupe step) |
| `UNION ALL` | kept (faster)                         |

---

## 9. Indexes

```sql
CREATE INDEX idx_customer_email ON customers(email);
```

An index speeds up lookups and filters on the indexed column (here, searching customers by email), at the cost of extra storage and slightly slower writes.

---

## Quick Reference: Logical Order of Execution

```
FROM / JOIN  ->  WHERE  ->  GROUP BY  ->  HAVING  ->  SELECT  ->  ORDER BY  ->  LIMIT / OFFSET
```

This explains why you can't use a `SELECT` alias inside `WHERE`, and why `HAVING` (not `WHERE`) is used to filter on aggregates.

---

## Things to Double-Check in the Original Script

A few queries in the source file don't quite match their comments:

1. **"Products starting with Wireless"** uses `LIKE '%less%'`, which matches _anywhere_. Use `'Wireless%'`.
2. **"Customers who do not have phone numbers"** is followed by `IS NOT NULL`, which returns the opposite. Use `IS NULL`.
3. **"Cheapest to expensive"** uses `ORDER BY price DESC`, which gives expensive first. Use `ASC`.
4. **Departments with combined salary > 200000**: the `HAVING` clause only checks `COUNT(*) > 1`. Add `AND SUM(salary) > 200000`.
5. **Unscoped `UPDATE`s** (`SET is_active = FALSE`, `SET price = price + 100`) change every row. Fine for practice, risky in real data.
