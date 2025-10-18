# Noon Food Launch SQL Analytics - Fabric
On 1st May 2024, Noon — one of the largest e-commerce companies in the Middle East — launched a new Food Delivery vertical in Dubai.
As part of the Data Analytics team, I was tasked to extract, analyze, and interpret key performance metrics from the company’s order data to evaluate the launch success and provide insights for the Growth and Product teams. All the Analysis and SQL Queries were done in Microsoft Fabric.

# Metrics Framework

| Type                  | Metric                           | Description                                    |
| --------------------- | -------------------------------- | ---------------------------------------------- |
| **Primary Metrics**   |  Number of Orders              | Volume of orders placed daily since launch     |
|                       |  Revenue per Day / per Cuisine | Key performance indicator of customer spending |
|                       |  New Customer Acquisition Rate | Growth in new customers post-launch            |
| **Secondary Metrics** |  Repeat Purchase Rate          | % of customers making multiple purchases       |
|                       |  Promo vs Organic Split        | Share of users acquired via promos             |
|                       |  Dormant Customer Count        | Users inactive in the last 7 days              |
| **Guardrail Metrics** |  Promo Dependency              | % of customers transacting only via promos     |
|                       |  Customer Churn                | Users who haven’t returned after first order   |


# Data Model
Table Given to Analyze Key Insights 

<img width="430" height="153" alt="Image" src="https://github.com/user-attachments/assets/617bd2e5-65ca-4d18-8976-c6d4bbc75ffc" />

# Business Questions To be Answered :
1. Find Top 3 Outlets By Cuisine Type.

2. Find the daily new customer count from the launch date (everyday how many new customers are we acquiring).

3. Count of all the users who were acquired in Jan 2025 and only placed one order in JAN and did not place any other order.

4. List all the customers with no order in the last 7 days but were acquired one month ago with their first order on promo.

5. Growth Team is planning to create a trigger that will target customers after every third order with a personalized communication and they have asked you to create a query for this.

6. List customers who have placed more than 1 order and all their orders on a promo only.

7. What percent of customers were organically acquired in Jan 2025 (placed their first order on promo code).


# SQL Analysis:

### Find Top 3 Outlets by Cuisine Type

```sql
SELECT * 
FROM (
    SELECT 
        [Cuisine],
        [Restaurant_id],
        COUNT(*) AS total_orders,
        DENSE_RANK() OVER (
            PARTITION BY [Cuisine] 
            ORDER BY COUNT(*) DESC
        ) AS Top_3_Restaurants
    FROM [WareHouse_Portfolio_Project].[restaurant].[orders]
    GROUP BY [Cuisine], [Restaurant_id]
) rk
WHERE rk.Top_3_Restaurants <= 3;
```
**Insight:** Italian and Lebanese cuisines emerged as the top-performing categories, with outlets **PIZZA123** and **KMKMH6787** leading overall — indicating strong customer preference for these cuisines.

### Find the daily new customer count from the launch date (everyday how many new customers are we aquiring)

```sql
SELECT 
rc.first_order_date , 
COUNT(*) as new_customers FROM  (
    SELECT [Customer_code] , CAST(MIN([Placed_at]) AS DATE) as first_order_date 
    FROM [WareHouse_Portfolio_Project].[restaurant].[orders]
    GROUP BY [Customer_code]
) as rc
GROUP BY rc.first_order_date
ORDER BY rc.first_order_date
```

### Count of all the users who were acquired in Jan 2025 and only placed one order in JAN and did not Place Any Other Order

```sql
SELECT 
    Customer_code,
    COUNT(*) AS Total_Orders
    FROM 
    [WareHouse_Portfolio_Project].[restaurant].[orders]
WHERE Customer_code NOT IN (SELECT 
                                DISTINCT Customer_code
                                FROM [WareHouse_Portfolio_Project].[restaurant].[orders]
                                WHERE MONTH(Placed_at) != 1 AND YEAR(Placed_at) = 2025
) AND MONTH(Placed_at) = 1 AND YEAR(Placed_at) = 2025
GROUP BY Customer_code
HAVING  COUNT(*) = 1
```

### List all the customers with no order in the last 7 days but were acquired one month ago with their first order on promo.

```sql
WITH cte_promo as (
SELECT 
    Customer_code,
    MIN(Placed_at)  as first_order_date,
    MAX(Placed_at)  as last_order_date
FROM 
[WareHouse_Portfolio_Project].[restaurant].[orders]
GROUP BY Customer_code
)
SELECT ct.*,o.[Promo_code_Name]
FROM [WareHouse_Portfolio_Project].[restaurant].[orders] o
INNER JOIN cte_promo as ct ON ct.Customer_code = o.Customer_code AND ct.first_order_date = o.Placed_at
WHERE last_order_date < DATEADD(DAY,-7,GETDATE()) AND 
ct.first_order_date < DATEADD(MONTH,-1,GETDATE()) AND o.[Promo_code_Name] IS NOT NULL
```

### Growth Team is planning to create a trigger that will target customers after every third order with a personalized communication and they have asked you to create a query for this.

```sql
WITH CTE_FT AS (
SELECT 
    [Customer_code] , 
    [Placed_at] ,
    ROW_NUMBER() OVER(PARTITION BY [Customer_code]  ORDER BY [Placed_at] ASC) as row_num
FROM 
[WareHouse_Portfolio_Project].[restaurant].[orders]
)

SELECT * FROM CTE_FT
where row_num % 3 = 0;
```

### List customers who have placed more than 1 order and all their orders on a promo only.

```sql
SELECT 
    [Customer_code],
    COUNT(*) as total_orders,
    COUNT([Promo_code_Name]) AS total_promo_number
FROM 
[WareHouse_Portfolio_Project].[restaurant].[orders] 
GROUP BY [Customer_code]
HAVING COUNT(*) > 1 AND COUNT(*) = COUNT([Promo_code_Name]);
```

### What percent of customers were organically acquired in Jan 2025 (placed their first order on promo code).

```sql


```
