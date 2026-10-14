# DAX Measure Definitions

Semantic model: `sm_retail` (Direct Lake). Tables: `gold_fact_sales`, `gold_fact_returns`, `gold_dim_customer`, `gold_dim_product`, `gold_dim_date`, `gold_cohort_retention`, `dq_cleaning_rules`.

Relationships (many-to-one, single direction, fact to dimension): sales and returns each relate to customer (`customer_id`), product (`stock_code`), and date (`date_key`). `gold_cohort_retention` and `dq_cleaning_rules` have no relationships.

Customer id `-1` is the "Unknown" placeholder for lines with no customer ID.

## How the measures behave with filters

Two groups behave differently, which explains why some numbers on the report change with the date filter and some do not.

- **Transaction measures** (Revenue, Orders, Identified Customers, Returns) come from the fact tables, so they follow the date and country slicers.
- **Customer status measures** (Active, Churned, Churn Rate, Repeat Customer Rate) come from `gold_dim_customer`, where status is a snapshot as of 9 Dec 2011. They do not change with the date slicer.

This is why Identified Customers can read 4,293 on the overview (December 2011 excluded) while the churn table holds 4,334 customers.

---

## Sales

**Revenue**
Total value of valid sale lines.
Revenue = SUM(gold_fact_sales[line_amount])


**Orders**
Number of distinct invoices.
Orders = DISTINCTCOUNT(gold_fact_sales[invoice_no])

**Avg Order Value**
Revenue per invoice.
```dax
Avg Order Value = DIVIDE([Revenue], [Orders])
```

**Identified Customers**
Distinct customers with an ID who bought in the selected period (excludes the Unknown placeholder).
```dax
Identified Customers =
CALCULATE(
    DISTINCTCOUNT(gold_fact_sales[customer_id]),
    gold_fact_sales[customer_id] <> -1
)
```

## Returns

**Returns Value**
Value of cancelled lines, shown as a positive number (cancellation amounts are stored as negatives).
```dax
Returns Value = ABS(SUM(gold_fact_returns[line_amount]))
```

**Return Rate**
Cancelled order value as a share of sales revenue.
```dax
Return Rate = DIVIDE([Returns Value], [Revenue])
```

## Customer status (snapshot as of 9 Dec 2011)

**Active Customers**
Last valid purchase on or after the cutoff (10 Sep 2011).
```dax
Active Customers =
CALCULATE(
    COUNTROWS(gold_dim_customer),
    gold_dim_customer[customer_status] = "Active"
)
```

**Churned Customers**
Last valid purchase before the cutoff.
```dax
Churned Customers =
CALCULATE(
    COUNTROWS(gold_dim_customer),
    gold_dim_customer[customer_status] = "Churned"
)
```

**Customers Judged**
Customers with enough history to be classed as Active or Churned. Used as the denominator and as a minimum-sample filter (20 or more) on the churn-by-country chart.
```dax
Customers Judged = [Active Customers] + [Churned Customers]
```

**Churn Rate**
Churned share of customers that can be judged. "Too new" customers are left out of the denominator because they have not had 90 days to churn.
```dax
Churn Rate = DIVIDE([Churned Customers], [Customers Judged])
```

**Revenue from Churned Customers**
Total revenue from customers now classed as churned.
```dax
Revenue from Churned Customers =
CALCULATE([Revenue], gold_dim_customer[customer_status] = "Churned")
```

**Repeat Customer Rate**
Share of identified customers with two or more orders at any point in the data.
```dax
Repeat Customer Rate =
DIVIDE(
    CALCULATE(
        COUNTROWS(gold_dim_customer),
        gold_dim_customer[customer_id] <> -1,
        gold_dim_customer[order_count] > 1
    ),
    CALCULATE(
        COUNTROWS(gold_dim_customer),
        gold_dim_customer[customer_id] <> -1
    )
)
```
Note: this measures buying again at any time, not in the following month, so it is much higher than month-1 cohort retention. Customers who first bought late in the data had less time to return.

## Cohort retention

**Retention %**
Share of a cohort's customers who purchased again N months after their first purchase. The values are precomputed in `gold_cohort_retention`, so `MAX` simply returns the single value for each cohort and month cell.
```dax
Retention % = MAX(gold_cohort_retention[retention_rate])
```
Matrix layout: `cohort_month` on rows, `month_index` on columns, totals turned off (totals of percentages are not meaningful). The color scale is capped at 0 to 0.4 so that values below 40% are distinguishable.

---

## Number formats

| Measure | Format |
|---|---|
| Revenue, Avg Order Value, Returns Value, Revenue from Churned Customers | Currency (£), thousands separator |
| Orders, Identified Customers, Active / Churned / Judged | Whole number, thousands separator |
| Return Rate, Churn Rate, Repeat Customer Rate, Retention % | Percentage |
