# Nashville Housing Data — SQL Data Cleaning & Analysis
## Portfolio Project Report

**Author:** Benard Mwinzi  
**Date:** March 2026  
**Tools:** MySQL · Python · Jupyter Notebook  
**Dataset:** Nashville Housing (56,477 records, 19 columns)  
**Domain:** Real Estate Analytics · Data Engineering

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Goals & Objectives](#2-project-goals--objectives)
3. [Dataset Description](#3-dataset-description)
4. [Data Quality Issues Identified](#4-data-quality-issues-identified)
5. [Cleaning Pipeline — Step-by-Step](#5-cleaning-pipeline--step-by-step)
6. [Feature Engineering](#6-feature-engineering)
7. [Post-Clean Exploratory Analysis](#7-post-clean-exploratory-analysis)
8. [Skills Demonstrated](#8-skills-demonstrated)
9. [Key Results & Impact](#9-key-results--impact)
10. [Lessons Learned & Best Practices](#10-lessons-learned--best-practices)

---

## 1. Executive Summary

Raw data is never analysis-ready. In real-world data pipelines, **data cleaning** is an important activity because because it ensures that downstream analytics, dashboards, and models are built on a trustworthy foundation.

This project applies a rigorous, **end-to-end SQL-based data cleaning pipeline** to the Nashville Housing dataset. Nashville Housing dataset is a publicly available real estate registry containing over 56,000 property transactions. Every transformation is documented, validated, and explained in a self-contained Jupyter notebook that runs MySQL queries inline.

The project moves through four phases:

| Phase | Activities |
|-------|-----------|
| **Assess** | Schema review, null audit, value distribution checks |
| **Clean** | Date standardization, null imputation, flag normalization, PII removal |
| **Enrich** | Address decomposition, feature engineering |
| **Validate** | Row count checks, distinct value audits, range verification |

The result is a clean, normalized dataset ready for business intelligence, geospatial analysis, or machine learning.

---

## 2. Project Goals & Objectives

### Primary Goal
The primary goal of this project is to demonstrate professional-grade **SQL data engineering skills** by transforming a real-world messy dataset into a clean, analysis-ready table.

### Specific Objectives

| # | Objective | Business Reason |
|---|-----------|----------------|
| **O1** | Standardize `SaleDate` to ISO 8601 format | Enables reliable date arithmetic, filtering, and time-series aggregations |
| **O2** | Impute missing `PropertyAddress` values | Eliminates null-address rows that would be excluded from location-based analysis |
| **O3** | Decompose `PropertyAddress` into `PropertyStreet` + `PropertyCity` | Enables city-level filtering, aggregation, and geographic joins |
| **O4** | Decompose `OwnerAddress` into `OwnerStreet`, `OwnerCity`, `OwnerState` | Adds state-level geographic dimension; separates owner from property location |
| **O5** | Normalize `SoldAsVacant` to a consistent `Y`/`N` flag | Removes data entry inconsistency; enables clean binary filtering |
| **O6** | Remove Personally Identifiable Information (PII) | Data governance best practice; protects privacy in shared analytics environments |
| **O7** | Engineer a `SaleMonth` feature from `SaleDate` | Enables seasonal analysis without needing date functions in every query |
| **O8** | Validate data integrity after every transformation | Ensures no rows lost, no silent data corruption, no type errors |

### Portfolio Showcase Goals
- Demonstrate comfort with **MySQL DDL and DML** in a real-world context
- Show **systematic thinking** about data quality — not just fixing issues, but understanding *why* they exist
- Illustrate **production-readiness**: every step is validated, not assumed
- Showcase ability to **communicate technical work** through structured documentation

---

## 3. Dataset Description

### Source
The [Nashville Housing dataset](https://www.kaggle.com/datasets/tmthyjames/nashville-housing-data?select=Nashville_housing_data_2013_2016.csv), downloaded from Kaggle is a widely used portfolio dataset containing public housing sales records from Davidson County, Tennessee, USA.

### Schema (Raw)

| Column | Type | Description |
|--------|------|-------------|
| `UniqueID` | INT (PK) | Unique identifier for each transaction |
| `ParcelID` | VARCHAR | Land parcel reference (shared by properties in same lot) |
| `LandUse` | VARCHAR | Property classification (e.g., SINGLE FAMILY, CONDO) |
| `PropertyAddress` | VARCHAR | Full property address as `"<street>, <city>"` |
| `SaleDate` | VARCHAR | Date of sale (was originally `"Month DD, YYYY"` format) |
| `SalePrice` | DECIMAL | Transaction price in USD |
| `LegalReference` | VARCHAR | Legal document reference number |
| `SoldAsVacant` | VARCHAR | Whether property was vacant at sale (`Yes`/`No`/`Y`/`N`) |
| `OwnerName` | VARCHAR | Owner's legal name (PII) |
| `OwnerAddress` | VARCHAR | Owner address as `"<street>, <city>, <state>"` |
| `Acreage` | DECIMAL | Property acreage |
| `TaxDistrict` | VARCHAR | Tax jurisdiction |
| `LandValue` | DECIMAL | Assessed land value (USD) |
| `BuildingValue` | DECIMAL | Assessed building value (USD) |
| `TotalValue` | DECIMAL | Total assessed value (USD) |
| `YearBuilt` | INT | Year the structure was built |
| `Bedrooms` | INT | Number of bedrooms |
| `FullBath` | INT | Number of full bathrooms |
| `HalfBath` | INT | Number of half bathrooms |

### Key Statistics
- **Total rows:** 56,477
- **Date range:** January 2013 – December 2019
- **Geographic scope:** Nashville Metro Area (Davidson County, TN)
- **Price range:** ~$1 to $4M+

---

## 4. Data Quality Issues Identified

A structured audit before any transformation is essential. The following issues were identified:

### Issue 1 — Date Format Inconsistency
- **Column:** `SaleDate`
- **Problem:** The CSV contained dates as `"April 9, 2013"` (natural language format). When loaded into a `DATE` column, MySQL interpreted them, but an explicit re-cast ensures no edge cases.
- **Risk:** Silent type errors in date arithmetic without explicit standardization.

### Issue 2 — Missing Property Addresses
- **Column:** `PropertyAddress`
- **Problem:** 29 rows had blank `PropertyAddress` values.
- **Root Cause:** Data entry gaps in the source system.
- **Key Insight:** Properties on the same parcel (`ParcelID`) share a physical address, so sibling records can be used to impute missing values.

### Issue 3 — Compound Address Columns
- **Columns:** `PropertyAddress`, `OwnerAddress`
- **Problem:** Multiple pieces of information stored in single fields, violating First Normal Form (1NF).
- **Impact:** City-level analysis impossible without parsing; joins to geographic lookup tables fail.

### Issue 4 — Inconsistent Categorical Encoding
- **Column:** `SoldAsVacant`
- **Problem:** Four distinct values observed: `'Yes'` (51,403 rows), `'No'` (4,623), `'Y'` (52), `'N'` (399).
- **Root Cause:** Multiple data entry operators using different conventions over time.
- **Impact:** `GROUP BY SoldAsVacant` produces 4 groups instead of 2; boolean logic breaks.

### Issue 5 — Personally Identifiable Information (PII)
- **Column:** `OwnerName`
- **Problem:** Individual names are PII and should be removed before analytics use cases or sharing.
- **Governance principle:** Retain the minimum data necessary for the analytical objective.

### Issue 6 — Missing Engineered Time Features
- **Column:** `SaleDate` exists, but no `SaleMonth`, `SaleYear`, or `SaleQuarter` fields
- **Impact:** Every monthly analysis requires `EXTRACT()` inline; a pre-computed column improves query performance and readability.

---

## 5. Cleaning Pipeline — Step-by-Step

### Step 1 — Date Standardization

**SQL Technique:** `DATE()` cast + `UPDATE`

```sql
--- Convert SaleDate from 'September 26, 2016' format to proper DATE
execute("UPDATE housing_data SET SaleDate = STR_TO_DATE(SaleDate, '%M %d, %Y');")

--- Now change the column type to DATE
execute("ALTER TABLE housing_data MODIFY COLUMN SaleDate DATE;")
```

**Validation:**
```sql
query("SELECT SaleDate FROM housing_data LIMIT 5;")
```

---

### Step 2 — Property Address Imputation

**SQL Technique:** Self-join on `ParcelID` → `CREATE VIEW` → `INNER JOIN UPDATE`

The logic relies on the fact that properties within the same parcel share a street address. By joining the table to itself on `ParcelID` (excluding same-row matches via `UniqueID <> UniqueID`), we can find a "donor" row that has an address.

```sql
-- Self-join preview
SELECT
    a.UniqueID,
    a.ParcelID,
    a.PropertyAddress          AS current_address,
    b.PropertyAddress          AS donor_address
FROM housing_data a
JOIN housing_data b
    ON  a.ParcelID  = b.ParcelID
    AND a.UniqueID <> b.UniqueID
WHERE a.PropertyAddress = '';

-- Materialise as a view
CREATE VIEW updated_address AS
SELECT a.UniqueID, a.ParcelID,
    IF(a.PropertyAddress = '', b.PropertyAddress, a.PropertyAddress) AS resolved_address
FROM housing_data a
JOIN housing_data b ON a.ParcelID = b.ParcelID AND a.UniqueID <> b.UniqueID
WHERE a.PropertyAddress = '';

-- Apply the imputation
UPDATE housing_data
INNER JOIN updated_address ON housing_data.ParcelID = updated_address.ParcelID
    AND housing_data.PropertyAddress = ''
SET housing_data.PropertyAddress = updated_address.resolved_address;
```

**Validation:**
```sql
-- Must return 0
SELECT COUNT(*) FROM housing_data WHERE PropertyAddress = '';

-- Must return 56,477
SELECT COUNT(*) FROM housing_data;
```

**Outcome:** All 29 missing addresses successfully imputed. Row count unchanged.

---

### Step 3 — Address Column Decomposition

**SQL Technique:** `SUBSTRING_INDEX()` + `TRIM()` + `ALTER TABLE ADD` + `UPDATE` + `DROP COLUMN`

#### PropertyAddress → PropertyStreet + PropertyCity
```sql
-- Add new columns
ALTER TABLE housing_data
ADD PropertyStreet VARCHAR(60),
ADD PropertyCity   VARCHAR(30);

-- Populate via string parsing
UPDATE housing_data
SET PropertyStreet = SUBSTRING_INDEX(PropertyAddress, ',', 1),
    PropertyCity   = TRIM(SUBSTRING_INDEX(PropertyAddress, ',', -1));

-- Remove original compound column
ALTER TABLE housing_data DROP COLUMN PropertyAddress;
```

#### OwnerAddress → OwnerStreet + OwnerCity + OwnerState
```sql
ALTER TABLE housing_data
ADD OwnerStreet VARCHAR(60),
ADD OwnerCity   VARCHAR(30),
ADD OwnerState  VARCHAR(10);

UPDATE housing_data
SET OwnerStreet = SUBSTRING_INDEX(OwnerAddress, ',', 1),
    OwnerCity   = TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(OwnerAddress, ',', -2), ',', 1)),
    OwnerState  = TRIM(SUBSTRING_INDEX(OwnerAddress, ',', -1));

ALTER TABLE housing_data DROP COLUMN OwnerAddress;
```

**Validation:**
```sql
SELECT PropertyStreet, PropertyCity, OwnerCity, OwnerState
FROM housing_data WHERE OwnerStreet <> '' LIMIT 5;
```

**Outcome:** Addresses fully atomized. City-level and state-level analysis is now possible without parsing in every query.

---

### Step 4 — SoldAsVacant Normalization

**SQL Technique:** `CASE WHEN` + `UPDATE`

```sql
-- Audit distinct values
SELECT SoldAsVacant, COUNT(*) AS count
FROM housing_data GROUP BY SoldAsVacant;
-- Returns: 'No' (51403), 'N' (399), 'Y' (52), 'Yes' (4623)

-- Normalize to Y/N
UPDATE housing_data
SET SoldAsVacant = CASE
    WHEN SoldAsVacant IN ('Yes', 'Y') THEN 'Y'
    ELSE 'N'
END;
```

**Validation:**
```sql
SELECT SoldAsVacant, COUNT(*) FROM housing_data GROUP BY SoldAsVacant;
-- Must return exactly 2 rows: 'Y' and 'N'
```

**Outcome:** 4 inconsistent values collapsed to 2 canonical values. All 56,477 rows reclassified.

---

### Step 5 — PII Removal

**SQL Technique:** `ALTER TABLE DROP COLUMN`

```sql
ALTER TABLE housing_data DROP COLUMN OwnerName;
```

**Rationale:** Owner names are personal identifiers with no analytical value for market-level housing analysis. Retaining them unnecessarily increases privacy risk and complicates GDPR/PDPA compliance.

**Outcome:** PII eliminated. The `PropertyCity` and geographic fields are retained for spatial analysis.

---

## 6. Feature Engineering

### SaleMonth Column

**Purpose:** Pre-computing `SaleMonth` as a stored column means analysts do not need to call `EXTRACT()` in every aggregation query. This improves:
- Query performance (avoids repeated function calls on 56K rows)
- Readability of downstream SQL
- Compatibility with BI tools that prefer plain column references

```sql
ALTER TABLE housing_data ADD SaleMonth TINYINT;

UPDATE housing_data
SET SaleMonth = EXTRACT(MONTH FROM SaleDate);
```

**Validation:**
```sql
SELECT SaleMonth, COUNT(*) AS transactions
FROM housing_data GROUP BY SaleMonth ORDER BY SaleMonth;
-- Expect 12 rows (months 1–12)
```

**Outcome:** New `SaleMonth` (TINYINT) column populated for all 56,477 rows.

---

## 7. Post-Clean Exploratory Analysis

Once the data is clean, we can derive meaningful insights:

### Average Sale Price by City
```sql
SELECT PropertyCity, COUNT(*) AS transactions,
       ROUND(AVG(SalePrice), 2) AS avg_price
FROM housing_data
WHERE SalePrice > 0
GROUP BY PropertyCity
ORDER BY avg_price DESC LIMIT 10;
```

### Monthly Transaction Volume
```sql
SELECT SaleMonth, COUNT(*) AS transactions, ROUND(AVG(SalePrice), 2) AS avg_price
FROM housing_data WHERE SalePrice > 0
GROUP BY SaleMonth ORDER BY SaleMonth;
```

### Price by Property Type
```sql
SELECT LandUse, COUNT(*) AS count, ROUND(AVG(SalePrice), 2) AS avg_price
FROM housing_data WHERE SalePrice > 0
GROUP BY LandUse ORDER BY avg_price DESC LIMIT 10;
```

### Year-Over-Year Market Trend
```sql
SELECT YEAR(SaleDate) AS year, COUNT(*) AS transactions,
       ROUND(AVG(SalePrice), 2) AS avg_price, ROUND(SUM(SalePrice), 2) AS total_volume
FROM housing_data WHERE SalePrice > 0
GROUP BY YEAR(SaleDate) ORDER BY year;
```

### Price by Construction Decade
```sql
SELECT FLOOR(YearBuilt / 10) * 10 AS decade_built,
       COUNT(*) AS count, ROUND(AVG(SalePrice), 2) AS avg_price
FROM housing_data WHERE YearBuilt > 0 AND SalePrice > 0
GROUP BY decade_built ORDER BY decade_built;
```

---

## 8. Skills Demonstrated

### SQL Skills

| Category | Specific Skills |
|----------|----------------|
| **DDL** | `CREATE TABLE`, `ALTER TABLE ADD/DROP/RENAME`, `CREATE VIEW`, `DROP TABLE`, `DROP VIEW` |
| **DML** | `UPDATE ... SET`, `UPDATE ... INNER JOIN`, multi-column `SET` in a single statement |
| **Data Types** | `DATE`, `DECIMAL(15,2)`, `TINYINT`, `VARCHAR(n)` — deliberate type selection |
| **String Functions** | `SUBSTRING_INDEX()`, `TRIM()`, `IF()` |
| **Date Functions** | `DATE()`, `EXTRACT(MONTH FROM ...)`, `YEAR()` |
| **Conditional Logic** | `CASE WHEN ... THEN ... ELSE ... END` |
| **Joins** | Self-joins for data imputation, `INNER JOIN UPDATE` |
| **Aggregations** | `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()`, `ROUND()` |
| **Grouping** | `GROUP BY`, `ORDER BY`, `LIMIT` |
| **Integrity Checks** | Row count validation, null/blank detection, distinct value audits |
| **Views** | `CREATE VIEW` for reusable intermediate logic |

### Soft / Process Skills

- **Structured approach:** Every cleaning step follows the pattern: *audit → transform → validate*
- **Documentation:** Each cell is narrated to explain the *why*, not just the *what*
- **Data governance awareness:** PII identification and removal with justification
- **Idempotency:** `DROP TABLE IF EXISTS`, `DROP VIEW IF EXISTS` — scripts can be re-run safely

---

## 9. Key Results & Impact

| Metric | Before | After |
|--------|--------|-------|
| Missing PropertyAddress rows | 29 | 0 ✅ |
| Distinct SoldAsVacant values | 4 | 2 ✅ |
| Address columns (compound) | 2 | 0 (decomposed into 5 atomic columns) ✅ |
| PII columns | 1 (`OwnerName`) | 0 ✅ |
| Engineered features | 0 | 1 (`SaleMonth`) ✅ |
| Total rows (integrity) | 56,477 | 56,477 ✅ (no data lost) |
| Date format | Mixed / informal | ISO 8601 `YYYY-MM-DD` ✅ |

---

## 10. Lessons Learned & Best Practices

### 1. Always audit before you clean
Understanding the nature and scale of data quality issues prevents over-engineering. The 29 missing addresses required a self-join strategy, but 4 inconsistent category values only needed a simple `CASE WHEN`. The right tool depends on the specific problem.

### 2. Validate at every step
Checking row counts, null counts, and value distributions after every `UPDATE` or `ALTER` statement catches silent bugs early. A silent data loss in step 2 will corrupt every result downstream.

### 3. Use views for reusable intermediate logic
Creating a view for the address imputation logic (rather than embedding the self-join in the `UPDATE` directly) makes the code readable, testable, and reusable.

### 4. Separate parsing from the update
Always write a `SELECT` preview of the transformation before applying it with an `UPDATE`. This is the SQL equivalent of running tests before deploying.

### 5. Document *why*, not just *what*
Good code comments explain the business reason for each decision (e.g., "removed PII — not required for market analysis"). This is critical for audits, handoffs, and portfolio presentation.

### 6. Preserve row count integrity
The dataset started with 56,477 rows. Any cleaning operation that changes this number requires explicit justification (duplicates removed, etc.). In this project, we imputed nulls rather than deleting rows — preserving all 56,477 records.
