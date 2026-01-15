
# Banking Database Performance Optimization (MySQL)

This project simulates a **production-scale banking database** and documents the process of optimizing a slow analytical query using real-world database techniques.

The goal was not just to make a query faster, but to understand *why* certain approaches fail at scale and what actually works in production systems.

---

## Dataset Overview

| Tables | Row Count |
|-----|----------|
| customers | ~49,000 |
| accounts | ~80,000 |
| transactions | ~6,000,000 |
| branches | 10 |

---

## Problem Statement

Retrieve the **top 50 customers by total account balance**, along with their **most recent transaction date within the last 30 days**.

This is a common reporting requirement in financial systems.

---

## Query Evolution & Results

### ❌ Version 1: Raw Joins on Transactions
- Joined `customers`, `accounts`, and `transactions` directly
- Filtered transactions by date
- Aggregated at customer level

```sql
SELECT 
c.customer_id,
SUM(a.balance) AS total_balance,
MAX(t.transaction_date) AS last_transaction_date
FROM customers c
JOIN accounts a
ON c.customer_id = a.customer_id
JOIN transactions t
ON a.account_id = t.account_id
WHERE balance > 90000
AND t.transaction_date >= NOW() - INTERVAL 30 day
GROUP BY c.customer_id
ORDER BY total_balance DESC
LIMIT 50;
```

![Screenshot 2026-01-15 172732](https://github.com/user-attachments/assets/1e48f2af-a8a6-4e28-94ac-49fe8b879073)

**Result:**  
⏱ ~4 minutes runtime 
❌ Massive nested loop fan-out  
❌ Late filtering of large transaction volumes

---

### ⚠️ Version 2: CTE with Pre-Aggregation
- Filtered recent transactions
- Aggregated per account before joining

```sql
WITH recent_txns AS (
    SELECT
        account_id,
        transaction_date
    FROM transactions
    WHERE transaction_date >= CURRENT_DATE - INTERVAL 30 DAY
)
SELECT
    c.customer_id,
    SUM(a.balance) AS total_balance,
    MAX(rt.transaction_date) AS last_transaction_date
FROM recent_txns rt
JOIN accounts a 
    ON a.account_id = rt.account_id
JOIN customers c 
    ON c.customer_id = a.customer_id
WHERE a.balance > 90000
GROUP BY
    c.customer_id
ORDER BY
    total_balance DESC
LIMIT 50;
```

![Screenshot 2026-01-15 172412](https://github.com/user-attachments/assets/4a6278b0-15a6-428a-8621-dacfbaaeb16e)


**Result:**  
⏱ ~10 seconds runtime  
✅ Significant improvement  
⚠️ Still scanning millions of rows

---

### ✅ Version 3: Summary Table Approach
- Introduced `account_summary` table
- Query ran against pre-aggregated data

```sql
SELECT
    customer_id,
    SUM(balance) AS total_balance,
    MAX(last_transaction_date) AS last_transaction_date
FROM account_summary
WHERE balance > 90000
GROUP BY customer_id
ORDER BY total_balance DESC
LIMIT 50;
```

 ![Screenshot 2026-01-15 173019](https://github.com/user-attachments/assets/87f8d831-c179-4fd9-b87b-017d53d871a1)

**Result:** 
⏱ ~195 milliseconds  
✅ Production-grade performance  
✅ Minimal I/O and CPU usage

---

## Key Lessons Learned

- Indexes alone do not solve performance problems at scale
- Query shape matters more than join order
- Pre-aggregation dramatically reduces workload
- Analytical queries should not scan raw transactional data in production

---

## Tools & Technologies

- MySQL
- EXPLAIN ANALYZE
- Index design & query refactoring
