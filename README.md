# 8-Week-SQL-Challenge
## Case Study #1 : Danny's Diner

<img src="https://8weeksqlchallenge.com/images/case-study-designs/1.png" alt="Image" width="300" height="320">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite.
Through these insights, Danny aims to forge deeper connects with his customers and deliver more personalised experience for his loyal customers.

***

## Entity Relationship Diagram

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

***

## Question and Solution

> 1. What is the total amount each customer spent at the restaurant?
<br>

````sql
SELECT 
    customer_id,
    SUM(price) total_amount_spent
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
USING(product_id)
GROUP BY 1
ORDER BY 2 DESC
````

#### Steps:
- Use **JOIN** to merge `dannys_diner.sales` and `dannys_diner.menu` tables as the required information `sales.customer_id` and `menu.price` are from both tables.
- Use **SUM** to compute `total amount spent`.
- Group the aggregated results by `sales.customer_id`. 

#### Answer:
| customer_id | total_sales |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

<br>

***

<br>

> 2. How many days has each customer visited the restaurant?
<br>

````sql
SELECT
    customer_id, 
    COUNT(DISTINCT TO_CHAR(order_date, 'dd-mm-yy')) order_date
FROM dannys_diner.sales
GROUP BY 1
ORDER BY 2 DESC
````

#### Steps:
- Use **TO_CHAR** to convert `order_date` from Timestamp to Date.
- Use **COUNT** with **DISTINCT** to compute unique number of day.
- Group the aggregated results by `sales.customer_id`. 

#### Answer:
| customer_id | visit_count |
| ----------- | ----------- |
| B           | 6          |
| A           | 4          |
| C           | 2          |

<br>
<br>

>[!NOTE]
>I've seen solutions that reached the same result without converting `order_date` to Date format. It would also work because all time values are set to 0 in the data, I chose to add the extra step to be more careful to avoid unexpectedly double same day visit more than once.

***
<br>

> 3. What was the first item from the menu purchased by each customer?
<br>

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

#### Steps:
- Use **JOIN** to merge `dannys_diner.sales` and `dannys_diner.menu` tables as the required information `sales.customer_id`, `sales.order_date` and `menu.price` are from both tables.
- Use **DENSE_RANK** to determine order of purchase :  **PARTITION BY customer_id** tells the code to re-starts ranking whenever `customer_id` changes, while **ORDER BY order_date** orders the rows within individual partition by `order_date`.
- Wrap the above query with brackets and make it a Sub-query, alias it **base**.
- In the outer query, **SELECT DISTINCT** `customer_id` and `product_name` to avoid getting duplicated result if customers ordered 2 of the same item on their first visit.
- Use **WHERE** to filter **drk=1** to obtain only the 1st in rank for each customer.

#### Answer:
| customer_id | product_name | 
| ----------- | ----------- |
| A           | curry        | 
| A           | sushi        | 
| B           | curry        | 
| C           | ramen        |

<br>
<br>

>[!Tip]
>**DENSE_RANK** is used instead of **RANK**, so that if customers purchased different items on their first visit, we would get the all that was purchased on that day.

***
<br>

> 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
<br>

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

#### Steps:
- Use **COUNT** to compute `num_of_time_purchased` and group the aggregated results by `sales.product_id`. 
- Use **DENSE_RANK** to determine item most purchased.
- Wrap the above query with brackets and make it a Sub-query, alias it **a**.
- In the outer query, **SELECT** `product_name` and `num_of_time_purchased`.
- Use **WHERE** to filter **drk=1** to obtain only the most purchased product(s).

#### Answer:
|  product_name | num_of_time_purchased | 
| ----------- | ----------- |
| ramen       | 8 |

***

> 5. Which item was the most popular for each customer?
<br>

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

#### Steps:
- Create a **CTE** (Common Table Expression) name `popular_item`.
- Within the CTE, use **Count** to compute `num_of_times_ordered` and group the aggregated results by `sales.customer_id` and `sales.product_id`. 
- Use **DENSE_RANK** to determine items most purchased.
- In the outer query, use **JOIN** to merge `popular_item` and `dannys_diner.menu` to get `product_name`.
- Use **WHERE** to filter **drk=1** to obtain only the most purchased product(s) for each customer.

#### Answer:
| customer_id | product_name | order_count |
| ----------- | ---------- |------------  |
| A           | ramen        |  3   |
| B           | sushi        |  2   |
| B           | curry        |  2   |
| B           | ramen        |  2   |
| C           | ramen        |  3   |

