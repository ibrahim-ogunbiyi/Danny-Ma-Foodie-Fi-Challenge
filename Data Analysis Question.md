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

#### Result
```sql
SELECT 
	EXTRACT(month from start_date) AS month,
	COUNT(*) AS no_of_subcriptions
FROM subscriptions
WHERE plan_id = 0
GROUP BY month
ORDER BY month;
```

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

#### Result
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

|pct_of_churned_customers|
| -- |
|9|  
