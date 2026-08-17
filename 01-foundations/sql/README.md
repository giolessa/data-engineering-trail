# SQL

## What is a query, really?

A SQL query is a question you ask a database, written in a specific format so the database engine can understand it. When a query runs, the database doesn't process it top to bottom in the order it's written — it follows a fixed internal order:

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

This explains some behavior that looks confusing at first — for example, why a column alias defined in `SELECT` can't be used inside `WHERE`. At the point `WHERE` runs, `SELECT` hasn't executed yet, so that alias doesn't exist yet.

Every example below uses one imaginary table, `orders`, so the examples build on each other:

| id | customer  | product   | quantity | price | order_date |
|----|-----------|-----------|----------|-------|------------|
| 1  | Ana       | Notebook  | 1        | 3500  | 2026-01-05 |
| 2  | Bruno     | Mouse     | 2        | 80    | 2026-01-06 |
| 3  | Ana       | Mouse     | 1        | 80    | 2026-01-10 |
| 4  | Carla     | Monitor   | 1        | 900   | 2026-02-01 |
| 5  | Bruno     | Notebook  | 1        | 3200  | 2026-02-03 |

And a second table, `customers`, used later for joins:

| id | name  | city           |
|----|-------|----------------|
| 1  | Ana   | Rio de Janeiro |
| 2  | Bruno | São Paulo      |
| 3  | Carla | Belo Horizonte |

## SELECT and FROM

`SELECT` picks which columns to return. `FROM` says which table they come from. Even though `SELECT` is written first, `FROM` runs first — the database needs to know which table it's working with before it can pull any columns.

```sql
SELECT customer, product
FROM orders;
```

`SELECT *` returns every column — fine for quick exploration, but in real pipelines it's better to name only the columns needed. It's clearer to read and won't silently change behavior if someone adds a column to the table later.

## WHERE

`WHERE` filters rows before anything else happens to them — before grouping, before ordering. Only rows where the condition evaluates to true make it through.

```sql
SELECT customer, product
FROM orders
WHERE customer = 'Ana';
```

Common comparisons: `=`, `!=`, `>`, `<`, `>=`, `<=`. Conditions combine with `AND` / `OR`:

```sql
SELECT *
FROM orders
WHERE customer = 'Ana' AND price > 100;
```

## ORDER BY and LIMIT

`ORDER BY` sorts the result set. It runs near the end, after filtering and grouping — it can only sort what's already been selected.

```sql
SELECT customer, price
FROM orders
ORDER BY price DESC;
```

`ASC` (default) or `DESC`. Sorting by more than one column breaks ties using the next column in the list:

```sql
SELECT customer, product, price
FROM orders
ORDER BY customer ASC, price DESC;
```

`LIMIT` restricts how many rows come back, and runs last — useful for "top N" questions:

```sql
SELECT *
FROM orders
ORDER BY price DESC
LIMIT 2;
```

## Aggregations: GROUP BY, HAVING, and aggregate functions

Aggregate functions collapse many rows into a single value: `COUNT` (how many), `SUM` (total), `AVG` (average), `MIN`/`MAX`.

```sql
SELECT SUM(price) AS total_revenue
FROM orders;
```

`GROUP BY` splits the table into groups first, then applies the aggregate function to each group separately, instead of the whole table at once. This answers questions like "how much did each customer spend?":

```sql
SELECT customer, SUM(price) AS total_spent
FROM orders
GROUP BY customer;
```

Every column in `SELECT` that isn't wrapped in an aggregate function must appear in `GROUP BY` — the database needs to know how to group rows that share that value.

`HAVING` filters *after* grouping, based on the aggregated result. This is why it's a separate clause from `WHERE`: `WHERE` filters raw rows before grouping happens, `HAVING` filters groups after they've been formed.

```sql
SELECT customer, SUM(price) AS total_spent
FROM orders
GROUP BY customer
HAVING SUM(price) > 1000;
```

Trying to write `WHERE SUM(price) > 1000` would fail — at the point `WHERE` runs, no aggregation has happened yet, so there's no sum to compare against.

## JOINs

Real data is split across tables to avoid repeating information (a customer's city shouldn't be copied into every single order row). Joins bring related tables back together for a query.

**INNER JOIN** returns only rows that have a match in both tables:

```sql
SELECT orders.customer, orders.product, customers.city
FROM orders
INNER JOIN customers ON orders.customer = customers.name;
```

This reads as: for each row in `orders`, find the row in `customers` where the names match, and combine them into one row. If a customer in `orders` has no matching row in `customers`, that order is left out entirely.

**LEFT JOIN** keeps every row from the left (first) table, even if there's no match on the right — unmatched columns come back as `NULL`:

```sql
SELECT customers.name, orders.product
FROM customers
LEFT JOIN orders ON customers.name = orders.customer;
```

This is the difference that matters most in practice: use `INNER JOIN` when you only want rows that exist in both tables, use `LEFT JOIN` when you want to keep everything from the main table regardless of whether it has a match (for example, listing every customer, including ones who've never ordered anything).

`RIGHT JOIN` is the mirror of `LEFT JOIN` (keeps everything from the right table), and `FULL JOIN` keeps everything from both sides, matched where possible.

## Subqueries

A subquery is a query nested inside another one, used when a question needs to happen in two steps. For example: "which customers spent more than the average order value?"

```sql
SELECT customer, price
FROM orders
WHERE price > (SELECT AVG(price) FROM orders);
```

The inner query `(SELECT AVG(price) FROM orders)` runs first and produces a single value — the average — which the outer query then compares each row against. Subqueries can also appear in `FROM` (treating a query's result as if it were a table) or inside `SELECT`.

## CTEs (Common Table Expressions)

A CTE, written with `WITH`, names a subquery so it can be referenced like a temporary table — mainly for readability when a query has multiple steps.

```sql
WITH customer_totals AS (
  SELECT customer, SUM(price) AS total_spent
  FROM orders
  GROUP BY customer
)
SELECT customer, total_spent
FROM customer_totals
WHERE total_spent > 1000;
```

This does the same thing a subquery could do, but reads top to bottom like a sequence of steps instead of nesting queries inside each other — which matters a lot once queries get more complex than this example.

## Next

Window functions — running totals, rankings, and calculations across rows without collapsing them into groups (the difference from `GROUP BY`).

## Resources

*(add whatever you use to study this — course, doc, video)*
