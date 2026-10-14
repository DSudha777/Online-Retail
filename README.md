# Online-Retail
# Customer Retention & Churn Analytics on Microsoft Fabric

End-to-end analytics solution built on Microsoft Fabric: raw transactions land in a Bronze layer, are cleaned in Silver, modeled as a star schema in Gold, and served through a Direct Lake semantic model to a four-page Power BI report.

**Business question:** Do customers come back, who are we losing, and how much revenue is at stake?

## Headline findings

- **61.4%** of identified-customer revenue comes from the top 10% of customers.
- **11.6%** of identified revenue comes from customers who have not bought in 90+ days.
- About **1 in 5** new customers buys again in their first month (15-24% across the 2011 cohorts).
- **43%** of customers with enough history are classed as churned under a 90-day rule.
- Revenue peaks in **November 2011** at about 2x the mid-year level, consistent with holiday stocking.

> Figures are for 1 Dec 2010 to 30 Nov 2011. December 2011 is excluded because the data ends on 9 Dec.

## Data

- **Source:** UCI Machine Learning Repository, *Online Retail* dataset (transactions from a UK online retailer). License: [confirm on the UCI dataset page].
- **Size:** 541,909 raw rows, 1 Dec 2010 to 9 Dec 2011.
- **Columns:** InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country.
- **Currency:** [confirm on the UCI page; the report uses £].

## Architecture

```
Raw CSV -> Bronze -> Silver -> Gold -> Direct Lake semantic model -> Power BI report
          (as-is)   (typed,    (star
                     flagged)   schema)
```

| Layer | What happens |
|---|---|
| **Bronze** | CSV loaded unchanged as strings, with source file and ingestion timestamp added. |
| **Silver** | Types applied, exact duplicates removed, rows flagged (cancellation, product, customer present, valid sale) instead of deleted. |
| **Gold** | Star schema, plus a cohort retention table. Customer status is derived here. |
| **Semantic model** | Direct Lake model with DAX measures. |
| **Report** | Four pages: overview, cohort retention, churn, data quality. |

All Delta tables live in a Fabric Lakehouse. Notebooks use PySpark and Spark SQL.

## Data cleaning and reconciliation

Rows are flagged, not deleted, so nothing is lost and every number can be traced.

| Step | Rows |
|---|---|
| Raw rows (Bronze) | 541,909 |
| Exact duplicates removed | -5,268 |
| Cancellations (kept in a returns table) | -9,251 |
| Other excluded (non-product lines, zero or negative price) | -4,886 |
| **Valid sale lines (sales fact)** | **522,504** |

Flags that overlap, for reference: 135,037 lines have no customer ID, 2,989 are non-product lines (postage, fees), and 2,516 have a zero or negative price.

**Decisions and assumptions**

- **Cancellations** (invoice starts with "C") are kept in `gold_fact_returns` to report a return rate.
- **Missing customer ID** (about 25% of lines, 14.7% of revenue): counted in revenue totals, excluded from retention, cohort, and churn analysis. Joined to an "Unknown" customer row so no fact row is orphaned.
- **Non-product lines:** stock codes that do not start with five digits are treated as non-products. Reviewed manually.
- **Duplicates:** exact duplicate rows are removed (about 1% of rows). Some repeats could be legitimate.

## Data model

Star schema, one fact table per business process:

- `gold_fact_sales` (valid sale lines)
- `gold_fact_returns` (cancellations)
- `gold_dim_customer` (country, first and last purchase, order count, status)
- `gold_dim_product`
- `gold_dim_date`
- `gold_cohort_retention` (cohort month x months since first purchase)
- `dq_cleaning_rules` (row counts for the data-quality page)

## Churn definition

- **As-of date:** 9 Dec 2011, the last invoice in the data (not today's date).
- **Cutoff:** 10 Sep 2011, which is 90 days before the as-of date.
- **Active:** last valid purchase on or after the cutoff.
- **Churned:** last valid purchase before the cutoff.
- **Too new:** first purchase on or after the cutoff; too early to judge, excluded from Churn Rate.
- **Unknown:** lines with no customer ID.

Churn status is a **snapshot as of 9 Dec 2011** and does not change with the report's date filter. The 90-day window is a choice; many customers are wholesalers who buy a few times a year, so the churn rate is likely overstated.

## Key measures (DAX)

Revenue, Orders, Avg Order Value, Identified Customers, Returns Value, Return Rate, Active Customers, Churned Customers, Churn Rate (Churned / (Active + Churned)), Repeat Customer Rate, Revenue from Churned Customers, Retention %.

## Report pages

1. **Executive overview:** revenue, orders, average order value, return rate, monthly trend, revenue by country.
2. **Cohort retention:** monthly acquisition cohorts, customers active and retention % by months since first purchase.
3. **Churn:** status breakdown, churn rate, churn by country (countries with 20+ customers), revenue from churned customers.
4. **Data quality:** cleaning rules, row reconciliation, churn definition, limitations, lineage.

Step	Rows
Raw rows (Bronze)	541,909
Exact duplicates removed	−5,268
Cancellations (moved to returns table)	−9,251
Other excluded (non-product, zero price, negative quantity)	−4,886
Valid sale lines (Gold sales fact)	522,504


## Limitations

- About one year of history, so cohorts span at most 12 months.
- The Dec 2010 cohort includes customers who were already buying before the data starts.
- Dec 2011 is a partial month and is excluded; 41 customers whose only purchases were in Dec 2011 therefore do not appear in the report pages (4,334 in the churn table vs 4,293 in the report).
- About 25% of lines have no customer ID, so retention and churn describe identified customers only.
- Country-level churn is only shown where there are enough customers; small samples are noisy.
- The November 2011 uptick in most cohorts reflects seasonal buying, so month-over-month retention after month 1 should not be read as steady loyalty.
- The UK accounts for about 85% of revenue.


## How to reproduce

1. Create a Fabric workspace and a Lakehouse.
2. Upload the CSV to `Files/raw/online_retail.csv`.
3. Run the notebooks in order: Bronze, Silver, Gold, cohort retention.
4. Create the semantic model from the Gold tables, set relationships and add the measures.
5. Open the report and refresh.

## Status

- [x] Bronze / Silver / Gold layers
- [x] Direct Lake semantic model and DAX measures
- [x] Four-page report
- [ ] Row-level security
- [ ] Dev / Test / Prod deployment pipeline

*(Update this list once the last two items are done.)*

## Tools

Microsoft Fabric (Lakehouse, Notebooks, Direct Lake), Spark SQL, PySpark, Power BI, DAX.
