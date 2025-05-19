
use adashi_staging;

-- 1. High-Value Customers with Multiple Products
/*Scenario: The business wants to identify customers who have both a savings and an
investment plan (cross-selling opportunity).
Task: Write a query to find customers with at least one funded savings plan AND one
funded investment plan, sorted by total deposits.*/


WITH CustomerPlans AS (
    SELECT
        u.id AS owner_id,
        u.name,
        COUNT(DISTINCT CASE WHEN p.plan_type_id = 'savings' THEN p.id END) AS savings_count,
        COUNT(DISTINCT CASE WHEN p.plan_type_id = 'investment' THEN p.id END) AS investment_count,
        SUM(CASE WHEN p.plan_type_id = 'savings' THEN s.new_balance ELSE 0 END) +
        SUM(CASE WHEN p.plan_type_id = 'investment' THEN s.new_balance ELSE 0 END) AS total_deposits
    FROM
        users_customuser u
    LEFT JOIN
        savings_savingsaccount s ON u.id = s.id
    LEFT JOIN
        plans_plan p ON s.plan_id = p.id
    WHERE
        s.new_balance > 0
    GROUP BY
        u.id, u.name
)
SELECT
    owner_id,
    name,
    savings_count,
    investment_count,
    total_deposits
FROM
    CustomerPlans
WHERE
    savings_count > 0 AND investment_count > 0
ORDER BY
    total_deposits DESC;    


    -- Explanation:
/*Joins: The query joins the users_customuser table with the savings_savingsaccount table on the user ID (u.id = s.id) and with the plans_plan table on the plan ID (s.plan_id = p.id).
Balance Filtering: The WHERE s.balance > 0 clause ensures that only accounts with a positive balance are considered, indicating they are funded.
Grouping: The GROUP BY u.id, u.name clause groups the results by customer ID and name.
Counting Plan Types: The HAVING clause ensures that each customer has at least one savings plan and one investment plan by counting distinct plan types.
Sorting: The ORDER BY total_balance DESC clause sorts the results by the total balance in descending order, highlighting customers with the highest total deposits.*/
    
        

-- 2 Transaction Frequency Analysis:
/*Scenario: The finance team wants to analyze how often customers transact to segment
them (e.g., frequent vs. occasional users).
Task: Calculate the average number of transactions per customer per month and
categorize them:
● "High Frequency" (≥10 transactions/month)
● "Medium Frequency" (3-9 transactions/month)
● "Low Frequency" (≤2 transactions/month)*/
   
   
    WITH MonthlyTransactions AS (
    SELECT
        u.id AS customer_id,
        COUNT(u.id) AS transactions_per_month,
        EXTRACT(YEAR FROM s.transaction_date) * 12 + EXTRACT(MONTH FROM s.transaction_date) AS 'year_month'
    FROM
        users_customuser u
    JOIN
        savings_savingsaccount s ON u.id = s.id
    WHERE
        s.transaction_date >= u.signup_device
    GROUP BY
        u.id, 'year_month'
),
CustomerFrequency AS (
    SELECT
        customer_id,
        AVG(transactions_per_month) AS avg_transactions_per_month
    FROM
        MonthlyTransactions
    GROUP BY
        customer_id
)
SELECT
    CASE
        WHEN avg_transactions_per_month >= 10 THEN 'High Frequency'
        WHEN avg_transactions_per_month BETWEEN 3 AND 9 THEN 'Medium Frequency'
        ELSE 'Low Frequency'
    END AS frequency_category,
    COUNT(customer_id) AS customer_count,
    ROUND(AVG(avg_transactions_per_month), 1) AS avg_transactions_per_month
FROM
    CustomerFrequency
GROUP BY
    frequency_category
ORDER BY
    frequency_category DESC;
    
    -- Explanation:
