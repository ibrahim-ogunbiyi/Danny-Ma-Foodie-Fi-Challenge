# 🍐Danny Ma Case Study 3 Foodie-Fi🍐

## Challenge C

The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:

- monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
- upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
- upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period
- once a customer churns they will no longer make payments


#### Steps for Achieving the Result:
```sql
SET search_path = foodie_fi;

WITH RECURSIVE CTE1 AS (
    SELECT 
        customer_id, 
        plan_id,
        plan_name,
        start_date AS payment_date,
        price AS amount
    FROM subscriptions
    INNER JOIN plans USING(plan_id)
    WHERE plan_name != 'trial' 
    AND DATE_PART('year', start_date) = 2020
    ORDER BY customer_id, start_date
), CTE2 AS (
    SELECT 
        customer_id, 
        plan_id,
        plan_name,
        payment_date,
        amount,
		COALESCE(LEAD(payment_date) OVER(PARTITION BY customer_id ORDER BY payment_date), '1900-01-01') AS next_date
    FROM CTE1
    UNION ALL
    SELECT 
        customer_id, 
        plan_id,
        plan_name,
        (payment_date + INTERVAL '1 month')::DATE AS payment_date,
        amount,
		next_date
    FROM CTE2
    WHERE (DATE_PART('month', payment_date) < 12 AND
        (plan_name = 'basic monthly' OR plan_name = 'pro monthly')
		 AND DATE_PART('month', payment_date)+1 != DATE_PART('month', next_date))
)

SELECT 
	customer_id,
	plan_id,
	plan_name,
	payment_date,
	CASE WHEN  LAG (plan_name)
	OVER(PARTITION BY customer_id ORDER BY payment_date) = 'basic monthly'
	AND plan_name = 'pro annual'
	THEN amount - 9.90 
	ELSE amount END AS amount,
	RANK()OVER(Partition BY customer_id ORDER BY payment_date) AS payment_order
FROM CTE2
WHERE plan_id !=4
ORDER BY customer_id, payment_date
```
**Result**

| customer_id | plan_id | plan_name      | payment_date | amount | payment_order |
|-------------|---------|---------------|--------------|--------|---------------|
| 1           | 1       | basic monthly | 2020-08-08   | 9.90   | 1             |
| 1           | 1       | basic monthly | 2020-09-08   | 9.90   | 2             |
| 1           | 1       | basic monthly | 2020-10-08   | 9.90   | 3             |
| 1           | 1       | basic monthly | 2020-11-08   | 9.90   | 4             |
| 1           | 1       | basic monthly | 2020-12-08   | 9.90   | 5             |
| 2           | 3       | pro annual    | 2020-09-27   | 199.00 | 1             |
| 3           | 1       | basic monthly | 2020-01-20   | 9.90   | 1             |
| 3           | 1       | basic monthly | 2020-02-20   | 9.90   | 2             |
| 3           | 1       | basic monthly | 2020-03-20   | 9.90   | 3             |
| 3           | 1       | basic monthly | 2020-04-20   | 9.90   | 4             |
| 3           | 1       | basic monthly | 2020-05-20   | 9.90   | 5             |
| 3           | 1       | basic monthly | 2020-06-20   | 9.90   | 6             |
| 3           | 1       | basic monthly | 2020-07-20   | 9.90   | 7             |
| 3           | 1       | basic monthly | 2020-08-20   | 9.90   | 8             |
|-------------|---------|---------------|--------------|--------|---------------|
| 998         | 2       | pro monthly   | 2020-10-19   | 19.90  | 1             |
| 998         | 2       | pro monthly   | 2020-11-19   | 19.90  | 2             |
| 998         | 2       | pro monthly   | 2020-12-19   | 19.90  | 3             |
| 999         | 2       | pro monthly   | 2020-10-30   | 19.90  | 1             |
| 999         | 2       | pro monthly   | 2020-11-30   | 19.90  | 2             |
| 1000        | 2       | pro monthly   | 2020-03-26   | 19.90  | 1             |
| 1000        | 2       | pro monthly   | 2020-04-26   | 19.90  | 2             |
| 1000        | 2       | pro monthly   | 2020-05-26   | 19.90  | 3             |
