# 8_weeks_sql_challenge

## Week 1's Challenge - [Danny's Diner SQL](https://8weeksqlchallenge.com/case-study-1/)
<details>
  <summary> SQL Schema </summary>

```
CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');

```
  
</details>

## Metadata
### Database Name
`danny_diner`

### Table Name
1. `sales`
2. `menu`
3. `members`

### 1. What is the total amount each customer spent at the restaurant?
<details>
  <summary> SQL Query </summary>

```
SELECT
  	s.customer_id,
    SUM(m.price)
FROM dannys_diner.menu AS m
JOIN dannys_diner.sales AS s
ON m.product_id = s.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id ASC;
```
  
</details>

### 2. How many days has each customer visited the restaurant?
<details>
  <summary> SQL Query </summary>

```
SELECT
	s.customer_id AS "Customer ID",
    COUNT(DISTINCT(s.order_date)) AS "No. of Visits"
FROM dannys_diner.sales AS s
GROUP BY s.customer_id
ORDER BY s.customer_id ASC;
```
  
</details>

### 3. What was the first item from the menu purchased by each customer?
<details>
  <summary> SQL Query </summary>

```
Select * 
FROM(
SELECT
	ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY s.order_date, s.product_id ASC) AS "ranks",
    s.customer_id,
    m.product_name
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id) as t
WHERE t.ranks = 1
```
Notes: This query is not optimised and may encounter latency issues with large databases. Ideally, we will want to filter to the first item **before** joining to the menu table
  
</details>

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
<details>
  <summary> SQL Query </summary>

```
SELECT
    m.product_name,
    COUNT(m.product_name) AS "Most Popular Food"
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY COUNT(m.product_name) DESC
LIMIT 1;
```
  
</details>

### 5. Which item was the most popular for each customer?
<details>
  <summary> SQL Query </summary>

```
WITH ranked_tbl AS (
SELECT
	s.customer_id,
    m.product_name,
    COUNT(m.product_name) AS "Most Popular Food",
    dense_rank() OVER (PARTITION BY s.customer_id ORDER BY s.customer_id ASC, COUNT(m.product_name) DESC) AS "ranks"
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
GROUP BY s.customer_id, m.product_name
ORDER BY s.customer_id ASC, COUNT(m.product_name) DESC)

SELECT 
	r.customer_id,
    r.product_name AS "Food",
    r.ranks AS "Most Popular Food(s)"
FROM ranked_tbl AS r
WHERE r.ranks = 1
```
  
</details>

Notes: Customer B has tied for all three foods.

### 6. Which item was purchased first by the customer after they became a member?
<details>
  <summary> SQL Query </summary>

```
WITH member_tbl AS (
SELECT 
	s.customer_id, 
    s.order_date,
    m.join_date,
    s.product_id,
  	row_number() OVER (PARTITION BY s.customer_id ORDER BY s.customer_id ASC, s.order_date ASC) AS "row_num"
FROM dannys_diner.sales AS s
INNER JOIN dannys_diner.members AS m
ON s.customer_id = m.customer_id
WHERE s.order_date >= m.join_date
ORDER BY s.customer_id ASC, s.order_date ASC)

SELECT 
	mt.customer_id,
    mt.order_date,
    m.product_name
FROM member_tbl AS mt
JOIN dannys_diner.menu AS m
ON mt.product_id = m.product_id
WHERE mt.row_num = 1
ORDER BY mt.customer_id ASC
```
  
</details>

Notes: 
* Assumption #01: Non-members (e.g. Customer C) are excluded from the query
* Assumption #02: Membership status is activated on the day it is applied on.

### 7. Which item was purchased just before the customer became a member?
<details>
  <summary> SQL Query </summary>

```
WITH member_tbl AS (
SELECT 
	s.customer_id, 
    s.order_date,
    m.join_date,
    s.product_id,
  	row_number() OVER (PARTITION BY s.customer_id ORDER BY s.customer_id ASC, s.order_date DESC) AS "row_num"
FROM dannys_diner.sales AS s
LEFT JOIN dannys_diner.members AS m
ON s.customer_id = m.customer_id
WHERE s.order_date < m.join_date
ORDER BY s.customer_id ASC, s.order_date ASC)

SELECT 
	mt.customer_id,
    mt.order_date,
    mt.join_date,
    mt.product_id,
    mt.row_num,
    m.product_name
FROM member_tbl AS mt
JOIN dannys_diner.menu AS m
ON mt.product_id = m.product_id
WHERE mt.row_num = 1
ORDER BY mt.customer_id ASC, row_num ASC
```
</details>

Notes:
* Assumption #01: we're only interested in customers who are eventually converted into members (e.g. Customer A & Customer B, excl. Customer C)
* Assumption #02: when items are tied (in terms of ranking), we will take the first item based Product ID
* Comment #01: If we want to display items that are tied for the first place, use `dense_rank()` instead of `row_number()` 

### 8. What is the total items and amount spent for each member before they became a member?
<details>
  <summary> SQL Query </summary>

```
WITH member_tbl AS (
SELECT 
	s.customer_id, 
    s.order_date,
    m.join_date,
    s.product_id
FROM dannys_diner.sales AS s
LEFT JOIN dannys_diner.members AS m
ON s.customer_id = m.customer_id
WHERE s.order_date < m.join_date
ORDER BY s.customer_id ASC, s.order_date ASC)

SELECT 
	mt.customer_id,
    COUNT(mt.product_id) AS "No. of Items",
    SUM(m.price) AS "Total Price"
FROM member_tbl AS mt
JOIN dannys_diner.menu AS m
ON mt.product_id = m.product_id
GROUP BY mt.customer_id
ORDER BY mt.customer_id ASC
```
  
</details>

### 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
<details>
  <summary> SQL Query </summary>

```
WITH total_tbl AS (
SELECT 
	s.customer_id,
    m.product_id,
    m.product_name,
    COUNT(m.product_name) AS "Items",
    SUM(m.price) AS "total",
    CASE
    	WHEN m.product_id = 1 THEN 20
    ELSE
    	10
    END AS "amp"  	
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
GROUP BY s.customer_id, m.product_id, m.product_name
ORDER BY s.customer_id ASC
)

SELECT 
	tt.customer_id AS "Customer ID",
	SUM(tt.total * tt.amp) AS "Total Points"
FROM total_tbl AS tt
GROUP BY tt.customer_id
```
</details>

Notes:
* e.g. Customer C --> $36 of ramen  = 360 points

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
