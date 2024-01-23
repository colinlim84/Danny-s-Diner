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
-- 5. Which item was the most popular for each customer?
-- 6. Which item was purchased first by the customer after they became a member?
-- 7. Which item was purchased just before the customer became a member?
-- 8. What is the total items and amount spent for each member before they became a member?
-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
