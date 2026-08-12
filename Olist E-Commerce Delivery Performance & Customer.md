# Olist E-Commerce Delivery Performance & Customer Satisfaction Analysis

## 

### ASK

Does late delivery predict lower customer satisfaction (review scores) on the Olist marketplace — and which sellers or regions show the worst delivery performance?

---

### PREPARE

**Data Source**

This project uses the Brazilian E-Commerce Public Dataset by Olist, sourced from Kaggle. It's real, anonymized commercial data — approximately 100,000 orders placed on the Olist marketplace between 2016 and 2018, across multiple Brazilian states. Company and seller names have been replaced with Game of Thrones house names as part of the anonymization process.

I chose this dataset because it's genuinely relational — nine separate linked tables (orders, customers, order items, products, sellers, payments, reviews, geolocation, and a product category translation file) .

**Tables relevant to this analysis**

| Table | Relevant columns | Purpose |
| --- | --- | --- |
| `orders` | order_id, customer_id, order_status, order_purchase_timestamp, order_delivered_customer_date, order_estimated_delivery_date | Core order and delivery timing data |
| `order_items` | order_id, seller_id | Links orders to the sellers responsible for fulfillment |
| `order_reviews` | order_id, review_score | Customer satisfaction rating per order |
| `customers` | customer_id, customer_state | Customer location |
| `sellers` | seller_id, seller_state | Seller location |

**Initial Structure Notes**

- ~99,441 orders, one row per order in the `orders` table
- `customer_id` is unique per order; `customer_unique_id` (a separate field) identifies the same person across multiple orders — I'll use `customer_id` since this analysis is order-level, not customer-lifetime-level
- The dataset's own documentation lists "Delivery Performance" as an intended use case for this data

**Early Observations (flagging for Process phase — not yet verified)**

- Based on the dataset's documentation, `order_status` has 8 possible values, not just "delivered" — I expect cancelled/unavailable orders to be missing delivery dates as a result.
- While researching how others have approached this dataset, I noticed a mention of duplicate rows in the Geolocation table — flagging to verify myself rather than taking it as given, since I'm not using this table in my core analysis.
- I want to look at the *size* of a delivery delay, not just whether an order was late or on time — a 1-day delay and a 3-week delay likely affect satisfaction very differently.

---

### PROCESS

#### Verifying the order_status assumption

**Question:** Do missing delivery dates actually line up with non-delivered order statuses, the way I assumed in Prepare?

Built a pivot table in Google Sheets: `order_status` as rows, count of `order_id` and count of `order_delivered_customer_date` as values.

| order_status | order count | orders with a delivered date |
| --- | --- | --- |
| approved | 2 | 0 |
| canceled | 625 | 6 |
| created | 5 | 0 |
| delivered | 96,478 | 96,470 |
| invoiced | 314 | 0 |
| processing | 301 | 0 |
| shipped | 1,107 | 0 |
| unavailable | 609 | 0 |

**Reading it:** 96,478 out of 99,441 orders (97%) are `delivered`. The remaining 2,963 are split across `canceled` (625), `shipped` (1,107), `unavailable` (609), `invoiced` (314), `processing` (301), `created` (5), `approved` (2).

**Findings:**

- Confirms the assumption for most statuses — `shipped`, `unavailable`, `invoiced`, `processing`, `created`, `approved` all correctly show zero delivered dates.
- **Anomaly 1:** 8 orders marked `delivered` have no delivered date recorded.
- **Anomaly 2:** 6 orders marked `canceled` do have a delivered date — possibly cancelled after delivery (a return/refund) rather than a data error.

**Decision:** flagging both anomalies for follow-up once in pandas, given the very small scale (8 and 6 rows out of 99,441).

#### Checking for null/missing values across all tables

```python
orders.isnull().sum()
order_items.isnull().sum()
reviews.isnull().sum()
customers.isnull().sum()
sellers.isnull().sum()
```

**Result:**

| Table | Column | Nulls |
| --- | --- | --- |
| orders | order_approved_at | 160 |
| orders | order_delivered_carrier_date | 1,783 |
| orders | order_delivered_customer_date | 2,965 |
| orders | (all other columns) | 0 |
| order_items | (all columns) | 0 |
| reviews | review_comment_title | 87,656 |
| reviews | review_comment_message | 58,247 |
| reviews | (all other columns, including review_score) | 0 |
| customers | (all columns) | 0 |
| sellers | (all columns) | 0 |

**Findings:**

