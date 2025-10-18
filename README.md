# Noon Food Launch SQL Analytics - Fabric
On 1st May 2024, Noon — one of the largest e-commerce companies in the Middle East — launched a new Food Delivery vertical in Dubai.
As part of the Data Analytics team, I was tasked to extract, analyze, and interpret key performance metrics from the company’s order data to evaluate the launch success and provide insights for the Growth and Product teams. All the Analysis and SQL Queries were done in Microsoft Fabric.

# Metrics Framework

| Metric Category            | Metric                        | Purpose / Insight                                        |
| -------------------------- | ----------------------------- | -------------------------------------------------------- |
| **Acquisition**            | Daily New Customers           | Measures growth rate and campaign effectiveness          |
|                            | Promo vs. Organic Acquisition | Tracks dependency on discounts vs. organic growth        |
|                            | Monthly New Customer Count    | Monitors month-over-month growth                         |
| **Engagement & Retention** | One-Time Customers            | Identifies churn-prone users                             |
|                            | Repeat Customers              | Tracks loyal users with multiple orders                  |
|                            | Promo-Dependent Customers     | Highlights users relying exclusively on discounts        |
|                            | Time to 2nd / 3rd Order       | Measures customer engagement speed                       |
| **Order Metrics**          | Total Orders per Customer     | Depth of engagement per user                             |
|                            | Orders by Cuisine / Outlet    | Identifies popular cuisines and outlets                  |
|                            | Promo vs. Non-Promo Orders    | Measures discount dependency                             |
|                            | Top 3 Outlets per Cuisine     | Highlights top-performing outlets                        |
| **Cohort Metrics**         | Cohort Retention              | Tracks how many customers return after acquisition month |
|                            | 3rd Order Trigger Candidates  | Identifies users eligible for targeted communications    |
| **High-Level Metrics**     | Organic Acquisition %         | Indicates share of non-promo users                       |
|                            | Promo Dependency %            | Measures reliance on promotional incentives              |
|                            | Churn Potential               | Monitors customers inactive for 7+ days                  |



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
   
# Executive Summary :

# SQL EDA Analysis:

### Find Top 3 Outlets by Cuisine Type
**Insight:** Italian and Lebanese cuisines emerged as the top-performing categories, with outlets **PIZZA123** and **KMKMH6787** leading overall — indicating strong customer preference for these cuisines.
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


### Find the daily new customer count from the launch date (everyday how many new customers are we aquiring)
**Insights :** Customer acquisition showed steady but modest daily growth since launch, with a few spikes on **Jan 1, Jan 5, Jan 10, and Jan 31,** suggesting the impact of initial promotions or marketing pushes during these dates. After January, **acquisition slowed significantly,** with only sporadic new users in February and March, indicating a drop in campaign momentum or reduced marketing visibility.

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
**Insights:** **A total of 33 customers were acquired in January 2025** but placed only one order that month and did not return, indicating low repeat engagement among newly acquired users. This pattern suggests the need to **analyze onboarding and post-purchase retention efforts** — especially to understand if these customers were promo-driven or lacked incentives for reordering.

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
**Insights :** Several customers (e.g., LMN9876543210JKL, HIJ9876543210DEF, SINGLE_ORDER_JAN) were acquired over a month ago with their first order on a promo but have shown **no activity in the last 7 days**, indicating a potential churn segment. These users are **promo-sensitive and should be re-engaged through retention campaigns or loyalty offers** to drive repeat purchases.

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
**Insights:** Multiple customers such as **THIRD_ORDER_CUST1, THIRD_ORDER_CUST2, and UVW7890123456JKL** have reached or crossed their third order milestone, making them ideal targets for personalized post-3rd order engagement campaigns. This segment represents high retention potential customers, where timely communication can boost loyalty and repeat purchase frequency.

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
**Insights :** Only a small subset of customers **(e.g., UVW7890123456JKL, DEF9876543210XYZ)** have placed **multiple orders exclusively using promo codes,** indicating a **promo-dependent customer segment** that may require strategies to encourage **full-price purchases or loyalty-based retention.**

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
**Insights** : Around **43% of customers were organically acquired** in January 2025, meaning the majority of new users still relied on promo codes — highlighting an opportunity to **reduce promo dependency and strengthen organic acquisition channels.**

```sql
WITH CTE_organic as (
        SELECT 
        [Customer_code] ,
        [Placed_at],
        [Promo_code_Name],
        ROW_NUMBER() OVER(PARTITION BY [Customer_code]  ORDER BY [Placed_at] ASC) as row_num 
        FROM [WareHouse_Portfolio_Project].[restaurant].[orders]
        WHERE MONTH([Placed_at]) = 1 AND YEAR([Placed_at]) = 2025
)

SELECT 
    COUNT( case when row_num = 1 AND [Promo_code_Name] IS NULL THEN [Customer_code] END)* 100  / COUNT(DISTINCT [Customer_code])  as pct
FROM CTE_organic 

```
