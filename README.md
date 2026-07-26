# Restaurant Orders & Delivery Analysis using PostgreSQL

<p align="center">
  <img src="food delivery image.png" alt="Restaurant SQL Project Banner" width="900">
</p>

## Overview

This project demonstrates advanced SQL skills by analyzing a Food Delivery database containing information about customers, restaurants, orders, deliveries, and riders. It solves 20 real-world business problems using PostgreSQL to uncover insights into customer behavior, restaurant performance, rider efficiency, sales trends, and business growth. The project showcases the practical use of SQL concepts such as JOINs, CTEs, Window Functions, Aggregate Functions, Date & Time Functions, CASE statements, and Ranking Functions to support data-driven decision-making.

---

## Objectives

Analyze customer ordering behavior and spending patterns.
Measure restaurant performance through revenue and order analysis.
Evaluate rider efficiency using delivery time and earnings.
Identify popular dishes across different cities and seasons.
Track monthly and yearly sales trends.
Calculate business KPIs such as Average Order Value (AOV), Customer Lifetime Value (CLV), cancellation rates, and restaurant growth.
Perform customer segmentation based on spending behavior.
Generate actionable business insights using SQL.
Demonstrate proficiency in advanced SQL concepts through real-world business scenarios.

---

## Dataset Description

### The project uses a relational Food Delivery dataset consisting of five interconnected tables:

### Customers:
Stores customer information for analyzing ordering behavior, spending patterns, and customer lifetime value.
### Restaurants:
Contains restaurant details used to evaluate revenue, growth, popularity, and city-wise performance.
### Orders:
Records all customer orders, including order details, amounts, dates, and menu items, serving as the primary table for sales and revenue analysis.
### Deliveries:
Includes delivery information such as rider assignments, delivery times, and order status, enabling delivery performance and cancellation analysis.
### Riders:
Stores delivery partner information used to evaluate rider efficiency, earnings, and performance.

This dataset enables comprehensive analysis of customer activity, restaurant operations, delivery performance, and overall business trends.

---

## SQL Concepts Used

SELECT Statements\
Filtering (WHERE)\
Sorting (ORDER BY)\
Aggregate Functions\
GROUP BY & HAVING\
INNER JOINs\
Common Table Expressions (CTEs)\
Window Functions\
Ranking Functions (RANK, DENSE_RANK)\
CASE Expressions\
Date & Time Functions\
Conditional Aggregation\
Subqueries\
Revenue and Time-Series Analysis

---

## Schema

```sql
CREATE TABLE customers
(
	customer_id INT PRIMARY KEY,
	customer_name VARCHAR(30) NOT NULL,
	registration_date DATE NOT NULL
);

CREATE TABLE restaurants
(
	restaurant_id INT PRIMARY KEY,
	restaurant_name VARCHAR(40) NOT NULL,
	city VARCHAR(20),
	opening_hours VARCHAR(10)
);

CREATE TABLE riders
(
	rider_id INT PRIMARY KEY,
	rider_name VARCHAR(25) NOT NULL,
	signup_date DATE
);

CREATE TABLE orders
(
	order_id INT PRIMARY KEY,
	customer_id INT,
	restaurant_id INT,
	order_item VARCHAR(20) NOT NULL,
	order_date DATE,
	order_time TIME,
	order_status VARCHAR(20) CHECK(order_status IN ('Pending', 'Preparing', 'Completed', 'Cancelled')),
	total_amount NUMERIC(10, 2) CHECK(total_amount > 0),
	CONSTRAINT fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
	CONSTRAINT fk_restaurants FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id)
);

CREATE TABLE deliveries
(
	delivery_id INT PRIMARY KEY,
	order_id INT,
	delivery_status VARCHAR(20) CHECK(delivery_status IN ('Not Assigned', 'Delayed', 'Delivered')),
	delivery_time TIME,
	rider_id INT,
	CONSTRAINT fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id),
	CONSTRAINT fk_riders FOREIGN KEY (rider_id) REFERENCES riders(rider_id)
);
```

