# UrbanCart: E-commerce Sales & Profitability

This is a fictional portfolio dataset, generated with a fixed random seed. It contains no real customers or real company results. All money is INR, exclusive of tax. It is intentionally clean: unique keys, consistent labels, ISO dates, numeric amounts and no missing required fields. The figures differ from the design mockup. Use the actual CSV results in your report.

## Business brief
UrbanCart sells clothing, electronics and furniture across eight Indian states. Management wants to monitor sales against plan, understand contribution profitability and investigate unprofitable products, discounts and returns. Build an executive overview and a product investigation page. Label the report “Synthetic portfolio project”.

## Import these nine files
| File | Grain / purpose | Unique key |
|---|---|---|
| Sales.csv | One product line in a fulfilled order; ready-to-use financial columns | LineID |
| Products.csv | 45 fictional products in 9 subcategories | ProductID |
| Customers.csv | 1,600 anonymous fictional customer profiles; some may not purchase | CustomerID |
| States.csv | Eight destination states; City is a representative city label, not delivery address | StateID |
| Categories.csv | Three product categories | CategoryID |
| Calendar.csv | Every day of 2024 and 2025 | Date |
| Months.csv | 24 months; shared monthly filtering of actuals and targets | MonthStart |
| Targets.csv | Planned sales and contribution profit by month, state and category | MonthStart + StateID + CategoryID |
| Returns.csv | One return event per returned sales line; reconciliation / reason analysis | ReturnID |

In Power BI use Get data > Text/CSV for each file. Use the filename without extension as the table name. Set IDs to Text; all date fields to Date; quantities/year/month numbers to Whole number; money to Fixed decimal number; DiscountRate to Decimal number formatted as Percentage. Dates use YYYY-MM-DD; rates are fractions (0.10 means 10%). UTF-8 encoding is used.

## Relationships
Create the following active relationships, with single-direction filtering from the left table to the right table. Do not accept extra automatic relationships.

| One side | Many side |
|---|---|
| Months[MonthStart] | Calendar[MonthStart] |
| Calendar[Date] | Sales[OrderDate] |
| Months[MonthStart] | Targets[MonthStart] |
| States[StateID] | Sales[StateID] |
| States[StateID] | Targets[StateID] |
| Categories[CategoryID] | Sales[CategoryID] |
| Categories[CategoryID] | Targets[CategoryID] |
| Products[ProductID] | Sales[ProductID] |
| Customers[CustomerID] | Sales[CustomerID] |

Mark Calendar as the date table using Calendar[Date]. Sort MonthName by MonthNumber. Use Months[Year], Months[Quarter] and Months[YearMonth] in overview slicers and monthly charts so targets and actuals receive the same time filter. Use States[State] and Categories[Category] for the other overview slicers.

Do not connect Months directly to Sales, Categories to Products, or Customers to States: these would add unnecessary competing filter paths. CategoryID in Products is descriptive metadata; use Categories for category filters.

For your first dashboard, keep Returns disconnected: Sales already includes its financial adjustments. For a return-reason page, merge Returns with selected Sales fields on LineID in Power Query into a separate return-analysis query, then relate the dimensions to that query. Never append Returns to Sales or subtract its refunds again. ReturnDate may fall in January 2026; the supplied Calendar is for order dates, not return-processing dates.

