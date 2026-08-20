# DAX Measures — OmniPulse Analytics Hub

All measures below were built and validated against the live model. Grouped by table/theme, with a short note on why each exists where it isn't self-explanatory.

## Sales & Orders
*Table: `ecommerce_data_Orders`*

```dax
Total Sales = SUM(ecommerce_data_OrderDetails[TotalAmount])
```

```dax
Total Quantity = SUM(ecommerce_data_OrderDetails[Quantity])
```

```dax
Total Orders = DISTINCTCOUNT(ecommerce_data_Orders[OrderID])
```

```dax
AOV = DIVIDE([Total Sales], [Total Orders])
```

## Returns
*Table: `ecommerce_data_Returns`*

```dax
Total Returns = COUNTROWS('ecommerce_data_Returns')
```

```dax
Return Rate % = DIVIDE([Total Returns], [Total Orders])
```

## Customers
*Tables: `ecommerce_data_Customers`, `ecommerce_data_CustomerFeedback`*

```dax
Total Customers = DISTINCTCOUNT('ecommerce_data_Customers'[CustomerID])
```

```dax
Average Rating = AVERAGE('ecommerce_data_CustomerFeedback'[Rating])
```

```dax
Total Feedback Count = COUNTROWS('ecommerce_data_CustomerFeedback')
```

### Customer segmentation (derived relationship)

`CustomerSegments` had no foreign key anywhere in the source data. Two of its four segments (`New`, `At-Risk`) also share the same `MinSpend` value (0), so spend alone can't separate them — the logic below adds recency as a second signal instead of guessing.

```dax
Total Spend =
SUMX(
    RELATEDTABLE('ecommerce_data_Orders'),
    CALCULATE(SUM('ecommerce_data_OrderDetails'[TotalAmount]))
)
```

```dax
Assigned SegmentID =
SWITCH(
    TRUE(),
    'ecommerce_data_Customers'[Total Spend] >= 5000, 1,     -- VIP
    'ecommerce_data_Customers'[Total Spend] >= 2000, 2,     -- Loyal
    'ecommerce_data_Customers'[SignupDate] >=
        (CALCULATE(MAX('ecommerce_data_Orders'[OrderDate]), ALL('ecommerce_data_Orders')) - 90), 3,  -- New
    4                                                        -- At-Risk
)
```
`Customers[Assigned SegmentID]` → `CustomerSegments[SegmentID]` (Many-to-One) relates on this calculated column.

## Products & Inventory
*Tables: `ecommerce_data_Products`, `ecommerce_data_Inventory`*

```dax
Total Products = DISTINCTCOUNT('ecommerce_data_Products'[ProductID])
```

```dax
Total Stock Level = SUM('ecommerce_data_Inventory'[StockLevel])
```

```dax
Low Stock Items =
COUNTROWS(
    FILTER(
        'ecommerce_data_Inventory',
        'ecommerce_data_Inventory'[StockLevel] <= 'ecommerce_data_Inventory'[ReorderLevel]
    )
)
```

## Shipping
*Table: `ecommerce_data_Shipping`*

```dax
Total Shipping Cost = SUM('ecommerce_data_Shipping'[ShippingCost])
```

```dax
Average Delivery Days = AVERAGE('ecommerce_data_Shipping'[DeliveryDays])
```

## Employees
*Tables: `ecommerce_data_Employees`, `ecommerce_data_EmployeePerformance`*

```dax
Total Employees = DISTINCTCOUNT('ecommerce_data_Employees'[EmployeeID])
```

```dax
Average KPI Score = AVERAGE('ecommerce_data_EmployeePerformance'[KPI_Score])
```

```dax
Total Bonus Awarded = SUM('ecommerce_data_EmployeePerformance'[BonusAwarded])
```

## Marketing
*Table: `ecommerce_data_MarketingChannels`*

```dax
Total Ad Spend = SUM('ecommerce_data_MarketingChannels (1)'[MonthlyAdSpend])
```
Not filterable against the rest of the model — this table has no shared key with Orders/Customers in the source data, so it's a standalone reference measure only.

## Calendar
*Table: `DimDate`* — a single calculated calendar table (`CALENDAR(DATE(2020,1,1), DATE(2026,12,31))`) replacing 7 separate auto-generated date tables, with `Year`, `Quarter`, `Month`, `MonthYear`, and `Day` columns for drill-down.

## Additional measures (added directly in Power BI Desktop)

```dax
Total Orders average per OrderDate =
AVERAGEX(
    KEEPFILTERS(VALUES('ecommerce_data_Orders'[OrderDate])),
    CALCULATE([Total Orders])
)
```
*Table: `ecommerce_data_Orders`*

```dax
Last Updated = "Last Refreshed: " & FORMAT(TODAY(), "dd/mm/yyyy")
```
*Table: `DimDate`* — used for the "Last Refreshed" label on the report header. Note: uses `TODAY()`, so it reflects the date the report is opened/refreshed, not the data's actual as-of date.

> **Cleanup note:** `Total Orders average per OrderDate 2` is a byte-for-byte duplicate of `Total Orders average per OrderDate` (same DAX, same table) — looks like an accidental double quick-measure. Safe to delete one of them before publishing.

