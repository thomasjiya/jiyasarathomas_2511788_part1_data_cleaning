# Cleaning Log — raw_orders.xlsx
**Project:** Business Data Cleaning, Validation & Excel Reporting  
**Dataset:** raw_orders.xlsx  
**Raw Records:** 932 | **Cleaned Records:** 900 | **Removed:** 32  
**Date Processed:** 2025-06-24  

## 1. Issues Identified

### 1.1 Missing Values
| Field | Missing Count | Type |
|-------|:---:|------|
| `region` | 26 | Categorical — region of sale |
| `ship_mode` | 22 | Categorical — shipping method |
| `discount` | 18 | Numeric — applied discount |
| All other columns | 0 | No missing values |

**Total missing cells in raw dataset: 66**

---

### 1.2 Duplicate Records
| Check | Count |
|-------|:---:|
| Fully identical duplicate rows | 20 |
| Rows with duplicate `order_id` (different data) | 12 |
| **Total rows removed** | **32** |

---

### 1.3 Inconsistent Text Formatting
Found across 10 text fields:
- **Extra spaces:** leading, trailing, and multiple internal spaces
- **Case inconsistencies:** mixed UPPER, lower, Title, and ALLCAPS
- **Variant spellings:** same value written differently across rows

| Field | Examples of Issues Found |
|-------|--------------------------|
| `customer_name` | `"  john smith"`, `"JOHN SMITH"`, `"John  Smith"` |
| `segment` | `"SMALL BUSINESS"`, `"Small  Business"`, `"Smallbusiness"` |
| `region` | `" West "`, `"WEST"`, `"west"` |
| `category` | `"OFFICE SUPPLIES"`, `"Office  Supplies"`, `"Officesupplies"` |
| `ship_mode` | `"STANDARD CLASS"`, `"Standard  Class"`, `"Standardclass"` |
| `order_status` | `"COMPLETED"`, `"completed"`, `"  Completed "` |

---

### 1.4 Date Format Inconsistencies
Four distinct date formats found in `order_date` and `ship_date`:

| Format | Example |
|--------|---------|
| `DD Mon YYYY` | `21 Jul 2024` |
| `MM/DD/YYYY` | `08/31/2024` |
| `DD-MM-YYYY` | `28-11-2024` |
| `YYYY-MM-DD` | `2024-05-24` |

**92 records** have `ship_date` earlier than `order_date` — a critical business logic violation.

---

### 1.5 Invalid Discount Values
| Issue | Count | Example |
|-------|:---:|---------|
| Missing / NaN | 18 | `NaN` |
| Negative value | 16 | `-0.19` |
| Percentage string | 8 | `"70%"` |
| **Total issues** | **42** | — |

---

### 1.6 Calculation Mismatches
| Rule | Violations Found |
|------|:---:|
| `sales ≠ quantity × unit_price × (1 − discount)` | 52 |
| `profit ≠ sales − cost` | 28 |
| Combined (either wrong) | 65 |

---

## 2. Cleaning Actions Performed

### Task 1 — Preserve Raw Data
- Original `raw_orders.xlsx` kept entirely unchanged
- All cleaning performed on a separate copy → `cleaned_orders.xlsx`

### Task 2 — Text Field Standardisation
Applied to: `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`

Steps taken:
1. `str.strip()` — removed leading and trailing whitespace
2. `str.replace(r'\s+', ' ')` — collapsed multiple internal spaces to single space
3. `str.title()` — normalised to Title Case
4. Canonical value mapping — corrected variant spellings to standard values:
   - `"SMALL BUSINESS"`, `"Smallbusiness"` → `"Small Business"`
   - `"OFFICE SUPPLIES"`, `"Officesupplies"` → `"Office Supplies"`
   - `"STANDARD CLASS"`, `"Standardclass"` → `"Standard Class"`
   - `"COMPLETED"`, `"completed"`, `"  Completed "` → `"Completed"`

### Task 3 — Date Cleaning
- Parsed all 4 date formats using Python `dateutil.parser`
- Unified all dates to `YYYY-MM-DD` ISO format
- Calculated `shipping_delay_days = ship_date − order_date`
- Added `date_flag` column with value `SHIP_BEFORE_ORDER` for 92 affected rows

### Task 4 — Duplicate Removal
- Removed 20 fully identical duplicate rows using `df.drop_duplicates()`
- For duplicate `order_id` rows with differing values: kept first occurrence, removed rest
- Total rows removed: **32**

