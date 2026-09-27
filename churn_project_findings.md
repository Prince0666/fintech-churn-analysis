# Fintech Customer Churn & Revenue Impact Analysis — Findings Log

**Dataset:** `fintech_customer_churn_data.csv` — 4,000 customers, fintech subscription/lending context (Personal Loan, Credit Card, BNPL, Insurance products)

**Tool used:** SQLite (SQL Online IDE)

---

## Q1. Overall churn rate

**Query:**
```sql
SELECT count(*), avg(churned) * 100 as churned_percentage
FROM fintech_customer_churn_data
```

**Result:** Total customers = 4000, Churn rate = **34.95%**

**Insight:** Roughly 1 in 3 customers has churned — a high enough rate to justify a dedicated retention analysis and prioritization by segment/product.

---

## Q2. Segment-wise churn rate

**Query:**
```sql
select count(*), segment, avg(churned) * 100 as churned_percentage
FROM fintech_customer_churn_data
where segment in ('Premium','Standard','Basic')
Group by segment
```

**Result:**
| Segment | Churn Rate |
|---|---|
| Basic | 53.46% |
| Standard | 27.04% |
| Premium | 9.32% |

**Insight:** Basic segment churns at ~5.7x the rate of Premium. Segment is the single strongest churn driver found so far — retention efforts should be prioritized on Basic-tier customers first.

---

## Q3. Product-wise churn rate

**Query:**
```sql
select count(*), product_type, avg(churned) * 100 as churned_percentage
FROM fintech_customer_churn_data
where product_type in ('Credit Card','BNPL','Personal Loan','Insurance')
Group by product_type
```

**Result:**
| Product | Churn Rate |
|---|---|
| BNPL | 36.90% |
| Credit Card | 34.94% |
| Personal Loan | 34.09% |
| Insurance | 32.99% |

**Insight:** Product type shows a much narrower spread (33-37%) compared to segment (9-53%). This means segment is the dominant churn driver, not product type — a key point for the recommendation section.

---

## Q4. City tier-wise churn rate

**Query:**
```sql
select count(*), city_tier, avg(churned) * 100 as churned_percentage
FROM fintech_customer_churn_data
where city_tier in ('Tier 1','Tier 2','Tier 3')
Group by city_tier
```

**Result:**
| City Tier | Churn Rate |
|---|---|
| Tier 1 | 32.97% |
| Tier 2 | 36.32% |
| Tier 3 | 36.15% |

**Insight:** No meaningful spread across city tiers — geography is not a churn driver in this dataset. Confirms segment remains the dominant lever found so far.

---

## Q5. Segments above overall average churn rate (HAVING)

**Query:**
```sql
select segment, avg(churned) * 100 as churned_percentage
FROM fintech_customer_churn_data
GROUP by segment
having Avg(churned)*100 > 34.95;
```

**Result:** Only **Basic** (53.46%) exceeds the overall average (34.95%). Standard (27.04%) and Premium (9.32%) are both below it.

**Insight:** Basic segment alone is pulling the overall churn rate up. Retention efforts should concentrate almost entirely on this segment.

---

## Q6. Late payments vs churn (bucketed)

**Query:**
```sql
select  
CASE
    WHEN late_payments_count = 0 THEN '0'
    WHEN late_payments_count BETWEEN 1 AND 2 THEN '1-2'
    ELSE '3+'
END as late_payment_bucket,
AVG(churned) * 100 as churned_percentage,
COUNT(*) as customer_count
FROM fintech_customer_churn_data
GROUP BY late_payment_bucket;
```

**Result:**
| Late Payments | Churn Rate |
|---|---|
| 0 | 27.58% |
| 1-2 | 44.04% |
| 3+ | 67.76% |

**Insight:** Strong, near-linear relationship — churn risk jumps ~2.5x from 0 to 3+ late payments. This is a clear early-warning signal: intervention should trigger at the first late payment, not wait until multiple.

---

## Q7. Support tickets vs churn (bucketed)

