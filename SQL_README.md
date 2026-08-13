# SQL Analysis — Personal Bank Statement (MySQL)

This folder contains the SQL side of the [Personal Bank Statement Analysis
project](../README.md) — the same dataset already analyzed in Excel, rebuilt
and cross-validated in MySQL to demonstrate query-based analysis, window
functions, and CTEs.

## Tools Used
- MySQL 8.0 / MySQL Workbench
- Data loaded via `LOAD DATA LOCAL INFILE` (356 transactions)

## Database Schema

```sql
CREATE TABLE transactions (
    txn_id          INT AUTO_INCREMENT PRIMARY KEY,
    txn_date        DATE NOT NULL,
    narration       VARCHAR(255),
    category        VARCHAR(50),
    ref_no          VARCHAR(50),
    withdrawal_amt  DECIMAL(10,2),
    deposit_amt     DECIMAL(10,2),
    amount          DECIMAL(10,2),
    closing_balance DECIMAL(10,2)
);
```
Two known self-transfer transactions (money moved between the account
holder's own accounts) were labeled `category = 'Self Transfer'` before
import, so they can be cleanly excluded from spend/savings calculations —
this was a finding carried over from the Excel analysis, not discovered here.

## Queries & Findings

### 1. Overall Income, Expense, and Net Savings
```sql
SELECT
    ROUND(SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END), 2) AS total_income,
    ROUND(SUM(CASE WHEN amount < 0 THEN -amount ELSE 0 END), 2) AS total_expense,
    ROUND(SUM(amount), 2) AS net_savings
FROM transactions
WHERE category != 'Self Transfer';
```
| total_income | total_expense | net_savings |
|---|---|---|
| ₹92,726.71 | ₹52,277.56 | ₹40,449.15 |

**Finding:** Over the 3-month period, net savings after excluding self-transfers
was ₹40,449.15 — result matches the independently-built Excel workbook exactly,
confirming the dataset and logic are consistent across tools.

---

### 2. Monthly Income vs. Expense Trend
```sql
SELECT
    DATE_FORMAT(txn_date, '%Y-%m') AS month,
    ROUND(SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END), 2) AS total_income,
    ROUND(SUM(CASE WHEN amount < 0 THEN -amount ELSE 0 END), 2) AS total_expense,
    ROUND(SUM(amount), 2) AS net_savings
FROM transactions
WHERE category != 'Self Transfer'
GROUP BY DATE_FORMAT(txn_date, '%Y-%m')
ORDER BY month;
```
| month | total_income | total_expense | net_savings |
|---|---|---|---|
| 2026-05 | ₹25,755.00 | ₹16,524.31 | ₹9,230.69 |
| 2026-06 | ₹32,590.71 | ₹21,290.10 | ₹11,300.61 |
| 2026-07 | ₹34,381.00 | ₹14,463.15 | ₹19,917.85 |

**Finding:** Savings grew every month. July was the strongest month by a wide
margin. June's income was boosted by a one-time allowance (~₹6–7k above
baseline), which is why raw savings alone slightly overstates June's actual
spending discipline — see the Excel report for the income-adjusted savings
rate that corrects for this.

---

### 3. Spend by Category (% of total)
```sql
SELECT
    category,
    ROUND(SUM(-amount), 2) AS total_spent,
    COUNT(*) AS txn_count,
    ROUND(100.0 * SUM(-amount) / SUM(SUM(-amount)) OVER (), 1) AS pct_of_total
FROM transactions
WHERE amount < 0 AND category != 'Self Transfer'
GROUP BY category
ORDER BY total_spent DESC;
```
| category | total_spent | txn_count | pct_of_total |
|---|---|---|---|
| Rent | ₹21,000.00 | 3 | 40.2% |
| Person-to-Person Transfer | ₹17,006.00 | 115 | 32.5% |
| Transport | ₹4,584.75 | 125 | 8.8% |
| Food & Dining | ₹3,939.00 | 37 | 7.5% |
| Groceries & Local Store | ₹2,086.00 | 29 | 4.0% |
| Recharge & Bills | ₹1,970.81 | 10 | 3.8% |
| Shopping | ₹876.00 | 2 | 1.7% |
| Business / Vendor Payment | ₹680.00 | 11 | 1.3% |
| Medical | ₹135.00 | 2 | 0.3% |

**Finding:** Rent is the single largest cost (40.2%) despite only 3
transactions — a fixed, non-negotiable expense. Transport is the most
*frequent* category (125 transactions) but only 8.8% of value, showing that
transaction count and transaction value tell very different stories and
shouldn't be confused with each other.