## Accounting and metric rules
- Only fulfilled orders are present. No cancelled, pending or abandoned orders are modeled. All returns within the modeled 5–24-day return window are settled as of 2026-01-31.
- GrossSales = Quantity × UnitPrice (before discounts).
- DiscountAmount = GrossSales × DiscountRate, rounded to 2 decimals.
- SalesBeforeReturns = GrossSales − DiscountAmount.
- RefundAmount = returned units × discounted unit price.
- NetSales = SalesBeforeReturns − RefundAmount.
- NetCOGS = (Quantity − ReturnedQuantity) × UnitCost. Assume returned stock is fully recoverable; no write-down is modeled.
- ContributionProfit = NetSales − NetCOGS − ShippingCost − PaymentFee − ReturnHandlingCost.
- Shipping and payment fees are not refunded. Shipping is a line-level allocated cost. There is no separate order shipping charge to add.
- Profit excludes salaries, rent, marketing, tax and other fixed overhead. Label it **Contribution Profit**, not net profit.
- Returns are attributed back to the original order date. This is an order-cohort performance view, not a cash-flow or statutory accounting report.
- Orders = distinct OrderID, including fully returned orders. AOV = NetSales / Orders. When slicing by category, this is category net sales per order containing that category; distinct orders are not additive across categories.
- Unit return rate = returned quantity / sold quantity. This differs from the percentage of orders returned.
- CustomerType is an order-level flag: the customer's first observed order is New, subsequent orders are Repeat. It does not represent acquisition before January 2024, and a customer can appear in both groups within a year. For multiple orders on one date, OrderID determines sequence.
- Discount bands: 0–5% includes 0 and 5%; >5–10% includes 10%; >10–20% includes 20%; >20% is above 20%.
- Targets are independently generated planning assumptions, not calculated from realized actuals. Target profit margin is 15%. Actuals are allowed to overperform or underperform.
- Targets support only month/quarter/year, state and category filters. Hide target visuals on pages filtered by product, customer, payment method, discount band or individual days; no targets exist at those grains.

## Starter DAX measures
Create each measure separately in Sales (the table used to store measures does not change their logic).

```dax
Net Sales = SUM(Sales[NetSales])
```
```dax
Contribution Profit = SUM(Sales[ContributionProfit])
```
```dax
Profit Margin = DIVIDE([Contribution Profit], [Net Sales])
```
```dax
Total Orders = DISTINCTCOUNT(Sales[OrderID])
```
```dax
Average Order Value = DIVIDE([Net Sales], [Total Orders])
```
```dax
Units Sold = SUM(Sales[Quantity])
```
```dax
Returned Units = SUM(Sales[ReturnedQuantity])
```
```dax
Unit Return Rate = DIVIDE([Returned Units], [Units Sold])
```
```dax
Sales Target = SUM(Targets[SalesTarget])
```
```dax
Sales Target Achievement = DIVIDE([Net Sales], [Sales Target])
```
```dax
Profit Target = SUM(Targets[ContributionProfitTarget])
```
```dax
Profit Target Achievement = DIVIDE([Contribution Profit], [Profit Target])
```
```dax
Repeat Customer Revenue = CALCULATE([Net Sales], Sales[CustomerType] = "Repeat")
```

For a year-on-year comparison with the Months slicers, capture the currently visible dates, shift them by 12 months, then remove both time-table filters:

```dax
Net Sales Previous Year =
VAR PriorDates =
    SELECTCOLUMNS(VALUES(Calendar[Date]), "PriorDate", EDATE(Calendar[Date], -12))
RETURN
    CALCULATE(
        [Net Sales],
        REMOVEFILTERS(Months),
        REMOVEFILTERS(Calendar),
        TREATAS(PriorDates, Calendar[Date])
    )
```
```dax
Sales YoY Growth = DIVIDE([Net Sales] - [Net Sales Previous Year], [Net Sales Previous Year])
```
Use YoY at month/quarter/year level with a single selected year, preferably 2025. 2024 has no prior-year baseline. The EDATE convention aligns matching month/day; leap-day daily comparisons need a separate business rule. Format ratio measures as percentages, not sums or averages of row percentages.

## Suggested dashboard
Top cards: Net Sales, Contribution Profit, Profit Margin, Total Orders, Average Order Value, Sales Target Achievement.
Main charts: monthly actual vs target; net sales by state; sales share by category; contribution profit by subcategory; margin by discount band; state sales and margin table.
Detail page: product sales/profit, unit return rate, payment mix and New vs Repeat revenue. Add product, discount and payment filters here; omit target visuals on this detail page.

## Portfolio handoff
Write three findings supported by calculated results. Explain how you validated totals and prevented double-counted orders/refunds. Investigate high-discount orders, furniture economics and differences between sales rank and profit rank. These patterns were intentionally simulated; association does not establish causation or real-world business impact. Do not claim that the project increased a real company's profits.

Validation_Summary.json contains expected annual totals and row counts for reconciliation. No data cleaning is required, but demonstrate Power Query data types, relationships, DAX and business interpretation.
