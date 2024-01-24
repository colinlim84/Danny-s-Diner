# Danny-s-Diner


Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite.


-- 1. What is the total amount each customer spent at the restaurant?

````sql
SELECT 
    customer_id,
    SUM(price) total_price
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
USING(product_id)
GROUP BY 1
ORDER BY 2 DESC
````


-- 2. How many days has each customer visited the restaurant?

````sql
SELECT
    customer_id, 
    COUNT(DISTINCT TO_CHAR(order_date, 'dd-mm-yy')) order_date
FROM dannys_diner.sales
GROUP BY 1
ORDER BY 2 DESC
````



-- 3. What was the first item from the menu purchased by each customer?

````sql
SELECT DISTINCT
    customer_id,
    product_name
FROM
  (SELECT 
      s.customer_id,
      s.order_date,
      m.product_name,
      DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) drk
  FROM dannys_diner.sales s
  JOIN dannys_diner.menu m
  USING(product_id)) base
WHERE drk=1
````



-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
````sql
SELECT 
      product_name,
      num_of_time_purchased
FROM
  (SELECT 
      product_id,
      COUNT(*) num_of_time_purchased,
      DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) drk
  FROM dannys_diner.sales
  GROUP BY 1) a
JOIN dannys_diner.menu
USING(product_id)
WHERE drk=1
````


-- 5. Which item was the most popular for each customer?
````sql
WITH popular_item AS (
SELECT
      customer_id,
      product_id,
      COUNT(*) num_of_times_ordered,
      DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(*) DESC) drk
FROM dannys_diner.sales
GROUP BY 1,2)


SELECT 
    p.customer_id, 
    m.product_name,
    p.num_of_times_ordered
FROM popular_item p
JOIN dannys_diner.menu m
USING(product_id)
WHERE drk=1
ORDER BY 1,2
````




-- 6. Which item was purchased first by the customer after they became a member?

````sql
WITH item_first_order AS (
SELECT 
    mem.*,
    s.order_date,
    s.product_id,
    m.product_name,
    DENSE_RANK() OVER(PARTITION BY mem.customer_id ORDER BY s.order_date) drk
FROM dannys_diner.members mem
JOIN dannys_diner.sales s ON mem.customer_id=s.customer_id AND mem.join_date<s.order_date
JOIN dannys_diner.menu m ON s.product_id=m.product_id)

SELECT 
	customer_id,
    product_name
FROM item_first_order
WHERE drk=1
````




-- 7. Which item was purchased just before the customer became a member?

````sql
WITH item_ordered_before_membership AS (
SELECT 
	mem.*,
    s.order_date,
    s.product_id,
    m.product_name,
    DENSE_RANK() OVER(PARTITION BY mem.customer_id ORDER BY s.order_date DESC) drk
FROM dannys_diner.members mem
JOIN dannys_diner.sales s ON mem.customer_id=s.customer_id AND mem.join_date>s.order_date
JOIN dannys_diner.menu m ON s.product_id=m.product_id)

SELECT 
	customer_id,
    order_date,
    product_name
FROM item_ordered_before_membership
WHERE drk=1
````

-- 8. What is the total items and amount spent for each member before they became a member?

````sql
SELECT 
    s.customer_id,
    COUNT(*) total_items_ordered,
    SUM(m.price) total_amount_spent
FROM dannys_diner.sales s
JOIN dannys_diner.members mem USING(customer_id)
JOIN dannys_diner.menu m USING(product_id)
WHERE s.order_date<mem.join_date
GROUP BY 1
ORDER BY 1
````


-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

````sql
SELECT
    s.customer_id,
    SUM(CASE WHEN m.product_name='sushi' THEN 20 ELSE 10 END) as points
FROM dannys_diner.sales s
JOIN dannys_diner.members mem USING(customer_id)
JOIN dannys_diner.menu m USING(product_id)
WHERE s.order_date>=mem.join_date
GROUP BY 1

````

-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
