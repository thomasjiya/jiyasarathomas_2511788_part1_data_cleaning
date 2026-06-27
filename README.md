# jiyasarathomas_2511788_part1_data_cleaning
# Part 1: Business Data Cleaning, Validation & Excel Reporting

## Problem Summary

A retail company exported order-level sales data from multiple internal systems. The raw dataset (`raw_orders.xlsx`, 932 records) contained numerous data quality issues including inconsistent text formatting, mixed date formats, duplicate records, missing values, invalid discount values, calculation mismatches, and order status inconsistencies. The goal was to clean, validate, and produce analysis-ready data along with summary reports for business review.

---

## Dataset Description

| Attribute | Detail |
|-----------|--------|
| **File** | `raw_orders.xlsx` |
| **Raw Records** | 932 rows |
| **Cleaned Records** | 900 rows |
| **Columns** | 21 original + 9 calculated |
| **Date Range** | 2024–2025 |
| **Domain** | Retail — orders across multiple regions, categories, and customer segments |

### Columns in Raw Dataset
`order_id`, `order_date`, `ship_date`, `customer_id`, `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `product_name`, `ship_mode`, `quantity`, `unit_price`, `discount`, `sales`, `cost`, `profit`, `payment_status`, `order_status`

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **Python 3** | Core data cleaning and processing |
| **pandas** | Data manipulation and transformation |
| **openpyxl** | Excel file creation and styling |
| **dateutil** | Parsing mixed date formats |
| **matplotlib** | Screenshot / chart generation |
| **numpy** | Numerical computations |

---

## Cleaning Steps Performed

1. **Preserved raw data** — `raw_orders.xlsx` kept unchanged; all work done on a copy
2. **Text standardisation** — TRIM, Title Case, canonical value mapping applied to 10 fields
3. **Date cleaning** — 4 mixed formats unified to `YYYY-MM-DD`; `shipping_delay_days` added
4. **Duplicate removal** — 20 fully identical rows removed; 12 duplicate `order_id` rows resolved
5. **Discount validation** — negative values made positive; `"70%"` strings converted to `0.70`; missing filled with `0`
6. **Missing value handling** — `region` and `ship_mode` filled with `"Unknown"`; `discount` filled with `0`
7. **Calculation validation** — sales and profit rules verified; mismatches flagged
8. **Calculated columns** — 9 new columns added (see Task 6 details below)
9. **Quality flags** — every record assigned `CLEAN`, `WARNING`, or `INVALID`

---

## Business Rules Applied

| # | Rule |
|---|------|
| 1 | `discount` ∈ [0.0, 1.0] — must be decimal |
| 2 | `ship_date` ≥ `order_date` — cannot ship before ordering |
| 3 | `sales = quantity × unit_price × (1 − discount)` |
| 4 | `profit = sales − cost` |
| 5 | No duplicate `order_id` values |
| 6 | All categorical fields must use canonical values |
| 7 | All text fields: Title Case, no extra whitespace |

---

## Summary of Data Quality Issues Found

| Issue | Count | Resolution |
|-------|:---:|------------|
| Missing values (region, ship_mode, discount) | 66 | Filled with defaults |
| Fully duplicate rows | 20 | Removed |
| Duplicate order IDs | 12 | Kept first occurrence |
| Negative discount values | 16 | Converted to absolute value |
| Percentage string discounts | 8 | Converted to decimal |
| Ship date before order date | 92 | Flagged `INVALID` |
| Sales calculation mismatches | 52 | Flagged `WARNING` |
| Profit calculation mismatches | 28 | Flagged `WARNING` |
| Text case / whitespace issues | 247 | Auto-corrected |
| Variant category/segment spellings | 107 | Canonical mapping applied |
| Mixed date formats | All rows | Unified to YYYY-MM-DD |

**Final record quality:**
- ✅ **CLEAN:** 743 records (82.6%) — ready for analysis
- ⚠️ **WARNING:** 65 records (7.2%) — review recommended
- ❌ **INVALID:** 92 records (10.2%) — source correction required

---

## Task 6: Calculated Columns Added

| Column | Formula / Logic |
|--------|----------------|
| `cleaned_discount` | `CLIP(discount, 0, 1)` |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount)` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit ÷ calculated_sales` |
| `shipping_delay_days` | `ship_date − order_date` (days) |
| `order_month` | Month extracted from `order_date` |
| `order_year` | Year extracted from `order_date` |
| `data_quality_flag` | `CLEAN` / `WARNING` / `INVALID` |
| `sales_variance` | `calculated_sales − sales` |

---

## Summary of Final Pivot Reports

### pivot_summary.xlsx — 6 Sheets

| Sheet | Contents |
|-------|---------|
| **1 · By Region** | Sales & profit by region, sorted by Total Sales ↓ |
| **2 · By Category** | Sales & profit by category and sub-category, with subtotals |
| **3 · By Ship Mode** | Order count, avg delay, and total sales by shipping method |
| **4 · By Segment** | Profit margin by customer segment, sorted by margin ↓ |
| **5 · Problem Orders** | Cancelled & returned orders by region with lost sales value |
| **6 · Monthly Trend** | Month-on-month sales, profit, and MoM % change |

---

## Key Business Insights

1. **South region generates the highest sales** — ₹22.1L total, leading all regions
2. **Furniture has the highest profit margin** at 28.9% among all categories
3. **Overall company profit margin is 28.5%** on total sales of ₹88.5L
4. **Corporate segment is most profitable** with a 29.4% margin
5. **Standard Class is the dominant ship mode** — used for 239 orders (26.6%)
6. **33.2% of orders are Cancelled or Returned** — a significant operational concern worth investigating
7. **92 orders (10.2%) have ship dates before order dates** — indicates a systemic data recording issue in the source system
8. **65 orders have sales/profit mismatches** — suggesting manual overrides or formula errors in the source system

---

## Assumptions and Limitations

### Assumptions
- Missing discounts assumed to be `0` (no discount applied)
- Missing `region` / `ship_mode` filled with `"Unknown"` (no reference data available)
- Negative discounts are sign-entry errors; converted to positive
- `dateutil` parses ambiguous dates with `dayfirst=False` (US format assumed)
- First occurrence kept for duplicate `order_id` records

### Limitations
- Ship-before-order records flagged but not corrected (cannot determine which date is correct)
- No product master list to validate `product_name` / `sub_category` values
- No customer master list to detect customer record mismatches
- No historical baseline for outlier detection
- Date format ambiguity (e.g. `06/08/2024`) cannot be resolved without source system metadata

---

## Screenshots Included

| File | Shows |
|------|-------|
| `screenshots/raw_data_preview.png` | Raw dataset — first 10 rows with visible issues |
| `screenshots/cleaned_data_preview.png` | Cleaned dataset with all Task 6 calculated columns |
| `screenshots/pivot_summary_1.png` | Sales/profit by region and category (4-panel chart) |
| `screenshots/pivot_summary_2.png` | Monthly trend, ship mode orders, and problem orders by region |

---