### Task 5 — Discount Validation
- Missing discounts (`NaN`): filled with `0` (no discount applied)
- Negative discounts: converted to absolute value using `abs()`
- Percentage strings (e.g. `"70%"`): stripped `%`, divided by 100
- Result stored in `cleaned_discount` column (original `discount` preserved)

### Task 6 — Calculated Columns Added
| Column | Formula |
|--------|---------|
| `cleaned_discount` | `CLIP(discount, 0, 1)` |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount)` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit ÷ calculated_sales` |
| `shipping_delay_days` | `ship_date − order_date` (days) |
| `order_month` | `MONTH(order_date)` |
| `order_year` | `YEAR(order_date)` |
| `data_quality_flag` | `CLEAN` / `WARNING` / `INVALID` |
| `sales_variance` | `calculated_sales − sales` |

### Missing Value Handling
- `region`: 26 missing → filled with `"Unknown"`
- `ship_mode`: 22 missing → filled with `"Unknown"`
- `discount`: 18 missing → filled with `0`

---

## 3. Business Rules Applied

| # | Rule | Field(s) | Violation Action |
|---|------|----------|-----------------|
| 1 | `discount` must be decimal in range [0.0, 1.0] | `discount` | Fix and store in `cleaned_discount` |
| 2 | `ship_date` must not precede `order_date` | `ship_date`, `order_date` | Flag as `INVALID` in `date_flag` |
| 3 | `sales = quantity × unit_price × (1 − discount)` | `sales` | Flag in `sales_mismatch`; add `expected_sales` |
| 4 | `profit = sales − cost` | `profit` | Flag in `profit_mismatch`; add `expected_profit` |
| 5 | No duplicate `order_id` values | `order_id` | Remove duplicates, keep first |
| 6 | All text fields Title Case, no extra whitespace | All text cols | Auto-corrected |
| 7 | Categorical fields must use canonical values | `segment`, `category`, `ship_mode`, `order_status` | Mapped to standard values |

---

## 4. Assumptions Made

1. **Missing `discount`** assumed to be `0` (no discount) rather than unknown — consistent with retail convention.
2. **Missing `region` and `ship_mode`** filled with `"Unknown"` since no lookup table was available to impute correct values.
3. **Negative discounts** assumed to be data-entry sign errors — converted to absolute value.
4. **Percentage discounts** (e.g. `"70%"`) assumed to mean 70% off → converted to `0.70`.
5. **Duplicate `order_id` with different data** — first occurrence kept as authoritative; business team should investigate source system.
6. **Mixed date formats** assumed to be from multiple source systems — all parsed and unified.
7. **Ship-before-order records (92)** are flagged `INVALID` rather than deleted — deletion would be a business decision, not a data-cleaning decision.
8. **Sales/profit mismatches** are flagged rather than auto-corrected — the original values are preserved and calculated values provided for reference.

---

## 5. Records Removed

| Removal Reason | Count |
|----------------|:---:|
| Fully identical duplicate rows | 20 |
| Duplicate `order_id` rows (kept first, removed rest) | 12 |
| **Total removed** | **32** |

No records were removed for missing values, date issues, or calculation mismatches — these were flagged instead.

---

## 6. Records Flagged

| Flag | Count | Reason |
|------|:---:|--------|
| `data_quality_flag = INVALID` | 92 | Ship date before order date |
| `data_quality_flag = WARNING` | 65 | Sales/profit mismatch or discount > 50% |
| `data_quality_flag = CLEAN` | 743 | No issues detected |
| `sales_mismatch = YES` | 52 | Calculated sales ≠ recorded sales |
| `profit_mismatch = YES` | 28 | Calculated profit ≠ recorded profit |
| `date_flag = SHIP_BEFORE_ORDER` | 92 | ship_date < order_date |

---

## 7. Limitations of the Cleaning Process

1. **Ship-before-order records not corrected** — 92 records have this issue. Without access to the source system, it is not possible to determine which date is correct. Flagged only.
2. **No product master list** — `product_name` and `sub_category` were not validated against a reference list; only formatting was cleaned.
3. **No customer master list** — `customer_name` and `customer_id` mismatches could not be detected.
4. **Profit margin not validated against cost structure** — only mathematical consistency (profit = sales − cost) was checked.
5. **Regional mapping not validated** — `state` → `region` mapping was not verified against a geographic reference.
6. **Near-duplicate detection limited to `order_id`** — fuzzy matching on customer + product + date was not performed.
7. **No historical baselines** — outlier detection (e.g. unusually high sales or discounts) could not be performed without historical benchmarks.
8. **Date ambiguity** — formats like `06/08/2024` could be June 8 or August 6. `dateutil` was used with `dayfirst=False` (US convention); some edge cases may be misinterpreted.

---