---

### 4. Anomaly Detection (transactions > 3x average spend)
```sql
WITH stats AS (
    SELECT AVG(-amount) AS avg_spend
    FROM transactions
    WHERE amount < 0 AND category != 'Self Transfer'
)
SELECT t.txn_date, t.narration, t.category, -t.amount AS spend_amount,
       ROUND(s.avg_spend, 2) AS overall_avg_spend
FROM transactions t, stats s
WHERE t.amount < 0 AND t.category != 'Self Transfer'
  AND -t.amount > s.avg_spend * 3
ORDER BY spend_amount DESC;
```
| txn_date | category | spend_amount | avg_spend |
|---|---|---|---|
| 2026-05-08 | Rent | ₹7,000.00 | ₹156.52 |
| 2026-06-04 | Rent | ₹7,000.00 | ₹156.52 |
| 2026-07-06 | Rent | ₹7,000.00 | ₹156.52 |
| 2026-06-05 | Person-to-Person Transfer | ₹5,000.00 | ₹156.52 |
| 2026-05-24 | Food & Dining | ₹2,202.00 | ₹156.52 |
| 2026-06-27 | Person-to-Person Transfer | ₹1,450.00 | ₹156.52 |
| 2026-07-15 | Transport | ₹780.00 | ₹156.52 |
| 2026-05-31 | Person-to-Person Transfer | ₹700.00 | ₹156.52 |
| 2026-05-18 | Person-to-Person Transfer | ₹500.00 | ₹156.52 |

**Finding:** The average transaction is only ₹156.52, so anything over ~₹470
is flagged. The 3 Rent payments are expected (fixed monthly cost). More
interesting: a single ₹2,202 food order at "MEGHANA FOODS" is over 14x the
average transaction — a genuine one-off outlier worth remembering, and one
that wasn't obvious from the Excel Pivot Table review alone, since it wasn't
large enough to distort a monthly total the way the self-transfers did.

---

### 5. Recurring Counterparties (3+ transactions)
```sql
SELECT
    SUBSTRING_INDEX(SUBSTRING_INDEX(narration, '-', 2), '-', -1) AS counterparty,
    category, COUNT(*) AS occurrences,
    ROUND(AVG(-amount), 2) AS avg_amount,
    ROUND(SUM(-amount), 2) AS total_amount
FROM transactions
WHERE amount < 0 AND category != 'Self Transfer'
GROUP BY counterparty, category
HAVING COUNT(*) >= 3
ORDER BY occurrences DESC;
```
| counterparty | category | occurrences | avg_amount | total_amount |
|---|---|---|---|---|
| BMTC | Transport | 58 | ₹16.69 | ₹968.00 |
| IRA CAFE | Food & Dining | 23 | ₹34.13 | ₹785.00 |
| BANGALORE METRO | Transport | 13 | ₹80.00 | ₹1,040.00 |
| INDIAN RAILWAYS UTS | Transport | 12 | ₹29.50 | ₹354.05 |
| SRI RANGA STORE | Groceries & Local Store | 12 | ₹24.17 | ₹290.00 |
| JEEVAN S R | Person-to-Person Transfer | 4 | ₹1,420.00 | ₹5,680.00 |

*(full list of 26 recurring counterparties in the query output — showing
top 6 here for brevity)*

**Finding:** BMTC (public bus) is by far the most frequent transaction — 58
times, once every ~1.8 days on average, totaling only ₹968. This is a
predictable, budgetable "routine spend" baseline. More notably, JEEVAN S R
appears only 4 times but at a much higher average (₹1,420/transaction,
₹5,680 total) — a recurring high-value relationship that's easy to miss when
scanning by frequency alone, and only surfaces by also looking at average
transaction size per counterparty.

## Key Takeaways
- **All SQL results were cross-validated against the Excel analysis** and
  matched exactly — confirming data integrity across both tools.
- SQL's systematic anomaly threshold (3x average) caught a real outlier
  (₹2,202 Food & Dining transaction) that a manual Excel Pivot Table review
  did not surface, since it wasn't large enough to move a monthly total.
- Grouping by counterparty (not just category) revealed that **frequency and
  value are independent signals** — BMTC is high-frequency/low-value, while
  JEEVAN S R is low-frequency/high-value — a distinction only visible once
  data is grouped both ways.

## Files
- [`mysql_analysis_queries.sql`](mysql_analysis_queries.sql) — all 5 queries
  with inline comments explaining each SQL pattern used (conditional
  aggregation, window functions, CTEs, string extraction).
