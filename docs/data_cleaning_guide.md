# Data Cleaning Guide — Detailed Documentation

This document provides a complete, step-by-step reference for every Power Query transformation applied in this project. It is intended as both a learning guide and a technical reference for reproducing or extending the workflow.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Source Data Description](#2-source-data-description)
3. [Step 1 — Connect Data](#3-step-1--connect-data)
4. [Step 2 — Filter Data](#4-step-2--filter-data)
5. [Step 3 — Clean Data](#5-step-3--clean-data)
6. [Step 4 — Transform Data](#6-step-4--transform-data)
7. [Step 5 — Combine Data](#7-step-5--combine-data)
8. [Step 6 — Clean Up](#8-step-6--clean-up)
9. [Query-by-Query Reference](#9-query-by-query-reference)
10. [Key Concepts](#10-key-concepts)
11. [Common Pitfalls](#11-common-pitfalls)

---

## 1. Project Overview

This project demonstrates a professional data cleaning workflow in **Power BI Power Query**. The goal is to take four messy source tables and produce a clean, analysis-ready data model.

**The core principle:** Follow a structured, repeatable 6-step process rather than applying transformations randomly. This makes the workflow easier to understand, debug, and maintain.

```
Connect → Filter → Clean → Transform → Combine → Clean Up
```

---

## 2. Source Data Description

### 2.1 `sales_flattable.csv`

The primary transactional dataset. Contains 13 rows (including 3 completely empty rows and 1 duplicate).

**Schema:**

| Column | Type | Description | Known Issues |
|---|---|---|---|
| `OrderID` | Whole Number | Unique order identifier | — |
| `Product_Info` | Text | Product name + category code, pipe-delimited | Compound field: `"Laptop Pro \| ELEC"` |
| `First_Name` | Text | Customer first name | `#` prefix, inconsistent casing, extra spaces |
| `Last_Name` | Text | Customer last name | Inconsistent casing, extra spaces |
| `Email` | Text | Customer email | Inconsistent casing, trailing spaces, missing values |
| `Customer_ID` | Text | Customer identifier with country suffix | Format: `C-NNNN-XX` |
| `Amount` | Decimal | Order quantity | Missing values (whitespace-only), decimals |
| `Price` | Decimal | Unit price | Missing values, decimals |
| `Order_Date` | Text | Order date | Prefixed with `D`, invalid dates (`D2024-99-13`) |
| `Technical_Log_ID` | Text | System log reference | Not useful for analysis — remove |

**Data quality issues catalog:**

| Row | Issue | Column | Value |
|---|---|---|---|
| 2 | `#` prefix in name | `First_Name` | `#ANna` |
| 3 | `#` prefix, inconsistent casing | `First_Name` | `#Jean` |
| 4 | Leading whitespace | `First_Name` | `"   Luca"` |
| 4 | Whitespace-only value | `Amount` | `" "` |
| 5 | Trailing whitespace in email | `Email` | `"carlos@email.com "` |
| 7 | Duplicate of row 2 | All columns | Exact copy of OrderID 1001 |
| 8 | Trailing whitespace in name | `First_Name` | `"Sofia  "` |
| 9, 12, 14 | Completely empty rows | All columns | All commas, no data |
| 10 | Missing email | `Email` | `"  "` (whitespace only) |
| 11 | `#` prefix + trailing whitespace | `First_Name` | `"#Claire  "` |
| 13 | Missing product | `Product_Info` | `" "` |
| 13 | Invalid date | `Order_Date` | `D2024-99-13` (month 99) |
| 13 | Missing price | `Price` | Empty |

### 2.2 `sales_monthly.csv`

Aggregated customer-level sales and cost data across three months.

| Column | Description |
|---|---|
| `Customer` | Customer name |
| `Jan Sales` / `Jan Cost` | January figures |
| `Feb Sales` / `Feb Cost` | February figures |
| `Mar Sales` / `Mar Cost` | March figures |

### 2.3 `orders_jan.csv`

| Column | Type | Description |
|---|---|---|
| `Order_ID` | Whole Number | Order identifier |
| `Product` | Text | Product name |
| `Sales` | Whole Number | Sales amount |

Contains 3 records for January.

### 2.4 `orders_feb.csv`

Same schema as `orders_jan.csv`. Contains 4 records for February.

---

## 3. Step 1 — Connect Data

**Purpose:** Import source files into Power Query as individual queries.

### What was done

| Query | Source | Connection Method |
|---|---|---|
| `Sales` | `sales_flattable.csv` | CSV file import |
| `Categories` | Manual/reference table | Created as lookup |
| `Orders` | `orders_jan.csv` | CSV file import |
| `orders_feb` | `orders_feb.csv` | CSV file import |

### Applied Steps (common)
1. **Source** — Point to the CSV file path
2. **Promoted Headers** — Use the first row as column headers

### Key decisions
- Each source gets its own query to keep transformations isolated
- The `Categories` table was created as a small lookup/reference table to map short codes (`ELEC`, `ACC`, `FURN`) to full names

---

## 4. Step 2 — Filter Data

**Purpose:** Reduce dataset size early by removing data that is not needed for the final model.

### Operations on Sales

| Operation | What it does | Why |
|---|---|---|
| **Remove Columns** | Dropped `Technical_Log_ID` and other system columns | Not relevant for business analysis |
| **Remove Blank Rows** | Deleted rows where all columns are empty | Rows 9, 12, 14 were entirely empty |
| **Remove Duplicates** | Eliminated exact duplicate records | Row 7 was an exact copy of row 2 |

### Best practice
> **Always filter early.** Removing unnecessary data before cleaning and transforming reduces processing time, avoids applying transformations to junk data, and keeps the Applied Steps panel manageable.

---

## 5. Step 3 — Clean Data

**Purpose:** Fix data quality problems — text inconsistencies, missing values, errors.

### 5.1 Text Cleaning

Applied in this order to ensure consistent results:

| Step | M Function | Purpose | Example |
|---|---|---|---|
| 1. Trim Text | `Text.Trim` | Remove leading/trailing spaces | `"   Luca"` → `"Luca"` |
| 2. Lowercase | `Text.Lower` | Normalize all text to lowercase | `"ANNA"` → `"anna"` |
| 3. Capitalize Each Word | `Text.Proper` | Proper case for display | `"anna schmidt"` → `"Anna Schmidt"` |

**Why this order matters:**
- Trim first to remove spaces that could interfere with casing
- Lowercase first to create a uniform base
- Then capitalize for consistent display format

### 5.2 Value Cleaning

| Operation | M Function | What was fixed |
|---|---|---|
| Replace Values | `Table.ReplaceValue` | Removed `#` prefix from names, replaced known bad values |
| Replace Errors | `Table.ReplaceErrorValues` | Handled formula/conversion errors gracefully |

**Specific replacements:**
- `#` → `""` (empty string) in `First_Name`
- Various `n/a`, `null` placeholders → standardized representation

### 5.3 Duplicate Handling

- **Remove Duplicates** was applied after text cleaning to ensure that records differing only in casing or whitespace are correctly identified as duplicates
- Example: `#Anna Schmidt` and `#ANna ScHmidt` become the same record after cleaning

---

## 6. Step 4 — Transform Data

**Purpose:** Reshape the data structure — split compound fields, extract substrings, fix data types, create new columns.

### 6.1 Data Types

Multiple type assignments were needed because some operations (like Split Column) reset types:

| Column | Final Type | Notes |
|---|---|---|
| `Order_Id` | Whole Number | — |
| `Sales` / `Amount` / `Price` | Decimal/Currency | — |
| `Category` | Text | — |
| `Order_Date` | Date | After removing the `D` prefix |

### 6.2 Split Column by Delimiter

**Target column:** `Product_Info`
**Delimiter:** `|` (pipe)
**Result:**

| Before | After (Col 1) | After (Col 2) |
|---|---|---|
| `Laptop Pro \| ELEC` | `Laptop Pro` | `ELEC` |
| `Gaming Mouse \| ACC` | `Gaming Mouse` | `ACC` |

The resulting columns were later renamed to `Product_Name` and `Category`.

### 6.3 Extract Text

| Operation | Source Column | Purpose | Example |
|---|---|---|---|
| Extract Last Characters | `Customer_ID` | Get country code | `C-1001-DE` → `DE` |
| Extract Text After Delimiter | Various | Parse specific segments | Field-specific extraction |

### 6.4 Add Custom Column

In both `Orders` and `orders_feb`, a new column `Tab` was added to identify the source month:

**M formula (Orders):**
```
= Table.AddColumn(#"Previous Step", "tab", each "jan")
```

**M formula (orders_feb):**
```
= Table.AddColumn(#"Previous Step", "tab", each "feb")
```

This is essential for maintaining traceability after the tables are appended together.

### 6.5 Rename Columns

| Original Name | New Name | Query |
|---|---|---|
| `Column1` | `Category_Short` | Categories |
| `Column2` | `Category_Name` | Categories |
| `Order_ID` | `Order_Id` | Orders |
| `tab` | `Tab` | Orders |

---

## 7. Step 5 — Combine Data

**Purpose:** Integrate multiple tables into unified datasets.

### 7.1 Append Queries (SQL UNION equivalent)

**What:** Stack `Orders` and `orders_feb` vertically into one combined table.

**Requirements:**
- Both tables must have the same column structure (names + types)
- The `Tab` column preserves which month each row came from

**Result:**

| Tab | Order_Id | Product | Sales |
|---|---|---|---|
| jan | 1 | Laptop | 1200 |
| jan | 2 | Mouse | 300 |
| jan | 3 | Keyboard | 500 |
| feb | 3 | Keyboard | 500 |
| feb | 4 | Laptop | 1400 |
| feb | 5 | Mouse | 350 |
| feb | 6 | Keyboard | 520 |

### 7.2 Merge Queries (SQL JOIN equivalent)

**What:** Enrich the `Sales` table with category names from the `Categories` lookup.

**Join details:**
- **Left table:** Sales
- **Right table:** Categories
- **Join column:** `Category` (Sales) = `Category_Short` (Categories)
- **Join type:** Left Outer Join

**After expanding the merged column:**

| Category | Category_Name |
|---|---|
| ELEC | Electronics |
| ACC | Accessories |
| FURN | Furnitures |
| n/a | Not Available |

### Merge vs. Append — When to use which

| Operation | When to use | SQL equivalent |
|---|---|---|
| **Append** | Stacking tables with the same columns (adding more rows) | `UNION ALL` |
| **Merge** | Joining tables on a shared key (adding more columns) | `LEFT JOIN` / `INNER JOIN` |

---

## 8. Step 6 — Clean Up

**Purpose:** Final polish — remove helper columns, standardize naming, and arrange columns logically.

### 8.1 Remove Unnecessary Columns

After all transformations, some helper/intermediate columns remain. These were removed:
- Duplicate columns created during splits
- Intermediate calculation columns
- System/technical fields

### 8.2 Rename Columns

Final naming convention applied across all queries:
- `PascalCase` with underscores: `Order_Id`, `Product_Name`, `Category_Name`, `Customer_ID`
- Consistent abbreviation: `Id` (not `ID` or `id`)

### 8.3 Reorder Columns

Columns were arranged in a logical business order, not the random order produced by transformations:

**Final Sales table schema:**

| Position | Column | Type |
|---|---|---|
| 1 | `Order_Id` | Whole Number |
| 2 | `Product_Name` | Text |
| 3 | `Category` | Text |
| 4 | `Category_Name` | Text |
| 5 | `Customer_ID` | Text |
| 6 | `Customer_Name` | Text |
| 7 | `Email` | Text |
| 8 | `Amount` | Decimal |
| 9 | `Price` | Decimal |
| 10 | `Order_Date` | Date |
| 11 | `Country` | Text |
| 12 | *(remaining fields)* | — |

**Final Orders table schema:**

| Position | Column | Type |
|---|---|---|
| 1 | `Tab` | Text |
| 2 | `Order_Id` | Whole Number |
| 3 | `Product` | Text |
| 4 | `Sales` | Whole Number |

---

## 9. Query-by-Query Reference

### Sales (Main query — 30+ applied steps)

| # | Step Name | Category | Description |
|---|---|---|---|
| 1 | Source | Connect | Import `sales_flattable.csv` |
| 2 | Promoted Headers | Connect | Use first row as headers |
| 3 | Changed Column Type | Transform | Assign initial data types |
| 4 | Removed Columns | Filter | Drop `Technical_Log_ID` |
| 5 | Removed Blank Rows | Filter | Delete empty rows |
| 6 | Removed Duplicates | Filter/Clean | Eliminate exact duplicates |
| 7 | Trimmed Text | Clean | Remove whitespace from text columns |
| 8 | Replaced Value | Clean | Remove `#` prefix from names |
| 9 | Capitalized Each Word | Clean | Proper case for names |
| 10 | Trimmed Text1 | Clean | Additional trim pass |
| 11 | Capitalized Each Word1 | Clean | Additional capitalize pass |
| 12 | Trimmed Text2 | Clean | Trim on other columns |
| 13 | Lowercased Text | Clean | Lowercase for email standardization |
| 14 | Replaced Value1 | Clean | Additional value replacements |
| 15 | Changed Type | Transform | Update types after text changes |
| 16 | Replaced Errors | Clean | Handle conversion errors |
| 17 | Replaced Value2 | Clean | Fix specific values |
| 18 | Replaced Value3 | Clean | Fix specific values |
| 19 | Replaced Value4 | Clean | Fix specific values |
| 20 | Split Column by Delimiter | Transform | Split `Product_Info` by `\|` |
| 21 | Changed Type1 | Transform | Fix types after split |
| 22 | Trimmed Text3 | Clean | Trim split results |
| 23 | Replaced Value5 | Clean | Clean split artifacts |
| 24 | Renamed Columns | Transform | Standardize column names |
| 25 | Merged Columns | Transform | Combine first + last name |
| 26 | Duplicated Column | Transform | Create working copy |
| 27 | Duplicated Column1 | Transform | Create working copy |
| 28 | Extracted Last Characters | Transform | Extract country from `Customer_ID` |
| 29 | Extracted Text After Delimiter | Transform | Parse additional fields |
| 30 | Replaced Value6 | Clean | Final value fixes |
| 31 | Merged Queries | Combine | Join with Categories |
| 32 | Expanded Categories | Combine | Bring in `Category_Name` |
| 33 | Reordered Columns | Clean Up | Logical column order |
| 34 | Renamed Columns1 | Clean Up | Final naming |
| 35 | Reordered Columns1 | Clean Up | Final order |

### Categories (2 steps)

| # | Step Name | Description |
|---|---|---|
| 1 | Source | Load reference data |
| 2 | Renamed Columns | `Column1` → `Category_Short`, `Column2` → `Category_Name` |

### Orders (9 steps)

| # | Step Name | Category | Description |
|---|---|---|---|
| 1 | Source | Connect | Import `orders_jan.csv` |
| 2 | Promoted Headers | Connect | Use first row as headers |
| 3 | Changed Column Type | Transform | Assign data types |
| 4 | Added Custom | Transform | Add `tab` = `"jan"` |
| 5 | Changed Type | Transform | Set `tab` type to Text |
| 6 | Appended Query | Combine | Append `orders_feb` |
| 7 | Removed Duplicates | Clean | Remove duplicate order IDs |
| 8 | Reordered Columns | Clean Up | Logical column order |
| 9 | Renamed Columns | Clean Up | `Order_ID` → `Order_Id`, `tab` → `Tab` |

### orders_feb (5 steps)

| # | Step Name | Category | Description |
|---|---|---|---|
| 1 | Source | Connect | Import `orders_feb.csv` |
| 2 | Promoted Headers | Connect | Use first row as headers |
| 3 | Changed Column Type | Transform | Assign data types |
| 4 | Added Custom | Transform | Add `tab` = `"feb"` |
| 5 | Changed Type | Transform | Set `tab` type to Text |

---

## 10. Key Concepts

### Why follow a structured order?

| Step | Purpose | If you skip it... |
|---|---|---|
| **Filter first** | Remove junk before processing | You waste time cleaning rows you'll delete later |
| **Clean before transform** | Standardize values | Splits and extractions fail on inconsistent data |
| **Transform before combine** | Ensure matching schemas | Append/merge will fail or produce misaligned columns |
| **Clean up last** | Polish the output | Your data model has unnecessary columns and poor naming |

### Trim → Lowercase → Capitalize: Why this order?

1. **Trim** removes invisible whitespace that can make `" John"` ≠ `"John"`
2. **Lowercase** creates a uniform base: `"JOHN"`, `"john"`, `"John"` all become `"john"`
3. **Capitalize Each Word** produces the final display format: `"john"` → `"John"`

If you capitalize first and then lowercase, you undo your own work.

### Append vs. Merge

- **Append** = stack rows vertically (same columns, more records) = SQL `UNION`
- **Merge** = join columns horizontally (shared key, more fields) = SQL `JOIN`

---

## 11. Common Pitfalls

| Pitfall | Solution |
|---|---|
| Cleaning data before filtering | Always filter first — fewer rows to process |
| Forgetting to trim before comparing | Whitespace makes `" John"` ≠ `"John"` |
| Splitting columns resets data types | Always re-assign types after a split operation |
| Not adding a source identifier before appending | Add a `Tab`/`Source` column so you know which file each row came from |
| Applying `Capitalize Each Word` to emails | Only capitalize name fields — emails should be lowercased |
| Leaving helper columns in the final model | Always do a final Clean Up step to remove intermediate columns |