<p align="center">
  <img src="Food Delivery GitHub project Schema.png" alt="Schema image" width="900">
</p>

---

### 1. Find the Top-5 most frequently ordered dishes by the customer 'Iqra Sheikh' in last 12 Months:

**Objective**

Identify each customer's favorite dishes to create personalized recommendations, targeted promotions, and improve customer retention.

```sql
SELECT
	o.order_item, COUNT(o.order_id) AS no_of_times_ordered
FROM orders o
	JOIN customers c
		ON o.customer_id = c.customer_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 YEAR'
	AND c.customer_name = 'Iqra Sheikh'
	AND o.order_status = 'Completed'
GROUP BY o.order_item
ORDER BY no_of_times_ordered DESC
LIMIT 5;
```

### 2. Identify the time-slots during which the most orders are placed, based on 6-hour interval:

**Objective**

Determine peak ordering hours to optimize staffing, kitchen capacity, rider availability, and reduce delivery delays.

```sql
SELECT
	CASE
		WHEN EXTRACT(HOUR FROM (order_time)) BETWEEN 0 AND 5 THEN '00:00 - 05:59' 
		WHEN EXTRACT(HOUR FROM (order_time)) BETWEEN 6 AND 11 THEN '06:00 - 11:59' 
		WHEN EXTRACT(HOUR FROM (order_time)) BETWEEN 12 AND 17 THEN '12:00 - 17:59'
		ELSE '18:00 - 23:59'
	END AS time_slots,
	COUNT(order_id) AS no_of_orders_placed
FROM orders
GROUP BY time_slots
ORDER BY no_of_orders_placed DESC;
```

### 3. Find the average order value per customer who has placed more than 50 ORDERS. Return customer_name, aov(average order value)

**Objective**

Identify high-value customers who consistently place large orders for premium marketing campaigns and loyalty rewards.

```sql
SELECT
	c.customer_name,
	ROUND(AVG(o.total_amount), 2) AS avg_ord_value
FROM orders o
	JOIN customers c
		ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING COUNT(o.order_id) > 50
ORDER BY avg_ord_value DESC;
```

### 4. List the customers who have spent more than 20k on orders, Return cust_id, cust_name:

**Objective**

Recognize the company's most valuable customers for VIP programs, exclusive offers, and customer retention strategies.

```sql
SELECT c.customer_id, c.customer_name, SUM(o.total_amount) AS total_spent
FROM orders o
	JOIN customers c
		ON o.customer_id = c.customer_id
WHERE o.order_status = 'Completed'
GROUP BY c.customer_id, c.customer_name
HAVING SUM(o.total_amount) > 20000
ORDER BY total_spent DESC;
```

### 5. Find orders that were place but NOT delivered, Find each restaurant name, city and no. of NOT-delivered orders.

**Objective**

Detect restaurants with frequent delivery failures to improve operational efficiency and customer satisfaction.

```sql
SELECT
	r.restaurant_name, r.city, COUNT(o.order_id) AS no_of_not_delivered_orders
FROM orders o
	JOIN deliveries d
		ON o.order_id = d.order_id
	JOIN restaurants r
		ON o.restaurant_id = r.restaurant_id
WHERE o.order_status <> 'Cancelled'		-- If an order is cancelled by customer, it is NOT a delivery failure.
	AND d.delivery_status <> 'Delivered'
GROUP BY r.restaurant_id, r.restaurant_name, r.city
ORDER BY no_of_NOT_delivered_orders DESC;
```

### 6. Rank restaurants by their total revenue from the last year, including their name, total revenue and their rank within their city.

**Objective**

Compare restaurant performance within each city to identify top-performing partners and support business growth decisions.

