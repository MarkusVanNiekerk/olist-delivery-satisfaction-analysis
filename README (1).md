# Olist Delivery Performance & Customer Satisfaction Analysis

Does late delivery predict lower customer satisfaction on an e-commerce marketplace — and which sellers or regions perform worst? A multi-table analysis of ~99,000 real Brazilian e-commerce orders using Python and pandas.

## Business Question

Does late delivery predict lower customer review scores on the Olist marketplace, and which sellers or states show the worst delivery performance?

## Key Findings

- **Late delivery is strongly associated with lower satisfaction.** On-time/early orders average a **4.29** review score (out of 5); late orders average **2.27** — a ~2-point drop.
- **Dissatisfaction scales with delay severity**, dropping roughly a full point per bucket before leveling off around 8-14 days late — consistent with a "floor effect" once a customer is already maximally dissatisfied.

| Delay | Avg. Review Score |
|---|---|
| On-time or early | 4.29 |
| 1-3 days late | 3.29 |
| 4-7 days late | 2.10 |
| 8-14 days late | 1.67 |
| 15+ days late | 1.72 |

- **Regional variation is real but sample-size-dependent.** Alagoas (AL) has both the highest late-delivery rate (21.4%) and one of the lower average review scores among states; high-volume states like São Paulo (40K+ orders, 4.5% late) show the same delay-satisfaction pattern with strong statistical support.
- **The pattern holds at the seller level too**, once sellers with too few orders to be reliable are filtered out (≥50 orders) — most high-late-rate sellers show correspondingly low review scores.

## Tools & Skills Demonstrated

- **Python / pandas** for multi-table cleaning, validation, and relational joins across 5 linked tables (distinct from the SQL-based join work in [Project 1](https://github.com/MarkusVanNiekerk/manufacturing-safety-analysis))
- **matplotlib / seaborn** for visualization
- Structured data-quality auditing: null checks, duplicate detection, cross-table row-count verification at every merge step, and documented judgment calls for every excluded row
- Debugging a real logic bug found *after* initial analysis (a `NaN`-handling inconsistency that silently miscategorized non-delivered orders) — traced to its root cause, fixed, and verified rather than left unresolved

## Data Source

This project uses the [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). The dataset is included in this repository's `Data/` folder for reproducibility, used here for non-commercial portfolio purposes. Company and seller names in the source data have been anonymized (replaced with Game of Thrones house names) by the original publisher.

## Repository Structure

```
├── olist-analysis2-cleaned.ipynb                              # Full analysis: cleaning, joins, delay/satisfaction analysis, charts
├── Olist E-Commerce Delivery Performance & Customer.md         # Full case study / process log
├── Data/                                                       # Source CSVs (see Data Source above)
└── README.md
```

## Full Case Study

The complete process log — including business framing, every data-quality decision and why it was made, and the investigation into the delay/satisfaction relationship — is written up in full here: **[Full Case Study](./Olist%20E-Commerce%20Delivery%20Performance%20%26%20Customer.md)**

## Running Locally

```bash
git clone <this-repo-url>
cd olist-delivery-satisfaction-analysis
pip install pandas numpy matplotlib seaborn
```
Open `olist-analysis2-cleaned.ipynb` in Jupyter or VS Code and run all cells.