- Missing delivery dates in `orders` roughly align with the non-delivered `order_status` values already found in the Sheets pivot table.
- `review_score` itself has zero nulls — every review row has a real rating, even when the written comment is missing. Most reviewers rate without writing text (87,656 of ~99K reviews have no title, 58,247 have no message) — a real finding worth mentioning later (review collection/incentive opportunity).

**Side quest, flagged for later:** compare total `orders` (with `delivered` status) against total rows in `reviews` — some delivered orders may have no review at all, which would matter for whether the review-score analysis is representative or biased toward whoever chose to respond.

**First attempt at the orders-vs-reviews check** (paused, needs a fix before trusting it):

```python
delivered_orders = orders[orders['order_status'] == 'delivered']
merged_check = delivered_orders.merge(reviews, on='order_id', how='left', indicator=True)
merged_check['_merge'].value_counts()
```

**Result:** `both`: 96,361 | `left_only`: 646 | `right_only`: 0 — totaling 97,007, which does *not* match the 96,478 delivered orders found in the Sheets pivot table (a difference of 529). Paused here to build the full cleaning checklist first, rather than chasing this one anomaly in isolation. (Later explained by duplicate reviews — see below.)

#### Cleaning Checklist (questions to answer before trusting any analysis)

**orders**

1. Are the 5 date columns stored as dates, or as text?
2. Does `order_id` contain any duplicates?
3. Does `order_delivered_customer_date` ever occur before `order_purchase_timestamp`?
4. Does `order_delivered_customer_date` ever occur before `order_delivered_carrier_date`?
5. Do the 8 "delivered but no date" and 6 "canceled but has a date" anomalies found in Sheets hold up in pandas?

**order_items**

1. Are there duplicate rows (same `order_id` + `order_item_id`)?
2. Are any `price` or `freight_value` values zero or negative?

**reviews**

1. Does `order_id` contain duplicates — multiple reviews for the same order?
2. Does `review_score` only contain valid values (1 through 5)?
3. Side quest: how many delivered orders have no review at all, and does that gap bias the analysis?

**customers**

1. Does `customer_id` contain duplicates?
2. Does `customer_state` only contain valid Brazilian state abbreviations?

**sellers**

1. Does `seller_id` contain duplicates?
2. Does `seller_state` only contain valid Brazilian state abbreviations?

**Cross-table**

1. Once each table is individually clean, do row counts behave as expected at each merge step, with nothing silently duplicated or lost?

#### Converting date columns and running the checklist

```python
date_columns = ['order_purchase_timestamp', 'order_approved_at', 'order_delivered_carrier_date',
                'order_delivered_customer_date', 'order_estimated_delivery_date']

for col in date_columns:
    orders[col] = pd.to_datetime(orders[col])
orders.dtypes
```

```python
orders['order_id'].duplicated().sum()
```

**Result:** 0 — no duplicate order IDs.

```python
(orders['order_delivered_customer_date'] < orders['order_purchase_timestamp']).sum()
```

**Result:** 0 — no order shows a delivery date before its purchase date.

```python
(orders['order_delivered_customer_date'] < orders['order_delivered_carrier_date']).sum()
weird_rows = orders[orders['order_delivered_customer_date'] < orders['order_delivered_carrier_date']]
weird_rows[['order_delivered_customer_date', 'order_estimated_delivery_date']]
```

**Result:** 23 orders show a customer delivery date earlier than the carrier delivery date — meaning the order appears to have reached the customer before it left the seller's hands. A data quality issue, at a very small scale (23 of 99,441 = 0.023%).

**Decision:** excluding these 23 rows, consistent with the treatment of the earlier 8+6 anomalies. Given the scale (0.023% of the data), this is a documented judgment call rather than a full re-verification — expected to have negligible impact on the headline findings below, in the same way the original 14-row exclusion did.

```python
order_items.duplicated(subset=['order_id', 'order_item_id']).sum()
order_items[(order_items['price'] <= 0) | (order_items['freight_value'] < 0)]
```

#### Excluding the anomalous rows

```python
rows_to_exclude = (
    (orders['order_status'] == 'delivered') & (orders['order_delivered_customer_date'].isnull())
) | (
    (orders['order_status'] == 'canceled') & (orders['order_delivered_customer_date'].notnull())
) | (
    orders['order_delivered_customer_date'] < orders['order_delivered_carrier_date']
)

orders_clean = orders[~rows_to_exclude]
print(len(orders))
print(len(orders_clean))
print(rows_to_exclude.sum())
```