```sql
SELECT
	r.city, r.restaurant_id, r.restaurant_name, SUM(o.total_amount) AS total_revenue,
	DENSE_RANK() OVER(PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS rank_in_city
FROM orders o
	JOIN restaurants r
		ON o.restaurant_id = r.restaurant_id
WHERE o.order_status = 'Completed' AND
	(o.order_date >= DATE_TRUNC('YEAR', CURRENT_DATE) - INTERVAL '1 YEAR'
		AND o.order_date < DATE_TRUNC('YEAR', CURRENT_DATE))
GROUP BY r.city, r.restaurant_id, r.restaurant_name
ORDER BY r.city, rank_in_city;
```

### 7. Identify most popular dish in each city based on their orders:

**Objective**

Understand regional food preferences to optimize menus, inventory planning, and location-specific marketing campaigns.

```sql
WITH dishes_rank
AS(
	SELECT
		r.city, o.order_item AS dish_name, COUNT(o.order_id) AS no_of_orders,
		RANK() OVER(PARTITION BY r.city ORDER BY COUNT(o.order_id) DESC) AS rank_in_city
	FROM orders o
		JOIN restaurants r
			ON o.restaurant_id = r.restaurant_id
	WHERE o.order_status = 'Completed'
	GROUP BY r.city, o.order_item
)
SELECT
	city, dish_name, no_of_orders, rank_in_city
FROM dishes_rank
WHERE rank_in_city = 1;
```

### 8. Find customers who have NOT placed orders in 2026, but placed orders in 2025:

**Objective**

Identify churned customers who have stopped ordering so targeted re-engagement campaigns can be launched.

```sql
SELECT
	c.customer_id,
	c.customer_name
FROM customers c
WHERE EXISTS(
		SELECT 1
		FROM orders o
		WHERE o.customer_id = c.customer_id
		AND (order_date >= DATE '2025-01-01' AND o.order_date < DATE '2026-01-01')
	)
AND NOT EXISTS(
		SELECT 1
		FROM orders o
		WHERE o.customer_id = c.customer_id
		AND (o.order_date >= DATE '2026-01-01' AND o.order_date < DATE '2027-01-01')
	);
```

### 9. Calculate and Compare the order cancellation rate for each restaurant between the current year and the previous year:

**Objective**

Measure whether restaurant cancellation performance is improving or declining over time and identify restaurants requiring operational improvements.


```sql
WITH cancelled_orders
AS(
	SELECT
		r.restaurant_id,
		r.restaurant_name,
		COUNT(*) FILTER (WHERE o.order_date >= DATE_TRUNC('YEAR', CURRENT_DATE) - INTERVAL '1 YEAR'
							AND o.order_date < DATE_TRUNC('YEAR', CURRENT_DATE)
		) AS prev_year_total_orders,

		COUNT(*) FILTER(WHERE o.order_date >= DATE_TRUNC('YEAR', CURRENT_DATE) - INTERVAL '1 YEAR'
						AND o.order_date < DATE_TRUNC('YEAR', CURRENT_DATE)
						AND o.order_status = 'Cancelled'
		) AS prev_year_cancelled_orders,
		
		COUNT(
			CASE WHEN order_date >= DATE_TRUNC('YEAR', CURRENT_DATE) THEN o.order_id END
		) AS curr_year_total_orders,
		
		COUNT(
			CASE WHEN order_date >= DATE_TRUNC('YEAR', CURRENT_DATE)
				AND o.order_status = 'Cancelled' THEN o.order_id
			END
		) AS curr_year_cancelled_orders
	FROM orders o
		JOIN restaurants r
			ON o.restaurant_id = r.restaurant_id
	GROUP BY r.restaurant_id, r.restaurant_name
),
cancellation_rate
AS(
	SELECT
		restaurant_id, restaurant_name,
		prev_year_total_orders, prev_year_cancelled_orders,
		ROUND((prev_year_cancelled_orders * 100.0 / NULLIF(prev_year_total_orders, 0)), 2) AS prev_year_cancellation_rate,
		curr_year_total_orders, curr_year_cancelled_orders,
		ROUND((curr_year_cancelled_orders * 100.0 / NULLIF(curr_year_total_orders, 0)), 2) AS curr_year_cancellation_rate
	FROM cancelled_orders
)
SELECT
	restaurant_id, restaurant_name,
	prev_year_total_orders, prev_year_cancelled_orders, prev_year_cancellation_rate,
	curr_year_total_orders, curr_year_cancelled_orders, curr_year_cancellation_rate,
	curr_year_cancellation_rate - prev_year_cancellation_rate AS cancellation_rate_change
FROM cancellation_rate
ORDER BY cancellation_rate_change DESC;
```

