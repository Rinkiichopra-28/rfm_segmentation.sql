#  B2B Customer RFM Segmentation & Analytics

##  Executive Summary
This project performs an **RFM (Recency, Frequency, Monetary)** customer segmentation on over **$10.03M in global sales data**. Using advanced SQL techniques like **Window Functions (`NTILE`) and CTEs**, 92 enterprise accounts were categorized into strategic tiers (*Champions, Loyal, At Risk, Lost*) to drive targeted retention strategies.

---

##  Database Schema & Setup

```sql
-- Create table structure for global sales transactions
CREATE TABLE sales_data (
    ORDERNUMBER INT,
    QUANTITYORDERED INT,
    PRICEEACH DECIMAL(10, 2),
    ORDERLINENUMBER INT,
    SALES DECIMAL(10, 2),
    ORDERDATE DATETIME,
    STATUS VARCHAR(50),
    QTR_ID INT,
    MONTH_ID INT,
    YEAR_ID INT,
    PRODUCTLINE VARCHAR(100),
    MSRP DECIMAL(10, 2),
    PRODUCTCODE VARCHAR(50),
    CUSTOMERNAME VARCHAR(255),
    PHONE VARCHAR(50),
    ADDRESSLINE1 VARCHAR(255),
    ADDRESSLINE2 VARCHAR(255),
    CITY VARCHAR(100),
    STATE VARCHAR(100),
    POSTALCODE VARCHAR(50),
    COUNTRY VARCHAR(100),
    TERRITORY VARCHAR(50),
    CONTACTLASTNAME VARCHAR(100),
    CONTACTFIRSTNAME VARCHAR(100),
    DEALSIZE VARCHAR(50)
);
```

---

##  Core RFM Segmentation SQL Query

```sql
-- =====================================================================
-- PROJECT: RFM Customer Segmentation Analysis
-- DATASET: Global B2B Sales Transactions ($10.03M Revenue)
-- AUTHOR: Data Analytics Portfolio
-- =====================================================================

WITH Reference_Date AS (
    -- Set reference anchor date slightly past the latest transaction date (2005-12-01)
    SELECT CAST('2005-12-02' AS DATE) AS Max_Date
),

-- 1. Calculate Raw RFM Metrics per Customer
Customer_Raw_RFM AS (
    SELECT 
        s.CUSTOMERNAME,
        s.COUNTRY,
        s.TERRITORY,
        -- Recency: Days elapsed since customer's last order
        DATEDIFF(day, MAX(CAST(s.ORDERDATE AS DATE)), (SELECT Max_Date FROM Reference_Date)) AS Recency_Days,
        -- Frequency: Count of distinct order numbers
        COUNT(DISTINCT s.ORDERNUMBER) AS Frequency_Orders,
        -- Monetary: Total sales revenue generated from fulfilled orders
        SUM(s.SALES) AS Monetary_Value
    FROM sales_data s
    WHERE s.STATUS = 'Shipped' -- Focus on successfully fulfilled revenue
    GROUP BY s.CUSTOMERNAME, s.COUNTRY, s.TERRITORY
),

-- 2. Assign Percentile Scores (1 to 5) using Window Functions (NTILE)
RFM_Scoring AS (
    SELECT 
        CUSTOMERNAME,
        COUNTRY,
        TERRITORY,
        Recency_Days,
        Frequency_Orders,
        Monetary_Value,
        -- Recency: Lower days get higher score (5 = most recent)
        NTILE(5) OVER (ORDER BY Recency_Days DESC) AS R_Score,
        -- Frequency: Higher orders get higher score (1 = lowest, 5 = highest)
        NTILE(5) OVER (ORDER BY Frequency_Orders ASC) AS F_Score,
        -- Monetary: Higher spending gets higher score (1 = lowest, 5 = highest)
        NTILE(5) OVER (ORDER BY Monetary_Value ASC) AS M_Score
    FROM Customer_Raw_RFM
),

-- 3. Combine Scores and Map to Strategic Business Segments
RFM_Segmented AS (
    SELECT 
        CUSTOMERNAME,
        COUNTRY,
        TERRITORY,
        Recency_Days,
        Frequency_Orders,
        ROUND(Monetary_Value, 2) AS Monetary_Value,
        R_Score,
        F_Score,
        M_Score,
        CONCAT(CAST(R_Score AS VARCHAR), CAST(F_Score AS VARCHAR), CAST(M_Score AS VARCHAR)) AS RFM_Cell,
        CASE 
            WHEN R_Score >= 4 AND F_Score >= 4 THEN 'Champions'
            WHEN R_Score >= 3 AND F_Score >= 3 THEN 'Loyal Customers'
            WHEN R_Score >= 3 AND F_Score <= 2 THEN 'Promising / Recent'
            WHEN R_Score <= 2 AND F_Score >= 3 THEN 'At Risk (High Value)'
            ELSE 'Lost / Dormant'
        END AS Customer_Segment
    FROM RFM_Scoring
)

-- 4. Final Output: Detailed Customer Level RFM Analysis
SELECT 
    CUSTOMERNAME,
    COUNTRY,
    TERRITORY,
    Recency_Days,
    Frequency_Orders,
    Monetary_Value,
    R_Score,
    F_Score,
    M_Score,
    RFM_Cell,
    Customer_Segment
FROM RFM_Segmented
ORDER BY Monetary_Value DESC;
```