**Result:** 37 rows excluded (8 delivered-with-no-date + 6 canceled-with-date + 23 carrier-date anomalies). `orders_clean` holds the filtered table; the original `orders` remains untouched for reference.

**Decision:** proceeding with `orders_clean` for all further analysis. The excluded rows represent 0.037% of the data — negligible in scale, but documented individually with reasoning rather than silently dropped.

#### Investigating duplicate reviews

```python
reviews['order_id'].duplicated().sum()
```

**Result:** 551 orders have more than one review attached — explains the earlier orders-vs-reviews merge discrepancy.

```python
duplicate_order_ids = reviews[reviews['order_id'].duplicated(keep=False)]
duplicate_order_ids.sort_values('order_id').head(20)
order_items[order_items['order_id'] == '0035246a40f520710769010f752e7507']
```

**Investigation:** looked at sample duplicate pairs — review_ids always differ (not a system glitch re-logging the same review), dates are usually a few days apart, and scores can differ between the two reviews for the same order. Checked whether this was explained by multiple sellers per order (it wasn't — sample order had a single seller/item). Most plausible explanation: a follow-up/reminder survey resulting in some customers submitting twice — a hypothesis, not confirmed.

```python
duplicate_ids = reviews[reviews['order_id'].duplicated(keep=False)]['order_id'].unique()
sample_check = orders_clean[orders_clean['order_id'].isin(duplicate_ids)]
(sample_check['order_delivered_customer_date'] > sample_check['order_estimated_delivery_date']).mean()
(orders_clean['order_delivered_customer_date'] > orders_clean['order_estimated_delivery_date']).mean()
```

**Bias check:** compared the late-delivery rate among duplicate-review orders (7.13%) against the overall cleaned dataset (7.87%) — no meaningful difference, so keeping the most recent review per order shouldn't introduce bias into the delay-vs-satisfaction analysis.

**Decision:** for orders with multiple reviews, keep only the most recent one (by `review_answer_timestamp`).

```python
reviews_clean = reviews.sort_values('review_answer_timestamp', ascending=False).drop_duplicates(subset='order_id', keep='first')
len(reviews) - len(reviews_clean)
```

**Result:** 551 rows removed, matching the duplicate count exactly.

```python
reviews['review_score'].unique()
```

**Result:** `[1, 2, 3, 4, 5]` — only valid values.

#### customers and sellers

```python
customers['customer_id'].duplicated().sum()
sellers['seller_id'].duplicated().sum()
```

**Result:** 0 and 0 — no duplicate IDs in either table.

```python
valid_states = ['AC', 'AL', 'AP', 'AM', 'BA', 'CE', 'DF', 'ES', 'GO', 'MA', 'MT', 'MS',
                 'MG', 'PA', 'PB', 'PR', 'PE', 'PI', 'RJ', 'RN', 'RS', 'RO', 'RR', 'SC',
                 'SP', 'SE', 'TO']

customers[~customers['customer_state'].isin(valid_states)]
sellers[~sellers['seller_state'].isin(valid_states)]
```

**Result:** both empty — every state value in both tables is valid. Both tables clean as-is, no exclusions needed.

#### Merge chain

```python
step1 = orders_clean.merge(order_items, on='order_id', how='inner')
print(len(orders_clean), len(step1))
```

**Result:** growth expected — some orders have multiple items.

```python
step2 = step1.merge(reviews_clean, on='order_id', how='left')
print(len(step1), len(step2))
```

**Result:** no growth — one review per order.

```python
step3 = step2.merge(customers, on='customer_id', how='left')
print(len(step2), len(step3))
```

**Result:** no growth.

```python
step4 = step3.merge(sellers, on='seller_id', how='left')
print(len(step3), len(step4))
```

**Result:** no growth. Final merged table is item-level (an order with multiple items appears as multiple rows). Every merge verified against expected row-count behavior before trusting the result.

#### Reflection: join type choice for Step 1

Used `how='inner'` for the orders → order_items merge, reasoning every order should have at least one item. Verified with row counts before/after (growth only, no shrinkage). **In hindsight, `left` would have been the more consistent, defensible default** —  moving to a more restrictive join only after proving nothing would be lost. With `inner`, an unmatched order is silently dropped with no visible trace; with `left`, it shows up as a row with blank item columns — a visible signal to investigate. The verification step caught what needed catching this time, but `left` first, `inner` only once proven safe, is the more rigorous habit going forward.

---

### ANALYZE

#### Building order_level and the 775-order gap

```python
order_level = step4.drop_duplicates(subset='order_id').copy()
print(len(order_level))
```

```python
order_level['delay_days'] = (order_level['order_delivered_customer_date'] - order_level['order_estimated_delivery_date']).dt.days
order_level['delay_days'].describe()
```

```python
orders_clean['order_id'].nunique()
order_level['order_id'].nunique()
missing_orders = orders_clean[~orders_clean['order_id'].isin(order_level['order_id'])]
len(missing_orders)
missing_orders['order_status'].value_counts()
```

**Finding:** `orders_clean` had significantly more unique orders than `order_level` — a gap of orders that never received an item record at all. All of the missing orders fall into non-delivered categories (unavailable, canceled, created, invoiced, shipped). Zero were `delivered`.

**Root cause:** the Step 1 `inner` join assumed every order has at least one row in `order_items`. That assumption was wrong for orders cancelled or marked unavailable early in their lifecycle — they never received an item record, so the inner join silently dropped them, exactly the risk flagged in the join-type reflection above.

#### Data integrity fix — non-delivered orders leaking into the delay analysis

**Problem found:** running the two delay calculations below gave two different answers for what should have been the same "on-time" population:

```python
order_level['was_late'] = order_level['delay_days'] > 0
order_level.groupby('was_late')['review_score'].mean()
```

```python
def bucket_delay(days):
    if days <= 0:
        return 'On-time or early'
    elif days <= 3:
        return '1-3 days late'
    elif days <= 7:
        return '4-7 days late'
    elif days <= 14:
        return '8-14 days late'
    else:
        return '15+ days late'

order_level['delay_bucket'] = order_level['delay_days'].apply(bucket_delay)
```

Before the fix: `was_late` grouping gave on-time = 4.23, but `delay_bucket` grouping gave "On-time or early" = 4.29 — the same `delay_days ≤ 0` population, two different averages.

**Root cause:** `order_level` was never filtered to `order_status == 'delivered'`. The item-record gap above only accounts for non-delivered orders with *no* item record; it missed non-delivered orders that *did* have an item record and survived into `order_level` with a **null** `delay_days` (no delivery date to compute a delay from). `NaN > 0` evaluates `False` in pandas, so those rows silently fell into the `was_late == False` ("on-time") group in the first calculation. But `NaN <= 0` is *also* `False`, so in the `bucket_delay()` function they fell through every `elif` into the final `else`: **15+ days late**. Same contaminated rows, opposite buckets in each calculation — hence the mismatch.

**Fix applied:**

```python
order_level = order_level[order_level['order_status'] == 'delivered'].copy()
```

placed immediately after `order_level = step4.drop_duplicates(subset='order_id').copy()`, before computing `delay_days`.

**Verification — re-ran both calculations after the fix:**

```
was_late  False    4.290386
          True     2.270491
```

```
On-time or early    4.290386
1-3 days late        3.291037
4-7 days late        2.102975
8-14 days late       1.670816
15+ days late        1.723596
```

The two on-time figures now match exactly (4.29), confirming the fix. Notably, the 15+ days late average barely moved from its pre-fix value (1.74 → 1.72) — suggesting the "floor effect" interpretation below mostly holds and wasn't purely a data-contamination artifact.

*Note: the 23-carrier-date-anomaly exclusion (Process phase) was decided after this fix was verified; given its scale (0.023% of the data), it was not re-run through the full pipeline — documented as a judgment call rather than re-verified end to end.*

#### Delay vs. review score — the core finding

**Result:**

- On-time or early orders: **4.29** average review score
- Late orders: **2.27** average review score

**Finding:** late delivery is strongly associated with lower customer satisfaction — a 2-point drop on a 5-point scale between on-time and late orders. This directly answers the first half of the business question.

#### Delay severity vs. review score

| Delay bucket | Order count | Avg review score |
| --- | --- | --- |
| On-time or early | 89,936 | 4.29 |
| 1-3 days late | 1,870 | 3.29 |
| 4-7 days late | 1,802 | 2.10 |
| 8-14 days late | 1,478 | 1.67 |
| 15+ days late | 1,384 | 1.72 |

**Finding:** review score declines sharply and consistently as delay severity increases, roughly a full point per bucket, leveling off around 8-14 days. The slight uptick at 15+ days is consistent with a "floor effect": once a customer is already maximally dissatisfied, further delay doesn't push the rating any lower, since 1 star is the worst available option. This held up after removing the non-delivered-order contamination that inflated this bucket's count in the earlier (buggy) version — the corrected count (1,384) is well under half the original uncorrected figure, confirming the earlier bucket was significantly overstated by data leakage rather than genuine 15+-day delays.

#### Delivery performance by state

```python
order_level.groupby('customer_state')[['delay_days', 'review_score']].mean().sort_values('delay_days', ascending=False)
```

**Finding:** every state showed a negative average delay (all deliveries early on average), and review score didn't track cleanly against average delay across states. This makes sense in light of the bucket finding above: it's not average delay that drives dissatisfaction, it's the proportion of orders landing in the worst delay buckets — an average can look fine while hiding a meaningful share of genuinely late, badly-reviewed orders.

**Revised approach — % of orders late, with sample size for reliability:**

```python
state_summary = order_level.groupby('customer_state').agg(
    order_count=('order_id', 'count'),
    pct_late=('was_late', 'mean'),
    avg_review=('review_score', 'mean')
).sort_values('pct_late', ascending=False)

state_summary
```

**Result (worst 5 by % late):**

| State | Orders | % Late | Avg Review |
| --- | --- | --- | --- |
| AL (Alagoas) | 397 | 21.4% | 3.85 |
| MA (Maranhão) | 717 | 17.4% | 3.83 |
| SE (Sergipe) | 335 | 15.2% | 3.91 |
| PI (Piauí) | 476 | 13.9% | 3.99 |
| CE (Ceará) | 1,279 | 13.8% | 3.94 |

**Best 5 by % late:**

| State | Orders | % Late | Avg Review |
| --- | --- | --- | --- |
| AM (Amazonas) | 145 | 2.8% | 4.24 |
| RO (Rondônia) | 243 | 2.9% | 4.17 |
| AP (Amapá) | 67 | 3.0% | 4.24 |
| AC (Acre) | 80 | 3.8% | 4.09 |
| PR (Paraná) | 4,923 | 4.0% | 4.24 |

**Finding:** states with a higher proportion of late orders generally show lower average review scores, and vice versa — confirming the delay-satisfaction relationship holds at the state level too, not just order-by-order. Worth noting the lowest-late-rate states (AM, RO, AP, AC) all have relatively small order counts (67-243), so those specific rankings are less statistically reliable than higher-volume states like SP (40,494 orders, 4.5% late) or MG (11,354 orders, 4.6% late), which show the same pattern with much stronger sample support.

#### Delivery performance by seller

```python
seller_summary = order_level.groupby('seller_id').agg(
    order_count=('order_id', 'count'),
    pct_late=('was_late', 'mean'),
    avg_review=('review_score', 'mean')
).sort_values('pct_late', ascending=False)
```

**Initial look:** sorting `pct_late` directly across all sellers was dominated by sellers with only 1-2 orders, producing meaningless 100%/0% extremes.

**Filtered to a reliable minimum:**

```python
reliable_sellers = seller_summary[seller_summary['order_count'] >= 50]
reliable_sellers.sort_values('pct_late', ascending=False).head(15)
```

**Result (worst 5):**

| Seller ID (truncated) | Orders | % Late | Avg Review |
| --- | --- | --- | --- |
| 54965bbe... | 72 | 30.6% | 3.18 |
| beadbee3... | 63 | 23.8% | 3.94 |
| a49928bc... | 94 | 22.3% | 3.06 |
| 712e6ed8... | 77 | 20.8% | 3.39 |
| 06a2c3af7... | 388 | 19.1% | 4.02 |

**Finding:** the delay-satisfaction relationship holds clearly at the seller level too. Most high-late-rate sellers show correspondingly low review scores. One notable exception: seller `06a2c3af7...` (388 orders, 19.1% late) maintains a relatively strong 4.02 average review despite a high late rate and the largest order count in this list — worth flagging as a case that doesn't fully fit the pattern, without overclaiming why.

---

### SHARE

**Visualizations**

Built three charts in Python (matplotlib/seaborn), staying consistent with a green-to-red color scale (green = good, red = bad) across all three for easy reading.

1. **Average Review Score by Delivery Delay Severity** — bar chart showing the core finding: satisfaction drops sharply and consistently as delay worsens (4.29 → 1.72), leveling off around 8-14 days late.
2. **Top 10 States by Late Delivery Rate** — highlights AL, MA, and SE as the states with the highest proportion of late deliveries.
3. **Top 10 Worst-Performing Sellers by Late Delivery Rate** (minimum 50 orders, to ensure statistical reliability) — identifies specific sellers with consistently poor delivery performance.

**Note:** seller IDs are long anonymized strings rather than real company names (part of the dataset's anonymization) — in a real business setting, this chart would use actual seller/account names for direct actionability.
