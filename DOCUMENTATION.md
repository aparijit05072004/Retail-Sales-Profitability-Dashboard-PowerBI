<div align="center">

# 📄 Project Documentation
### Data Cleaning · Modeling · DAX Reference

*Companion document to [README.md](README.md) — everything here is extracted directly from the `.pbix` file's Power Query, DAX engine, and data model.*

</div>

---

## 📑 Contents

1. [Data Source](#1-data-source)
2. [Data Cleaning & Transformation (Power Query / M)](#2-data-cleaning--transformation-power-query--m)
3. [Data Model (Star Schema)](#3-data-model-star-schema)
4. [DAX Measures Catalog](#4-dax-measures-catalog)
5. [Report Pages](#5-report-pages)
6. [Known Limitations / Notes for Reviewers](#6-known-limitations--notes-for-reviewers)

---

## 1. Data Source

| Item | Detail |
|---|---|
| Source file | `retail--multi year.xlsx` (single sheet: `Retail_Sales_MultiYear`) |
| Rows | 5,000 transaction-level records |
| Time span | Jan 1, 2022 – Dec 31, 2025 (4 full years) |
| Grain | 1 row = 1 order line item |

### Raw columns (20)

| Column | Type | Description |
|---|---|---|
| OrderID | Text | Unique order identifier |
| OrderDate | Date | Date of transaction |
| CustomerID / CustomerName | Text | Customer identifiers |
| City / Region | Text | Customer location (5 cities, 4 regions) |
| ProductID / ProductName | Text | Product identifiers |
| Category / Brand | Text | Product classification (5 categories, 5 brands) |
| StoreID / StoreName | Text | Store identifiers (5 stores) |
| Channel | Text | Online / Store |
| SalespersonID | Text | Sales rep handling the order (60 salespeople) |
| PaymentMode | Text | Card / Cash / UPI |
| Quantity | Integer | Units sold |
| UnitPrice | Integer | Price per unit before discount |
| DiscountPct | Decimal | Discount applied (0 – 0.25) |
| CostPrice | Integer | Unit cost to the business |
| ReturnFlag | Integer (0/1) | Whether the order line was returned (~20% flagged) |

### ✅ Data quality check (performed before modeling)

| Check | Result |
|---|:---:|
| Missing / null values across all 20 columns | ✅ None found |
| Duplicate `OrderID`s | ✅ None found |
| Negative or zero `Quantity` / `UnitPrice` | ✅ None found |
| Consistent categorical values (City, Region, Category, Brand, Channel) | ✅ No typos/case mismatches |
| Balanced volume across years (~1,200–1,280 orders/year) | ✅ Trends aren't skewed by uneven volume |

---

## 2. Data Cleaning & Transformation (Power Query / M)

The raw sheet was loaded once, then **split into a star schema** using Power Query. Each dimension table selects only its relevant columns and de-duplicates on its key — this is the standard approach to normalize a flat file into a star schema inside Power BI.

<details>
<summary><b>🔧 Common first steps (applied in every query) — click to expand</b></summary>

```m
Source = Excel.Workbook(File.Contents("...retail--multi year.xlsx"), null, true),
Retail_Sales_MultiYear_Sheet = Source{[Item="Retail_Sales_MultiYear", Kind="Sheet"]}[Data],
Promoted Headers = Table.PromoteHeaders(Retail_Sales_MultiYear_Sheet, [PromoteAllScalars=true]),
Changed Type = Table.TransformColumnTypes(Promoted Headers, { ...20 explicit type casts... })
```
</details>

#### 🧾 Fact Sales
Kept transactional columns, removed duplicate `OrderID`s, and merged against `Dim Store` on a composite key (`StoreID` + `SalespersonID`) to bring in a surrogate `Store Key`:
```m
Removed Other Columns → {OrderID, OrderDate, CustomerID, ProductID, StoreID, SalespersonID,
                          PaymentMode, Quantity, UnitPrice, DiscountPct, CostPrice, ReturnFlag}
Removed Duplicates (by OrderID)
Merged Queries → LEFT OUTER JOIN with Dim Store on {StoreID, SalespersonID}
Expanded → brings in "Dim Store.Store Key"
```

#### 📦 Dim Product
`{ProductID, ProductName, Category, Brand}`, de-duplicated on `ProductID` → **200 unique products.**

#### 👤 Dim Customer
`{CustomerID, CustomerName, City, Region}`, de-duplicated on `CustomerID` → **500 unique customers.**

#### 🏬 Dim Store
`{StoreID, SalespersonID, StoreName, Channel}`, de-duplicated on the `{StoreID, SalespersonID}` pair, then an **index column** (`Store Key`) was added as a clean surrogate key → **300 unique Store × Salesperson combinations** across 5 physical stores and 60 salespeople.

#### 📅 Dim Calender
Generated directly in DAX (not Power Query) as a calculated table spanning the full order-date range:
```dax
Dim Calender =
ADDCOLUMNS(
    CALENDARAUTO(),
    "MONTH", MONTH([Date]),
    "YEAR", YEAR([Date]),
    "QUARTER", QUARTER([Date]),
    "WEEKDAY", WEEKDAY([Date])
)
```

### 🧮 Calculated columns added to Fact Sales

| Column | DAX | Purpose |
|---|---|---|
| `Sales` | `(UnitPrice * Quantity) − (DiscountPct * UnitPrice * Quantity)` | Net revenue per line after discount |
| `Profit` | `Sales − (CostPrice * Quantity)` | Net margin per line |
| `Profit/Loss` | `IF(Profit < 0, "Loss", "Profit")` | Flags loss-making lines |
| `Sales Define` | Buckets Sales into "Low / Medium / High Sale" (<300 / <500 / else) | Sales-tier segmentation |
| `Discount Category` | Buckets DiscountPct into "Low / Medium / High Discount" (≤0.20 / ≤0.45 / else) | Discount-tier segmentation |

---

## 3. Data Model (Star Schema)

<div align="center">

| Relationship | Cardinality | Cross-filter | Active |
|---|:---:|:---:|:---:|
| Fact Sales[CustomerID] → Dim Customer[CustomerID] | Many : 1 | Single | ✅ |
| Fact Sales[ProductID] → Dim Product[ProductID] | Many : 1 | Single | ✅ |
| Fact Sales[Dim Store.Store Key] → Dim Store[Store Key] | Many : 1 | Single | ✅ |
| Fact Sales[OrderDate] → Dim Calender[Date] *(implicit, via time-intelligence)* | Many : 1 | Single | ✅ |

</div>

All relationships are single-direction (Many-to-One into each dimension), which keeps filter propagation predictable — the standard best practice for star schemas.

---

## 4. DAX Measures Catalog

> 💡 **35+ measures** organized into four groups: core KPIs, time-intelligence, ratios, and exploratory/diagnostic measures built while practicing filter-context functions.

### 🔑 Core KPI measures
| Measure | DAX |
|---|---|
| `Total Sales` | `SUM('Fact Sales'[Sales])` |
| `Total Profit` | `SUM('Fact Sales'[Profit])` |
| `Total Orders` | `COUNTA('Fact Sales'[OrderID])` |
| `total customer` | `COUNTA('Dim Customer'[CustomerID])` |
| `Total Products` | `COUNTA('Dim Product'[ProductID])` |
| `Total_qnt_sold` | `SUM('Fact Sales'[Quantity])` |
| `Avg_sale_value` | `DIVIDE([Total Sales], [Total Orders])` |
| `Total Gross Sales` | `SUMX('Fact Sales', UnitPrice * Quantity)` (before discount) |

### ⏱️ Time-Intelligence measures
| Measure | DAX | Business meaning |
|---|---|---|
| `TotalYTD(Sales)` | `TOTALYTD([Total Sales], 'Dim Calender'[Date])` | Year-to-date sales |
| `DatesYTD(PROFIT)` | `CALCULATE([Total Profit], DATESYTD('Dim Calender'[Date]))` | Year-to-date profit |
| `DatesMTD` | `CALCULATE([Total Sales], DATESMTD('Dim Calender'[Date]))` | Month-to-date sales |
| `DatesQTD` | `CALCULATE([Total Sales], DATESQTD('Dim Calender'[Date]))` | Quarter-to-date sales |
| `TotalMtd(Profit)` | `TOTALMTD([Total Profit], 'Dim Calender'[Date])` | Month-to-date profit |
| `PrevYear` | `CALCULATE([Total Sales], PREVIOUSYEAR('Dim Calender'[Date]))` | Prior-year sales (for YoY comparison) |
| `PrevMonth(Profit)` | `CALCULATE([Total Profit], PREVIOUSMONTH('Dim Calender'[Date]))` | Prior-month profit |

### 📐 Ratio / % measures
| Measure | DAX | Business meaning |
|---|---|---|
| `Total Profit(%)` | `DIVIDE([Total Profit], CALCULATE([Total Profit], ALL('Dim Product'[Category])))` | Category's share of total profit |
| `Total Sales(%)` | `DIVIDE([Total Sales], CALCULATE([Total Sales], ALL('Dim Product'[Category])))` | Category's share of total sales |
| `Total Sales Region(%)` | `DIVIDE([Total Sales], CALCULATE([Total Sales], ALL('Dim Customer'[Region])))` | Region's share of total sales |
| `Total Quantity % By city` | `DIVIDE(SUM(Quantity), CALCULATE(SUM(Quantity), ALL('Dim Customer'[City])))` | City's share of total quantity |

### 🧪 Diagnostic / exploratory measures

<details>
<summary>Click to expand full list</summary>

`maxprofit`, `minprofit`, `maxup` (max unit price), `minup`, `maxdis` (max discount), `totalup`, `totalcp`, `Total Sales (Brand A)`, `Total Sales (s1)`, `Min Sales(s1)`, `Total Sales(all)`, `Total Profit(all Store)`, `Total Sales (AllExcept Region)`, `TotalSales(PY)` — built while practicing `CALCULATE`, `ALL`, `ALLEXCEPT`, and filter-context modifiers. Retained in the model as a demonstration of DAX filter-context concepts, though not all are used on the report canvas.

</details>

---

## 5. Report Pages

<table>
<tr><td valign="top" width="50%">

### 🟦 Page 1 — Overview
General health-check of the business:
- 5 KPI cards (Sales, Profit, Customers, Orders, Products)
- Sales by Region (column) & Profit by Region (pie)
- Sales by Channel (pie)
- Sales by Brand × City (bar)
- Sales by Product (bar)
- Category-level Profit/Sales/Qty table
- **Slicers:** Region, City, Category, Year, Month

</td><td valign="top" width="50%">

### 🟩 Page 2 — Sales Trend & Profitability Analysis
Deeper, time- and performance-oriented view built to complement (not repeat) Page 1:
- Current-year vs. previous-year sales trend line
- Monthly Sales & Profit trend
- MTD vs. previous-month profit trend
- Sales & Profit by Store (bar)
- Top customers by Sales (table)
- Orders & Sales by Salesperson (table)
- **Slicers:** Year, Quarter, Store, Channel, Brand

</td></tr>
</table>

---

## 6. Known Limitations / Notes for Reviewers

| ⚠️ Note | Detail |
|---|---|
| Legacy scratch page | The workbook still contains one page ("Page 9") used while practicing time-intelligence DAX (QTD/YTD/MTD lines). Not part of the 2-page business report — hide or delete for a fully polished submission. |
| Calculated columns | `Sales` and `Profit` are calculated columns (not Power Query steps), so they recompute at model-refresh time. This is intentional — they depend on row-level `DiscountPct` and `CostPrice`. |
| Returns not excluded | Return-flagged orders (`ReturnFlag = 1`, ~20% of rows) are **not excluded** from Sales/Profit — they remain as raw transactions. For a "net of returns" view, add: `CALCULATE([Total Sales], 'Fact Sales'[ReturnFlag] = 0)` |

<div align="center">

---
📘 Back to [README.md](README.md)

</div>