### 10. Determine each rider average delivery time:

**Objective**

Evaluate rider performance by measuring average delivery times to improve service quality and delivery efficiency.

```sql
SELECT
    r.rider_id,
    r.rider_name,
    AVG(d.delivery_time - o.order_time) AS average_delivery_time
FROM orders o
JOIN deliveries d
    ON o.order_id = d.order_id
JOIN riders r
    ON d.rider_id = r.rider_id
WHERE d.delivery_status = 'Delivered'
GROUP BY r.rider_id, r.rider_name
ORDER BY average_delivery_time;
```

### 11. Monthly Restaurant's Growth Ratio: Calculate each restaurant's growth ratio based on the total number of delivered orders since its joining:

**Objective**

Monitor restaurant growth trends over time to identify rapidly growing restaurants and those requiring business support.


```sql
WITH delivered_orders
AS(
	SELECT
		r.restaurant_id, r.restaurant_name,
		DATE_TRUNC('MONTH', o.order_date) AS order_year_month,
		COUNT(*) FILTER(WHERE o.order_status = 'Completed') AS no_of_delivered_orders
	FROM orders o
		JOIN restaurants r
			ON o.restaurant_id = r.restaurant_id
	GROUP BY 1, 2, 3
),
growth_ratio
AS(
	SELECT
		restaurant_id, restaurant_name, order_year_month, no_of_delivered_orders,
		LAG(no_of_delivered_orders) OVER(PARTITION BY restaurant_id ORDER BY order_year_month)
																					AS prev_month_delivered_orders
	FROM delivered_orders
)
SELECT 
	restaurant_id, restaurant_name, order_year_month, no_of_delivered_orders,
	prev_month_delivered_orders,
	(no_of_delivered_orders - prev_month_delivered_orders) * 100.0 / NULLIF(prev_month_delivered_orders, 0) AS growth_ratio
FROM growth_ratio;
```

### 12. Segment the customers into 'Gold' or 'Silver' groups based on their total spending compared to the average order value(AOV). If a customer total spending exceeds the AOV, label them as 'Gold', otherwise label them as 'Silver'. Determine each segment's total no. of orders and total revenue.

**Objective**

Classify customers into spending segments (Gold/Silver) to design personalized loyalty programs and targeted marketing campaigns.

```sql
WITH total_spend
AS(
	SELECT
		c.customer_id, c.customer_name,
		COALESCE(
		    SUM(o.total_amount)
		    FILTER (WHERE order_status='Completed'),
		    0
		) AS total_spending,
		COUNT(*) FILTER(WHERE o.order_status = 'Completed') AS total_orders
	FROM customers c
		LEFT JOIN orders o
			ON c.customer_id = o.customer_id
	GROUP BY c.customer_id, c.customer_name
)
SELECT
	CASE
		WHEN total_spending > (SELECT AVG(total_amount) FROM orders WHERE order_status = 'Completed') THEN 'Gold'
		ELSE 'Silver'
	END AS customer_category,
	SUM(total_spending) AS total_revenue,
	SUM(total_orders) AS total_orders
FROM total_spend
GROUP BY customer_category;
```

### 13. Calculate each rider's total monthly earning, assuming they earn 8% of the order amount:

**Objective**

Estimate monthly rider earnings for payroll processing, incentive planning, and workforce cost analysis.


```sql
SELECT
    r.rider_id,
    r.rider_name,
    DATE_TRUNC('MONTH', o.order_date) AS earning_month,
    SUM(o.total_amount * 0.08) AS total_monthly_earning
FROM orders o
JOIN deliveries d
    ON o.order_id = d.order_id
JOIN riders r
    ON d.rider_id = r.rider_id
GROUP BY
    r.rider_id,
    r.rider_name,
    DATE_TRUNC('MONTH', o.order_date)
ORDER BY
    earning_month,
    r.rider_id;
```