***
<br>

> 6. Which item was purchased first by the customer after they became a member?

<br>

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

#### Steps:
- Create a **CTE** name `item_first_order`.
- Within the CTE, use **JOIN** to merge `dannys_diner.members`, `dannys_diner.sales` and `dannys_diner.menu` to get all the required fields.
- While merging `dannys_diner.sales`, use **AND mem.join_date<s.order_date** to merge only order after `join_date`.
- Use **DENSE_RANK** to sequence of purchase.
- In the outer query, filter **drk=1** to obtain only the item(s) first purchased by the customer after they became a member.

#### Answer:
| customer_id | product_name |
| ----------- | ---------- |
| A           | ramen        |
| B           | sushi        |

***
<br>

> 7. Which item was purchased just before the customer became a member?

<br>

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

#### Steps:
- Create a **CTE** name `item_ordered_before_membership`.
- Within the CTE, use **JOIN** to merge `dannys_diner.members`, `dannys_diner.sales` and `dannys_diner.menu` to get all the required fields.
- While merging `dannys_diner.sales`, use **AND mem.join_date>s.order_date** to merge only order before `join_date`.
- Use **DENSE_RANK** to sequence of purchase, use **ORDER BY s.order_dat** order the ranking starting with latest  `order_date`.
- In the outer query, filter **drk=1** to obtain only the item(s) purchased by the customer just before became a member.

#### Answer:
| customer_id | order_date               | product_name |
| ----------- | ------------------------ | ------------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        |
| A           | 2021-01-01T00:00:00.000Z | curry        |
| B           | 2021-01-04T00:00:00.000Z | sushi        |

***
<br>

> 8. What is the total items and amount spent for each member before they became a member?

<br>

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

#### Steps:
- Use **JOIN** to merge `dannys_diner.members`, `dannys_diner.sales` and `dannys_diner.menu` to get all the required fields.
- Use **COUNT** to compute `total_items_ordered` and group the aggregated results by `sales.customer_id`. 
- Use **WHERE** to filter **s.order_date<mem.join_date** to get only the item(s) purchased by the customer just before became a member.


#### Answer:
| customer_id | total_items_ordered | total_amount_spent |
| ----------- | ------------------- | ------------------ |
| A           | 2                   | 25                 |
| B           | 3                   | 40                 |

***
<br>


> 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
<br>

````sql
SELECT
    s.customer_id,
    SUM(CASE WHEN m.product_name='sushi' THEN m.price*20 ELSE m.price*10 END) as points
FROM dannys_diner.sales s
JOIN dannys_diner.menu m USING(product_id)
GROUP BY 1
ORDER BY 1

````

#### Steps:
- Use **JOIN** to merge `dannys_diner.sales` and `dannys_diner.menu` to get all the required fields.
- Use conditional **CASE** statement to calculate points given if customers purchased sushi or any other items.
- Use **SUM** to compute `points` and group the aggregated results by `sales.customer_id`. 

#### Answer:
| customer_id | points |
| ----------- | ------ |
| A           | 860    |
| B           | 940    |
| C           | 360    |

***
<br>


> 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
<br>

````sql
SELECT 
    s.customer_id,
    SUM(CASE 
        WHEN s.order_date>=mem.join_date AND s.order_date<=mem.join_date+6 THEN m.price*20
    	WHEN m.product_name='sushi' THEN m.price*20
        WHEN m.product_name!='sushi' THEN m.price*10
    END) points_earned_by_end_jan       
FROM dannys_diner.sales s
LEFT JOIN dannys_diner.menu m USING(product_id)
LEFT JOIN dannys_diner.members mem USING(customer_id)
WHERE s.order_date<'2021-02-01'
	AND customer_id in ('A','B')
GROUP BY 1
ORDER BY 1
````
#### Steps:
- Use **LEFT JOIN** to merge `dannys_diner.sales`, `dannys_diner.menu` and `dannys_diner.members` to get all the required fields.
- Use conditional **CASE** statement to calculate points given depending when order was made and the item purchased.
- Use **SUM** to compute `points` and group the aggregated results by `sales.customer_id`.
- Use **WHERE** and **AND** to filter **s.order_date<'2021-02-01'** to only include order made up till 2021-01-31 and filter only Customer A and Customer B.

#### Answer:
| customer_id | points_earned_by_end_jan |
| ----------- | ------------------------ |
| A           | 1370                     |
| B           | 820                      |

***
