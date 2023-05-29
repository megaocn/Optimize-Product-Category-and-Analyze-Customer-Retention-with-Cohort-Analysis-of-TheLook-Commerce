# SQL Code

## **What is/are the categories with the lowest revenue growth in the past 1 year?**

WITH revenue AS (SELECT category, EXTRACT(MONTH FROM shipped_at) as month_num, FORMAT_DATE('%B', date(shipped_at)) as month_name, ROUND(SUM(sale_price),2) as revenue_per_month <br>
FROM `sql-project-376612.thelook_ecommerce.order_items` as order_items <br>
INNER JOIN `sql-project-376612.thelook_ecommerce.products` as products <br>
ON order_items.product_id = products.id <br>
WHERE status = 'Complete' <br>
AND date(shipped_at) >= DATE_SUB(DATE '2023-01-01', interval 1 year) AND date(shipped_at) < '2023-01-01' <br>
GROUP BY 1, 2, 3 <br>
ORDER BY 1, 2 DESC), <br>
<br>
revenue_lag AS ( <br>
SELECT category, month_num, month_name, revenue_per_month, <br>
LAG(revenue_per_month, 1) OVER(PARTITION BY category ORDER BY month_num) as LAG_revenue, ((revenue_per_month - (LAG(revenue_per_month, 1) OVER(PARTITION BY category ORDER BY month_num))) / LAG(revenue_per_month, 1) OVER(PARTITION BY category ORDER BY month_num)) * 100 as growth <br>
FROM revenue <br>
ORDER BY 1, 2 DESC) <br>
<br>
SELECT revenue.category, ROUND(AVG(revenue_lag.growth),2) as average_growth_category <br>
FROM revenue <br>
JOIN revenue_lag <br>
ON revenue.category = revenue_lag.category <br>
AND revenue.month_num = revenue_lag.month_num <br>
AND revenue.month_name = revenue_lag.month_name <br>
GROUP BY 1 <br>
ORDER BY 2 <br>
<br>
## **What is/are the categories with the lowest profit growth in the past 1 year?** <br>
WITH profit AS (SELECT category, EXTRACT(MONTH FROM shipped_at) as month_num, FORMAT_DATE('%B', shipped_at) as month_name, ROUND(SUM(sale_price - cost),2) as profit_per_month <br>
FROM `sql-project-376612.thelook_ecommerce.order_items` as order_items <br>
INNER JOIN `sql-project-376612.thelook_ecommerce.products` as products <br>
ON order_items.product_id = products.id <br>
WHERE status = 'Complete' <br>
AND date(shipped_at) >= DATE_SUB(DATE '2023-01-01', interval 1 year) AND date(shipped_at) < '2023-01-01' <br>
GROUP BY 1, 2, 3 <br>
ORDER BY 1, 2 DESC), <br>
<br>
profit_lag AS ( <br>
SELECT category, month_num, month_name, profit_per_month, <br>
LAG(profit_per_month, 1) OVER(PARTITION BY category ORDER BY month_num) as LAG_profit, ((profit_per_month - (LAG(profit_per_month, 1) OVER(PARTITION BY category ORDER BY month_num))) / (LAG(profit_per_month, 1) OVER(PARTITION BY category ORDER BY month_num))) * 100 as profit_growth <br>
FROM profit <br>
ORDER BY 1, 2 DESC) <br>
<br>
SELECT profit.category, ROUND(AVG(profit_lag.profit_growth),2) as average_growth_profit <br>
FROM profit <br>
JOIN profit_lag <br>
ON profit.category = profit_lag.category <br>
AND profit.month_num = profit_lag.month_num <br>
AND profit.month_name = profit_lag.month_name <br>
GROUP BY 1 <br>
ORDER BY 2 <br>

## **Cohort analysis to understand customer retention rate** 
WITH cohort_item AS ( <br>
SELECT user_id, min(date_trunc(date(created_at), month)) as cohort_month <br>
FROM `sql-project-376612.thelook_ecommerce.orders` <br>
WHERE date(created_at) >= DATE_SUB(DATE '2023-01-01', interval 1 year) AND date(created_at) < '2023-01-01' <br>
GROUP BY 1), <br>
<br> 
user_activities AS ( <br>
SELECT orders.user_id, DATE_DIFF(date_trunc(date(created_at), month), cohort_month, month) as different_time <br>
FROM `sql-project-376612.thelook_ecommerce.orders` as orders <br>
LEFT JOIN cohort_item as cohort <br>
ON orders.user_id = cohort.user_id <br>
WHERE date(created_at) >= DATE_SUB(DATE '2023-01-01', interval 1 year) AND date(created_at) < '2023-01-01' <br>
group by 1,2), <br>
<br>
cohort_size AS ( <br>
SELECT cohort_month, COUNT(DISTINCT user_id) as first_purchase_users <br>
FROM cohort_item <br>
GROUP BY 1 <br>
ORDER BY 1), <br>
<br>
retention AS ( <br>
SELECT cohort_month, different_time, COUNT(DISTINCT user.user_id) as total_users <br>
FROM user_activities as user <br>
LEFT JOIN cohort_item as cohort <br>
ON user.user_id = cohort.user_id <br>
GROUP BY 1,2) <br>
<br>
SELECT r.cohort_month, c.first_purchase_users, r.different_time, r.total_users, (r.total_users/c.first_purchase_users) as percentage <br>
FROM retention as r <br>
LEFT JOIN cohort_size as c <br>
ON r.cohort_month = c.cohort_month <br>
WHERE r.cohort_month IS NOT NULL <br>
ORDER BY 1, 3