### 14. Find the no. of 5-star, 4-star and 3-star ratings of each rider.
### Riders recieve their rating based on the delivery time as follows:
### 1. if the order is delivered in less than 15-minutes of order-received time, the rider will get 5-star.
### 2. if they deliver in 16 to 20 minutes, they get 4-star.
### 3. if they deliver after 20 minutes, they get 3-star.

**Objective**

Measure rider service quality based on delivery speed to identify top performers and training opportunities.

```sql
WITH all_delivery_time
AS(
	SELECT
		r.rider_id, r.rider_name,
		CASE WHEN d.delivery_time < o.order_time
				THEN ((EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))) / 60)::INT + 1440
			 ELSE
				((EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))) / 60)::INT
		END AS delivery_minutes
	FROM riders r
		JOIN deliveries d
			ON r.rider_id = d.rider_id
		JOIN orders o
			ON d.order_id = o.order_id
)
SELECT
	rider_id, rider_name,
	COUNT(*) FILTER(WHERE delivery_minutes <= 15) AS Five_Stars,
	COUNT(*) FILTER(WHERE delivery_minutes <= 20 AND delivery_minutes > 15) AS Four_Stars,
	COUNT(*) FILTER(WHERE delivery_minutes > 20) AS Three_Stars
FROM all_delivery_time
GROUP BY 1, 2;
```

### 15. Analyze order frequency per day of the week and identify the peak day for each restaurant:

**Objective**

Identify each restaurant busiest weekday to optimize employee scheduling, inventory management, and promotional campaigns.

```sql
WITH order_day
AS(
	SELECT
		r.restaurant_id, r.restaurant_name,
		EXTRACT(ISODOW FROM o.order_date) AS day_of_week,
		TRIM(TO_CHAR(o.order_date, 'DAY')) AS day_name,
		COUNT(*) AS order_frequency,
		DENSE_RANK() OVER(PARTITION BY r.restaurant_id, r.restaurant_name ORDER BY COUNT(*) DESC) AS rank_lvl
	FROM orders o
		JOIN restaurants r
			ON o.restaurant_id = r.restaurant_id
	GROUP BY r.restaurant_id, r.restaurant_name, day_of_week, day_name
)
SELECT
	restaurant_id, restaurant_name, day_of_week, day_name, order_frequency
FROM order_day
WHERE rank_lvl = 1
ORDER BY restaurant_name, day_of_week;
```

### 16. Customer Lifetime Value(CLV): Calculate the total revenue generated by each customer over all their orders:

**Objective**

Calculate the total revenue generated by each customer to identify long-term high-value customers and improve retention strategies.

```sql
SELECT
	c.customer_id, c.customer_name, SUM(o.total_amount) AS total_revenue
FROM orders o
	JOIN customers c
		ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY total_revenue DESC;
```

### 17. Identify sales trends by comparing each month total sales to the previous month:

**Objective**

Analyze month-over-month sales performance to identify growth patterns, seasonality, and potential business risks.

```sql
WITH month_sales
AS(
	SELECT
		DATE_TRUNC('MONTH', order_date) AS order_year_month,
		SUM(total_amount) AS current_month_sales
	FROM orders
	GROUP BY order_year_month
),
previous_sales
AS(
	SELECT
		order_year_month, current_month_sales,
		LAG(current_month_sales) OVER (ORDER BY order_year_month) AS previous_month_sales
	FROM month_sales
)
SELECT
	order_year_month, current_month_sales, previous_month_sales,
	ROUND((current_month_sales - previous_month_sales) * 100.0 / NULLIF(previous_month_sales, 0), 2) AS sales_growth_%
FROM previous_sales
ORDER BY order_year_month;
```