/*MonthlyTransactions CTE: This Common Table Expression (CTE) calculates the number of transactions each customer made per month. It joins the users_customuser table with the savings_savingsaccount table on the user ID and filters transactions that occurred after the customer's signup date. The EXTRACT function is used to derive the year and month from the transaction date, creating a unique identifier (year_month) for each month.
CustomerFrequency CTE: This CTE calculates the average number of transactions per month for each customer by averaging the transactions_per_month from the MonthlyTransactions CTE.
Final SELECT: The final query categorizes customers based on their average transactions per month:
"High Frequency" for customers with 10 or more transactions per month.
"Medium Frequency" for customers with 3 to 9 transactions per month.
"Low Frequency" for customers with 2 or fewer transactions per month.
It then counts the number of customers in each category and calculates the average number of transactions per month for each category. The results are ordered by frequency_category in descending order.*/
    
    
    
    
    -- 3. . Account Inactivity Alert
/*Scenario: The ops team wants to flag accounts with no inflow transactions for over one
year.
Task: Find all active accounts (savings or investments) with no transactions in the last 1
year (365 days).*/


WITH LastTransaction AS (
    SELECT
        s.plan_id,
        MAX(s.transaction_date) AS last_transaction_date
    FROM
        savings_savingsaccount s
    GROUP BY
        s.plan_id
)
SELECT
    p.plan_type_id,
    p.owner_id,
    p.plan_type_id,
    lt.last_transaction_date,
    DATEDIFF(CURRENT_DATE, lt.last_transaction_date) AS inactivity_days
FROM
    plans_plan p
JOIN
    LastTransaction lt ON p.plan_type_id = lt.plan_id
WHERE
    lt.last_transaction_date <= CURRENT_DATE - INTERVAL 365 DAY
ORDER BY
    inactivity_days DESC;

    -- Explanation:
/*LastTransaction CTE: This Common Table Expression (CTE) calculates the most recent transaction date for each plan by selecting the maximum transaction_date for each plan_id from the savings_savingsaccount table.
Main Query: The main query joins the plans_plan table with the LastTransaction CTE on plan_id to retrieve the plan details. It then calculates the number of days since the last transaction using the DATEDIFF function.
WHERE Clause: Filters the results to include only those plans where the last transaction occurred 365 days ago or more.
ORDER BY: Sorts the results by inactivity_days in descending order, highlighting the most inactive accounts.*/
    
    
   -- 4.  Customer Lifetime Value (CLV) Estimation
/*Scenario: Marketing wants to estimate CLV based on account tenure and transaction
volume (simplified model).
Task: For each customer, assuming the profit_per_transaction is 0.1% of the transaction
value, calculate:
● Account tenure (months since signup)
● Total transactions
● Estimated CLV (Assume: CLV = (total_transactions / tenure) * 12 *
avg_profit_per_transaction)
● Order by estimated CLV from highest to lowest*/

    WITH UserTransactions AS (
    SELECT
        u.id AS user_id,
        u.signup_device,
        COUNT(s.id) AS total_transactions,
        SUM(s.confirmed_amount - s.deduction_amount) AS total_transaction_value
    FROM
        users_customuser u
    LEFT JOIN
        savings_savingsaccount s ON u.id = s.owner_id
    WHERE
        s.confirmed_amount > 0  -- Consider only inflow transactions
    GROUP BY
        u.id, u.signup_device
),
CLVCalculation AS (
    SELECT
        ut.user_id,
        TIMESTAMPDIFF(MONTH, ut.signup_device, CURRENT_DATE) AS tenure_months,
        ut.total_transactions,
        ut.total_transaction_value,
        (ut.total_transaction_value * 0.001) / ut.total_transactions AS avg_profit_per_transaction
    FROM
        UserTransactions ut
    WHERE
        ut.total_transactions > 0
)
SELECT
    c.user_id,
    c.tenure_months,
    c.total_transactions,
    c.avg_profit_per_transaction,
    (c.total_transactions / c.tenure_months) * 12 * c.avg_profit_per_transaction AS estimated_clv
FROM
    CLVCalculation c
ORDER BY
    estimated_clv DESC;
    
    
    -- Explanation
/*UserTransactions CTE: This Common Table Expression calculates the total number of transactions and the total transaction value (in kobo) for each user. It joins the users_customuser table with the savings_savingsaccount table, considering only inflow transactions (confirmed_amount > 0).
CLVCalculation CTE: This CTE computes the account tenure in months and the average profit per transaction for each user. The average profit per transaction is calculated as 0.1% of the total transaction value divided by the total number of transactions.
Final SELECT: The main query calculates the estimated CLV using the provided formula and orders the results by estimated CLV in descending order.*/