**Query:**
```sql
select  
CASE
    WHEN support_tickets = 0 THEN '0'
    WHEN support_tickets = 1 Then '1'
    ELSE '2+'
END as support_ticket_bucket,
AVG(churned) * 100 as churned_percentage,
COUNT(*) as customer_count
FROM fintech_customer_churn_data
GROUP BY support_ticket_bucket;
```

**Result:**
| Support Tickets (90d) | Churn Rate |
|---|---|
| 0 | 22.90% |
| 1 | 31.57% |
| 2+ | 50.11% |

**Insight:** Another strong, near-linear driver — churn risk more than doubles (2.2x) from 0 to 2+ tickets. Combined with late payments, this gives two clean early-warning signals for a risk-scoring flag.

---

## Q8. Total estimated revenue lost

**Query:**
```sql
SELECT sum(estimated_revenu) as total_revenue_lost
from fintech_customer_churn_data
```

**Result:** ₹1,07,30,484 total estimated revenue at risk (monthly-revenue-equivalent, across all churned customers)

**Insight:** This is the headline business-impact number for the project — frames the analysis in ₹ terms, not just a churn %.

---

## Q9. Revenue lost — segment-wise

**Query:**
```sql
SELECT segment, sum(estimated_revenu) as estimated_revenue_lost
from fintech_customer_churn_data
where segment in('Premium','Standard','Basic')
Group by segment
```

**Result:**
| Segment | Revenue Lost |
|---|---|
| Basic | ₹38,35,800 |
| Standard | ₹54,77,382 |
| Premium | ₹14,17,302 |

**Insight:** Churn rate and revenue impact don't rank the same way — Standard segment loses the most total revenue (₹54.7L) despite a lower churn rate than Basic, because retained/churned Standard customers carry more revenue each. Basic drives the highest churn %, but Standard drives the highest ₹ loss. Retention budget should be split across both, not concentrated only on the highest-churn segment.

---

## Q10. Revenue lost — product-wise

**Query:**
```sql
SELECT product_type, sum(estimated_revenu) as estimated_revenue_lost
from fintech_customer_churn_data
where product_type in('Credit Card','BNPL','Personal Loan','Insurance')
Group by product_type
```

**Result:**
| Product | Revenue Lost |
|---|---|
| Personal Loan | ₹42,94,152 |
| Credit Card | ₹38,89,314 |
| BNPL | ₹16,06,212 |
| Insurance | ₹9,40,806 |

**Insight:** Personal Loan and Credit Card drive the most revenue loss despite BNPL having the highest churn %. Same pattern as segment analysis — high churn % doesn't automatically mean high ₹ impact; per-customer revenue matters just as much.

---

## Q11. Engagement gap — days_since_last_active (churned vs non-churned)

**Query:**
```sql
SELECT churned, AVG(days_since_last) as avg_days_inactive
FROM fintech_customer_churn_data
GROUP BY churned
```

**Result:**
| Churned | Avg Days Since Last Active |
|---|---|
| No (0) | 16.36 |
| Yes (1) | 38.47 |

**Insight:** Churned customers were inactive ~2.35x longer before leaving. Inactivity is a leading indicator, not just a symptom — supports a proactive re-engagement trigger (e.g., automated nudge after 25+ days of inactivity).

---

## SQL Findings Summary — Key Drivers Ranked

1. **Segment** — strongest churn-rate driver (Basic 53.5% vs Premium 9.3%), but **not** the strongest revenue-impact driver
2. **Late payments** — strong, linear early-warning signal (27.6% → 67.8% as late payments go 0 → 3+)
3. **Support tickets** — strong, linear early-warning signal (22.9% → 50.1% as tickets go 0 → 2+)
4. **Days since last active** — churned customers show ~2.35x more inactivity before leaving (leading indicator)
5. **Revenue impact ≠ churn %** — Standard segment (₹54.7L) and Personal Loan/Credit Card products lose the most ₹, despite Basic/BNPL having the highest churn rates
6. **Product type & city tier** — weak/no signal on churn rate

---

## Final Business Recommendation
_(to be written next — should include: target segment/product, metric, timeline, owner, expected impact)_
