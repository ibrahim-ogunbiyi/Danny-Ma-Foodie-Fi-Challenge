# Danny Ma Case Study 3 Foodie-Fi

## Data Analysis Question

### 1. How many customers has Foodie-Fi ever had?

#### Steps for Achieving the Result:

Use the DISTINCT and COUNT function to count the number of customers in the subscription table.

```sql
SELECT COUNT( DISTINCT customer_id) AS total_customers
FROM subscriptions; 
```

#### Result
|total_customers|
| -- |
|1000|

### 2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value.

#### Steps for Achieving the Result:

Filter the subscription table by plan_id = 0 which stands for trial, then utilize the EXTRACT function in PostgreSQL to extract month from date, and count the number of records. Also lastly GROUP BY the month extracted.


```sql
SELECT 
	EXTRACT(month from start_date) AS month,
	COUNT(*) AS no_of_subcriptions
FROM subscriptions
WHERE plan_id = 0
GROUP BY month
ORDER BY month;
```
#### Result
| month | no_of_subcriptions |
|-------|--------------------|
| 1     | 88                 |
| 2     | 68                 |
| 3     | 94                 |
| 4     | 81                 |
| 5     | 88                 |
| 6     | 79                 |
| 7     | 89                 |
| 8     | 88                 |
| 9     | 87                 |
| 10    | 79                 |
| 11    | 75                 |
| 12    | 84                 |


### 3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name

**Steps for Achieving the Result:**
Find the next start_date for each plan after 2021 by finding the minimum start_date and also join with the number of events that happen in each plan after 2020

```sql
WITH CTE1 AS
(SELECT 
	plan_name,
	MIN(start_date) AS next_start_date
FROM subscriptions
INNER JOIN plans
USING (plan_id)
WHERE  DATE_PART('year', start_date) > 2020
GROUP BY plan_name
),CTE2 AS(
SELECT
	plan_name,
	COUNT(*) AS event_counts
FROM subscriptions
INNER JOIN plans
USING(plan_id)
WHERE DATE_PART('year', start_date) > 2020
GROUP BY plan_name
)

SELECT *
FROM CTE1
INNER JOIN CTE2
USING(plan_name)
ORDER BY next_start_date;
```

**Result**
| plan_name      | next_start_date | event_counts |
| -------------- | --------------- | ------------ |
| pro monthly    | 2021-01-01      | 60           |
| basic monthly  | 2021-01-01      | 8            |
| pro annual     | 2021-01-03      | 63           |
| churn          | 2021-01-03      | 71           |

### 4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

#### Steps for Achieving the Result:

Use the DISTINCT and COUNT function with CASE to count the number of churned customers. Also divide the count by the total number of customers and round the value to one decimal place.

#### Result
```sql
SELECT 
	COUNT(DISTINCT CASE WHEN plan_id = 4 THEN customer_id END) AS no_of_churned_customer,
	ROUND(SUM(CASE WHEN plan_id = 4 THEN 1 END) 
	/(SELECT COUNT(DISTINCT customer_id)::NUMERIC FROM subscriptions) * 100, 1) AS percentage_of_churned_customer
FROM subscriptions;
```
| no_of_churned_customer | percentage_of_churned_customer |
|------------------------|-------------------------------|
| 307                    | 30.7                          |


### 5. How many customers have churned straight after their initial free trial? What percentage is this rounded to the nearest whole number?

#### Steps for Achieving the Result:
Utilize LEAD the next plan by a particular customer and then use CASE to get percentage of customer who transitioned straight from their current plan to churn out of all customers

```sql
SELECT 
	ROUND(COUNT(DISTINCT CASE WHEN (plan_name = 'trial') AND (next_subscription = 'churn')
	THEN customer_id END) /
	(SELECT COUNT(DISTINCT customer_id)::NUMERIC FROM subscriptions) * 100) AS pct_of_churned_customers
FROM(
SELECT 
	*,
	LEAD(plan_name)OVER(partition by customer_id ORDER BY start_date) AS next_subscription
FROM subscriptions
INNER JOIN plans
USING (plan_id)
ORDER BY customer_id) AS result;
```
#### Result
|pct_of_churned_customers|
| -- |
|9|  

### 6. What is the number and percentage of customer plans after their initial free trial?
Utilize LEAD to find next_plan and then GROUP BY customer to find the number of customers to find number and percentage of customers after their initial plan

**Steps for Achieving the Result:**

```sql
WITH CTE1 AS(
SELECT 
	*,
	LEAD(plan_id) OVER(PARTITION BY customer_id ORDER BY start_date) AS next_plan
FROM subscriptions
),
CTE2 AS(
SELECT
	plan_name,
	COUNT(DISTINCT customer_id) AS no_of_customers,
	ROUND(((100.0 * COUNT(DISTINCT customer_id))
	/ (SELECT COUNT(DISTINCT customer_id) FROM subscriptions)),1) AS pct_customers
FROM CTE1 AS c
INNER JOIN plans AS p
ON c.next_plan = p.plan_id
WHERE c.plan_id = 0 AND  c.next_plan is not null
GROUP BY plan_name

)

SELECT * 
FROM CTE2
```
**Result**
| plan_name      | no_of_customers | pct_customers |
| -------------- | --------------- | ------------- |
| basic monthly  | 546             | 54.6          |
| churn          | 92              | 9.2           |
| pro annual     | 37              | 3.7           |
| pro monthly    | 325             | 32.5          |


