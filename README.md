# Personal-Bank-Statement-Analysis (Interactive Dashboard using MS Excel)

## Project Objective
Analyze 3 months (May–July 2026) of personal bank transactions to understand
spending patterns and calculate a true, income-adjusted savings rate.

## Dataset Used
- [Personal bank e-statement ](https://github.com/thanush912/Personal-Bank-Statement-Analysis/blob/main/Acct%20Statement.xlsx)

## Questions (KPIs)
- Income, expense, and net savings per month?
- Which category is the largest share of spending?
- Any transactions that distort the "true" spending picture?
- Actual savings rate (% of income saved) per month?
- Most frequent recurring merchants?
- Dashboard Interaction — [View Dashboard]((https://github.com/thanush912/Personal-Bank-Statement-Analysis/blob/main/index.html))

## Process
- Cleaned raw data: fixed date formats, removed footer rows, verified no
  missing values.
- Parsed UPI/NEFT narrations into Category using an `IF(SEARCH())` formula.
- Built a signed `Amount` column, cross-checked against `SUM(Deposits) −
  SUM(Withdrawals)`.
- Used Pivot Tables to summarize by month and category, and to drill into
  anomalies.
- Flagged and excluded two self-transfers that were inflating apparent spend.
- Calculated Total Income, Net Savings, and Savings Rate % per month with
  `SUMIFS`.
- Built a combo chart (income/savings + savings rate line) and a category
  pie chart, merged onto one Report sheet.

## Dashboard
<img width="1301" height="415" alt="Dashboard" src="https://github.com/user-attachments/assets/dc66c675-d51e-4be9-9f17-55aa8f5ca61a" />


## Project Insight
- June's raw expenses looked alarming (-₹25,435) but were mostly a ₹36,736
  self-transfer to another personal account, not real spending.
- Excluding self-transfers, all 3 months show positive, growing savings:
  May ₹9,230 → June ₹11,301 → July ₹19,918.
- Adjusted for a night-shift allowance, May and June had nearly the same
  savings rate (~36%) — June's higher raw number was just higher income.
- July is the strongest month by far (~58–61% savings rate).
- Rent is the largest expense category (43%), followed by Person-to-Person
  Transfers (19%) and Food & Dining (18%) — the two most flexible categories
  to cut back on.

## Final Conclusion
Raw totals can mislead when income and internal transfers fluctuate.
Savings rate (relative to income) — not raw savings — is the right metric
for comparing months. July 2026 was the strongest month of financial
discipline, and Person-to-Person Transfers and Food & Dining are the
categories with the most room to improve going forward.