### 18. Evaluate rider's efficiency by determining average delivery times in minutes and identifying those with the lowest and highest averages:

**Objective**

Identify the fastest and slowest riders to evaluate delivery efficiency, recognize top performers, and improve operational performance.


```sql
WITH riders_delivery_stats
AS(
	SELECT
		r.rider_id, r.rider_name,
		ROUND(
			AVG(
				CASE WHEN d.delivery_time < o.order_time
						THEN (EXTRACT(EPOCH FROM (d.delivery_time - o.order_time)))/60 + 1440
					 ELSE
						(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time)))/60
				END
			), 2
		) AS average_delivery_time
	FROM orders o
		JOIN deliveries d
			ON o.order_id = d.order_id
		JOIN riders r
			ON d.rider_id = r.rider_id
	GROUP BY r.rider_id, r.rider_name
),
limits
AS (
    SELECT
        MIN(average_delivery_time) AS min_time,
        MAX(average_delivery_time) AS max_time
    FROM riders_delivery_stats
)
SELECT
	rider_id, rider_name, average_delivery_time
FROM riders_delivery_stats rds
	CROSS JOIN limits l
		WHERE rds.average_delivery_time IN (l.min_time, l.max_time);
```		
		
### 19. Analyze monthly and seasonal ordering trends by calculating the total number of orders for each menu item, enabling the identification of popular items across different seasons.

**Objective**

Understand seasonal demand for menu items to optimize inventory, menu planning, and seasonal marketing campaigns.

```sql
WITH seasonal_orders
AS(
SELECT
	CASE
		WHEN EXTRACT(MONTH FROM order_date) IN (12, 1, 2) THEN 'Winter'
		WHEN EXTRACT(MONTH FROM order_date) IN (3, 4, 5) THEN 'Spring'
		WHEN EXTRACT(MONTH FROM order_date) IN (6, 7, 8) THEN 'Summer'
		WHEN EXTRACT(MONTH FROM order_date) IN (9, 10, 11) THEN 'Autumn'
	END AS season,
	EXTRACT(MONTH FROM order_date) AS order_month,
	TRIM(TO_CHAR(order_date, 'MONTH')) AS month_name,
	order_item,
	COUNT(order_id) AS total_orders
FROM orders
GROUP BY season, order_month, month_name, order_item
ORDER BY season, order_month, total_orders DESC
)
SELECT
	season, order_month, month_name, order_item, total_orders
FROM seasonal_orders
ORDER BY
	CASE season
		WHEN 'Winter' THEN 1
		WHEN 'Spring' THEN 2
		WHEN 'Summer' THEN 3
		WHEN 'Autumn' THEN 4
	END,
		order_month, total_orders DESC;
```

### 20. Rank each city based on the total revenue generated during the previous calendar year.

**Objective**

Rank cities based on revenue generation to identify high-performing markets and prioritize future business expansion

```sql
SELECT
	r.city, SUM(o.total_amount) AS total_revenue,
	RANK() OVER (ORDER BY SUM(o.total_amount) DESC) AS city_rank
FROM orders o
	JOIN restaurants r
		ON o.restaurant_id = r.restaurant_id
WHERE o.order_date >= DATE_TRUNC('YEAR', CURRENT_DATE) - INTERVAL '1 YEAR'
	AND o.order_date < DATE_TRUNC('YEAR', CURRENT_DATE)
GROUP BY r.city;
```

## Key Insights
• Highest revenue restaurants
• Peak ordering hours
• Customer segmentation
• Rider performance
• Seasonal trends

## Database Indexes
```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id);

CREATE INDEX idx_orders_restaurants
ON orders(restaurant_id);

CREATE INDEX idx_orders_date
ON orders(order_date);

CREATE INDEX idx_delivery_rider
ON deliveries(rider_id);
```

## Tools Used
- PostgreSQL
- pgAdmin 4
- Git & GitHub

## How to Run
1. Create the database.
2. Run schema.sql.
3. Import the dataset.
4. Create indexes.
5. Execute the queries.

## Author
Muhammad Umar
