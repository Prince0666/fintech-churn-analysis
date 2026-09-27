# Fintech Customer Churn & Revenue Impact Analysis

**Author:** Prince Pal | [GitHub](https://github.com/Prince0666) | [LinkedIn](https://linkedin.com/in/prince-pal-311537311)

## Business Problem

A fintech company offering Personal Loans, Credit Cards, BNPL, and Insurance products is losing customers at a high rate. This project identifies **who churns, why they churn, and how much revenue is at risk** — then translates those findings into a targeted retention recommendation.

## Dataset

4,000 customers with signup date, demographics, segment (Premium/Standard/Basic), product type, credit score, revenue, engagement metrics, and churn status.

**Columns:** customer_id, signup_date, age, city_tier, segment, product_type, credit_score, monthly_revenue, avg_monthly_transactions, avg_transaction_amount, support_tickets_last_90d, late_payments_count, days_since_last_active, tenure_months, churned, churn_date, months_active_before_churn, estimated_revenue_lost

## Tools Used

SQL (SQLite) for exploratory querying · Python (Pandas, Matplotlib, Seaborn) for EDA and visualization

---

## Part 1 — SQL Analysis

Full query-by-query breakdown with results: see `churn_project_findings.md`.

**Key SQL findings:**

| # | Question | Finding |
|---|---|---|
| 1 | Overall churn rate | 34.95% |
| 2 | Churn by segment | Basic 53.46% · Standard 27.04% · Premium 9.32% |
| 3 | Churn by product | BNPL 36.9% · Credit Card 34.9% · Personal Loan 34.1% · Insurance 33.0% (narrow spread) |
| 4 | Churn by city tier | 33–36% across all tiers — no signal |
| 5 | Segments above avg churn (HAVING) | Only Basic exceeds the 34.95% average |
| 6 | Late payments vs churn | 0 → 27.6%, 1-2 → 44.0%, 3+ → 67.8% |
| 7 | Support tickets vs churn | 0 → 22.9%, 1 → 31.6%, 2+ → 50.1% |
| 8 | Total revenue lost | ₹1,07,30,484 |
| 9 | Revenue lost by segment | Standard ₹54.77L · Basic ₹38.36L · Premium ₹14.17L |
| 10 | Revenue lost by product | Personal Loan ₹42.94L · Credit Card ₹38.89L · BNPL ₹16.06L · Insurance ₹9.41L |
| 11 | Avg. inactivity, churned vs not | 38.47 days vs 16.36 days (2.35x) |

---

## Part 2 — Python EDA

### 2.1 Data Load & Inspection
```python
import pandas as pd

df = pd.read_csv('fintech_customer_churn_data.csv')
print(df.shape)
print(df.info())
print(df.head())
```

### 2.2 Churn Distribution
```python
import matplotlib.pyplot as plt

print(df['churned'].value_counts())

df['churned'].value_counts().plot(kind='bar')
plt.title('Churned vs Not Churned')
plt.xlabel('Churned (0=No, 1=Yes)')
plt.ylabel('Count')
plt.show()
```
![Churn Distribution](charts/churn_distribution.png)

~2,600 retained vs ~1,400 churned — confirms the 34.95% churn rate visually.

### 2.3 Churn Rate by Segment & Product
```python
import seaborn as sns

segment_churn = df.groupby('segment')['churned'].mean() * 100
print(segment_churn)
segment_churn.plot(kind='bar', color='steelblue')
plt.title('Churn Rate by Segment')
plt.ylabel('Churn Rate (%)')
plt.show()

product_churn = df.groupby('product_type')['churned'].mean() * 100
print(product_churn)
product_churn.plot(kind='bar', color='coral')
plt.title('Churn Rate by Product Type')
plt.ylabel('Churn Rate (%)')
plt.show()
```
![Churn by Segment](charts/churn_by_segment.png)
![Churn by Product](charts/churn_by_product.png)

Numbers match the SQL output exactly (Basic 53.5%, Premium 9.3%, Standard 27.0%) — a good cross-validation between the two tools. Product type shows a much flatter spread, confirming segment is the stronger categorical driver.

### 2.4 Numerical Drivers vs Churn (Boxplots)
```python
sns.boxplot(x='churned', y='late_payments_count', data=df)
plt.title('Late Payments vs Churn')
plt.show()

sns.boxplot(x='churned', y='support_tickets_last_90d', data=df)
plt.title('Support Tickets vs Churn')
plt.show()

sns.boxplot(x='churned', y='days_since_last_active', data=df)
plt.title('Days Since Last Active vs Churn')
plt.xlabel('Churned (0=No, 1=Yes)')
plt.show()
```
![Late Payments Boxplot](charts/late_payments_boxplot.png)
![Support Tickets Boxplot](charts/support_tickets_boxplot.png)
![Days Inactive Boxplot](charts/days_inactive_boxplot.png)

**Late payments:** medians look similar between churned/not-churned at the individual-customer level — the effect only becomes clearly visible once bucketed (as done in SQL Q6). This is a useful nuance: raw boxplots can understate a driver that's actually a strong step-function risk signal.

**Support tickets:** churned customers skew slightly higher, consistent with the SQL bucketed finding.

**Days since last active:** the clearest separation of all three — churned customers have a visibly higher median (~30 days) and wider spread than retained customers (~10 days), matching the 38.47 vs 16.36 day averages from SQL.

### 2.5 Correlation Heatmap
```python
numeric_cols = ['age', 'credit_score', 'monthly_revenue', 'avg_monthly_transactions', 
                 'avg_transaction_amount', 'support_tickets_last_90d', 'late_payments_count',
                 'days_since_last_active', 'tenure_months', 'churned']

corr = df[numeric_cols].corr()

plt.figure(figsize=(10, 8))
sns.heatmap(corr, annot=True, cmap='coolwarm', fmt='.2f')
plt.title('Correlation Heatmap')
plt.show()
```
![Correlation Heatmap](charts/correlation_heatmap.png)

**Correlation with churn (strongest to weakest):**
- `days_since_last_active`: **+0.39** (strongest driver)
- `support_tickets_last_90d`: +0.26
- `credit_score`: −0.28
- `monthly_revenue`: −0.28
- `late_payments_count`: +0.21
- `age`, `avg_monthly_transactions`, `avg_transaction_amount`, `tenure_months`: negligible (≤0.04)

Inactivity is the single strongest linear signal — stronger even than late payments or support tickets individually, even though all three combine into the clearest churn pattern when bucketed together.

---

## Key Insights (Combined SQL + Python)

1. **Segment is the dominant churn-rate driver** — Basic churns 5.7x more than Premium — but is **not** the dominant revenue-impact driver.
2. **Revenue impact ≠ churn rate.** Standard segment loses the most total revenue (₹54.7L) despite a mid-range churn rate, because retained Standard customers carry meaningful revenue per head. Basic drives volume of churn; Standard drives ₹ loss.
3. **Inactivity is the earliest and strongest warning sign** (correlation +0.39, 2.35x gap between churned/retained). Late payments and support tickets are strong but more step-function than linear — they matter most once a customer crosses a threshold (3+ late payments, 2+ tickets), not as a smooth gradient.
4. **Product type and city tier are not meaningful churn drivers** on their own.

## Business Recommendation

**Build a 3-signal risk flag — inactivity (25+ days), late payments (1+), and support tickets (2+) — and run a 90-day pilot retention campaign targeting two groups in parallel:**
- **Basic segment** customers (highest churn volume) — low-cost automated interventions (re-engagement nudges, fee waivers)
- **Standard-segment Personal Loan/Credit Card holders** (highest revenue-at-risk) — higher-touch retention (relationship manager outreach, personalized offers)

**Owner:** Retention/CRM team, in coordination with Risk (for the late-payment signal) and Customer Support (for the ticket signal)
**Success metric:** Reduce combined-cohort churn rate by 15-20% within the pilot window; track ₹ revenue retained against the ₹1.07 Cr baseline monthly loss
**Validation:** Re-run the SQL/EDA pipeline monthly to confirm the risk flag still holds as new cohorts churn

---

## Files in this project
- `fintech_customer_churn_data.csv` — dataset
- `churn_project_findings.md` — full SQL query-by-query log
- `churn_eda.ipynb` — Python EDA notebook
- `charts/` — exported chart images