### 7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?


**Steps for Achieving the Result:**

```sql
SELECT 
	plan_name,
	COUNT(customer_id) AS no_of_customers,
	ROUND(((100.0 * COUNT(customer_id))
	/ (SELECT COUNT(DISTINCT customer_id) FROM subscriptions)),1) AS pct_customers
FROM subscriptions
INNER JOIN plans
USING(plan_id)
WHERE start_date = '2020-12-31'
GROUP BY plan_name;
```

**Result**
| plan_name | no_of_customers | pct_customers |
|-----------|-----------------|---------------|
| churn     | 1               | 0.1           |

### 8. How many customers have upgraded to an annual plan in 2020?

**Steps for Achieving the Result:**

Use CASE to count number of customers that upgraded to annual plan in 2020
```sql
SELECT
	COUNT(DISTINCT CASE WHEN plan_name LIKE '%annual%' THEN customer_id END) AS no_of_customers
FROM subscriptions
INNER JOIN plans
USING(plan_id)
WHERE DATE_PART('Year', start_date) = 2020;
```
**Result**
|no_of_customers|
| -- |
|195| 

### 9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?

USE LEAD to get next plan and then minus min start_date from next subscription date

**Steps for Achieving the Result:**


```sql
WITH CTE1 AS
(
SELECT
	*,
	LEAD(plan_name)OVER(PARTITION BY customer_id ORDER BY start_date) AS next_subscription_plan,
	LEAD(start_date)OVER(PARTITION BY customer_id ORDER BY start_date) AS next_subscription_date
FROM subscriptions
INNER JOIN plans
USING(plan_id)
),

CTE2 AS(
SELECT
	*,
	ROUND(CASE WHEN next_subscription_plan LIKE '%annual%' THEN 
	next_subscription_date END
		  - MIN(start_date)over(partition by customer_id)) AS days
FROM CTE1
)

SELECT ROUND(AVG(days)) AS average_days
FROM (
	SELECT customer_id,
	days
	FROM CTE2
) AS Subquery
```
**Result**
|average_days|
| -- |
|105| 

### 10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)

**Steps for Achieving the Result:**

```sql

WITH CTE1 AS
(
SELECT
	*,
	LEAD(plan_name)OVER(PARTITION BY customer_id ORDER BY start_date) AS next_subscription_plan,
	LEAD(start_date)OVER(PARTITION BY customer_id ORDER BY start_date) AS next_subscription_date
FROM subscriptions
INNER JOIN plans
USING(plan_id)
),

CTE2 AS(
SELECT
	*,
	ROUND(CASE WHEN next_subscription_plan LIKE '%annual%' THEN 
	next_subscription_date END
		  - MIN(start_date)over(partition by customer_id)) AS days
FROM CTE1
)

SELECT 
	CASE WHEN AVG(days) BETWEEN 0 AND 30 THEN '0-30 days'
		WHEN AVG(days) BETWEEN 31 AND 60 THEN '31-60 days'
		WHEN AVG(days) BETWEEN 61 AND 90 THEN '61-90 days'
		WHEN AVG(days) BETWEEN 91 AND 120 THEN '91-120 days'
		WHEN AVG(days) BETWEEN 121 AND 150 THEN '121-150 days'
		WHEN AVG(days) BETWEEN 151 AND 180 THEN '151-180 days'
		WHEN AVG(days) BETWEEN 181 AND 210 THEN '181-210 days'
		WHEN AVG(days) BETWEEN 151 AND 180 THEN '151-180 days'
		WHEN AVG(days) BETWEEN 211 AND 240 THEN '211-240 days'
		WHEN AVG(days) BETWEEN 241 AND 270 THEN '241-270 days'
		WHEN AVG(days) BETWEEN 271 AND 300 THEN '271-300 days'
		WHEN AVG(days) BETWEEN 301 AND 330 THEN '301-330 days'
		WHEN AVG(days) BETWEEN 331 AND 360 THEN '331-360 days'
		END AS periods
FROM (
	SELECT customer_id,
	days
	FROM CTE2
) AS Subquery
```

**Result**
|periods|
| -- |
|91-120 days| 

### 11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

Use LEAD to know customers who downgraded from pro monthly to basic month in 2020

**Steps for Achieving the Result:**

```sql
WITH CTE1 AS
(
SELECT 
	*,
	LEAD(plan_id)OVER(PARTITION BY customer_id ORDER BY start_date) AS next_subscription_plan
FROM subscriptions
WHERE  DATE_PART('Year', start_date) = 2020
)

SELECT COUNT(DISTINCT customer_id) AS no_of_customers
FROM CTE1
WHERE plan_id = 2 AND next_subscription_plan = 1;
```

**Result**
|no_of_customers|
| -- |
|0| 
