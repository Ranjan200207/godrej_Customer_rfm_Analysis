## godrej RFM Customer Analytics and Revenue Analysis

A customer analytics project that combines SQL data preparation, Python-based RFM segmentation, and a Power BI report. The analysis explores transaction revenue, customer activity, customer value, churn risk, and RFM segments.

> **Data and branding note:** The Power BI design uses Godrej branding, while the supplied transaction records appear to be general online retail data (invoices, stock codes, quantities, prices, customers, and countries). The project files do not verify that the transactions belong to Godrej. Confirm the source and usage rights before presenting or publishing the analysis as Godrej company data.

## Project goals

- Combine and validate transaction data from two source periods.
- Clean transaction fields and create line-level revenue.
- Build customer-level Recency, Frequency, and Monetary (RFM) metrics.
- Score and segment customers using explicit business rules.
- Compare customer revenue and activity by month, country, and segment.
- Explore the results in an interactive Power BI report.

## Tools

- **SQL Server  - combine source tables, convert types, audit data quality, remove exact duplicate transaction rows, and query the cleaned table.
- **Python** - pandas for data preparation and analysis; matplotlib for charts; Jupyter Notebook for the workflow.
- **Power BI Desktop** - KPI cards, slicers, revenue and customer visuals, churn views, and segment summary tables.

## Data workflow

### 1. SQL preparation

The SQL scripts combine the `Year 2009-2010` and `Year 2010-2011` tables into `dbo.RetailCombined`. Dates are parsed using supported input formats, prices are converted to decimal values, and a `SourceSheet` field records the origin period. The scripts check for failed date/price conversions, missing customer IDs or descriptions, non-positive quantities, and duplicate transaction rows. A deduplicated table is written to `dbo.RetailCleaned` and queried for validation/export.

### 2. Python cleaning and RFM analysis

The notebook reads `Retailfixed.csv` and works with these fields:

`Invoice`, `StockCode`, `Description`, `Quantity`, `InvoiceDate`, `Price`, `Customer_ID`, `Country`, and `SourceSheet`.

It parses dates, normalizes customer IDs and numeric fields, and calculates line revenue:

```text
LineRevenue = Quantity * Price
```

For customer purchase behavior, it excludes cancelled invoices (invoice numbers beginning with `C`) and rows with non-positive quantity or price. Frequency is the distinct count of completed invoices; last purchase date is the latest valid purchase date. Monetary value sums customer transaction line revenue, including negative lines so returns/credits reduce customer monetary value.

The snapshot date is the day after the last transaction date in the data: **2011-12-10** in the supplied run. Recency is the number of days between this snapshot and each customer's last valid purchase. Customers without a valid purchase date are excluded from RFM scoring.

### RFM scoring

The notebook assigns five relative score bands using `pandas.qcut` over ranked values:

- **R score:** lower recency (more recent purchase) gets the higher score, 5 to 1.
- **F score:** more distinct invoices gets the higher score, 1 to 5.
- **M score:** greater monetary value gets the higher score, 1 to 5.

Because these are quantile-based scores, the cutoffs depend on the data being analyzed. They are relative rankings, not universal customer-value thresholds.

### Customer status and segment rules

The customer-level churn flag is:

```text
IsChurned = Recency > 90 days
```

Segments are assigned in this order:

1. **Lost** - Recency is greater than 90 days.
2. **Champions** - R, F, and M scores are all at least 4.
3. **Loyal** - R score is at least 3 and F score is at least 3.
4. **At Risk** - R score is at most 2 and F score is at least 3.
5. **New** - R score is at least 4 and F score is at most 2.
6. **Needs Attention** - customers not matched by the rules above.

The rule order matters: customers more than 90 days from their last purchase are assigned to Lost before the other conditions are checked.

## Power BI report

The supplied report includes an overview of revenue and customer KPIs, date/country/segment/churn slicers, revenue trends and segment breakdowns, churn visuals, and a segment performance table with customer counts, R/F/M scores, and revenue. The report is powered by transaction-level data and customer-level RFM results.

For accurate filtering, keep the model grain in mind: transaction rows belong in the transaction/fact table, while RFM attributes belong at customer grain. Use a unique customer key to relate them, and use a dedicated Calendar table for date filtering and month/quarter/year axes. Validate that country and date filters reach the tables used by each measure.

## Results from the supplied Python run

The notebook found 5,942 customers with purchase history; 64 did not have a valid purchase date for RFM, leaving **5,878 customers** in the scored RFM table.

| Segment | Customers | Customer share | Net monetary value | Revenue share |
|---|---:|---:|---:|---:|
| Champions | 1,290 | 21.95% | 11,396,998.70 | 69.70% |
| Lost | 2,989 | 50.85% | 3,025,670.08 | 18.50% |
| Loyal | 971 | 16.52% | 1,605,424.88 | 9.82% |
| New | 441 | 7.50% | 220,858.26 | 1.35% |
| Needs Attention | 187 | 3.18% | 102,089.02 | 0.62% |
| **Total** | **5,878** | **100.00%** | **16,351,040.94** | **100.00%** |

The notebook also creates monthly revenue/activity summaries, country summaries, a customer-level EIRE analysis, and charts for monthly revenue, top-country revenue, and revenue share by segment.

## Important interpretation and validation notes

- **Churn is a snapshot, not historical monthly churn.** `IsChurned` is calculated from recency relative to 2011-12-10. Plotting that static customer flag by transaction month does not by itself calculate churn as of each historical month. A historical churn trend requires a period-specific definition and measure (for example, no purchase in the 90 days before each month-end).
- **Reconcile the revenue KPI before publishing.** The Python RFM segment table totals **16,351,040.94** in net monetary value, while the supplied Power BI overview shows a Total Revenue KPI of approximately **18.86M**. These values may use different row inclusion, return/cancellation, or filter logic. Align the definitions and confirm the measure before treating them as the same metric.
- Check that totals shown in visuals respond correctly to slicers. Repeated country/customer values can indicate a relationship or filter-context issue.
- The example notebook uses a local absolute path for `Retailfixed.csv`. Update it to a relative or configurable path before running on another computer.
- Confirm dataset provenance and applicable license/permissions before committing raw data to a public repository. Do not publish customer-identifiable or confidential records.

## Reproducing the analysis

1. **SQL Server:** Load the two source-period tables with the names expected by `SQLQuery3.sql` (`Year 2009-2010` and `Year 2010-2011`). Run `SQLQuery3.sql` to create and clean the combined table. Use `SQLQuery1.sql` to validate dates and query the cleaned data; `SQLQuery7.sql` is a simple cleaned-table query.
2. **Python:** Install Python, Jupyter, pandas, and matplotlib. Open `godrej_rfm_Analysis.ipynb`, update the input file path and output directory for your machine, then run the notebook cells in order.
3. **Power BI:** Open the PBIX in Power BI Desktop. If prompted, update the data-source paths to the CSV exports in your local project folder and refresh. Check relationships and validate core totals before using the visuals.

## Possible next improvements

- Move local paths and the 90-day churn threshold into configuration variables.
- Add a `requirements.txt` and a data dictionary.
- Reconcile Python and Power BI revenue definitions and add validation checks.
- Create historical churn measures evaluated at each date period.
- Add a Calendar dimension and verify sorting and filter propagation in the Power BI model.
- Keep large/private data outside Git and document how authorized users can provide it locally.


