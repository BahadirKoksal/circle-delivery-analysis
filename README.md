# 📦 Circle Delivery Analysis

BigQuery SQL project analyzing parcel delivery performance for **Circle**, a French sportswear brand.

---

## 🎯 Objective

The logistics team at Circle wanted to understand how long it takes to deliver parcels to customers — and whether delivery performance differs by transporter, priority level, or month.

This project answers:
- How long does it take on average to ship and deliver a parcel?
- Is UPS or Colissimo faster?
- Do high-priority parcels actually get delivered faster?
- Is delivery performance improving over time?

---

## 🗂️ Dataset

Two tables from BigQuery (`course15` dataset):

| Table | Description |
|---|---|
| `cc_parcel` | One row per parcel — tracking info, transporter, priority, and key dates |
| `cc_parcel_product` | One row per product per parcel — model name and quantity |

**Key columns in `cc_parcel`:**

| Column | Type | Description |
|---|---|---|
| `parcel_id` | INT | Unique parcel identifier |
| `transporter` | STRING | Shipping carrier (colissimo / ups) |
| `priority` | STRING | Shipment priority (High / Medium / Low) |
| `date_purchase` | STRING → DATE | Order purchase date |
| `date_shipping` | STRING → DATE | Date parcel was handed to carrier |
| `date_delivery` | STRING → DATE | Date parcel was delivered |
| `date_cancelled` | STRING → DATE | Date parcel was cancelled (null if not cancelled) |

---

## 🪜 Project Steps

### 1. Data Exploration
Previewed both tables with `SELECT *` to understand structure and content.

### 2. Data Quality — Primary Key Checks
- Confirmed `parcel_id` is unique in `cc_parcel` ✅
- Confirmed `(parcel_id, model_name)` is the composite key in `cc_parcel_product` ✅

### 3. Status Column (CASE WHEN)
Derived a `status` field from date columns:

| Status | Condition |
|---|---|
| `Cancelled` | `date_cancelled IS NOT NULL` |
| `In Progress` | `date_shipping IS NULL` |
| `In Transit` | `date_delivery IS NULL` |
| `Delivered` | `date_delivery IS NOT NULL` |

### 4. Date Format Fix
Date columns were stored as strings (`"April 6, 2022"`).  
Converted to DATE type using `PARSE_DATE("%B %e, %Y", ...)`.  
Used `SAFE.PARSE_DATE` inside `DATE_DIFF` to handle nulls safely.

> 💡 **Bug found & fixed:** `"%B %e,%Y"` fails — the format requires a space: `"%B %e, %Y"`

### 5. KPI Columns
Three delivery time metrics added (in days):

| Metric | Formula |
|---|---|
| `shipping_time` | `date_shipping - date_purchase` |
| `delivery_time` | `date_delivery - date_shipping` |
| `total_time` | `date_delivery - date_purchase` |

### 6. Aggregate Analysis
Built four summary tables:

| Table | Granularity |
|---|---|
| `cc_parcel_kpi_global` | Overall averages |
| `cc_parcel_kpi_transporter` | Grouped by transporter |
| `cc_parcel_kpi_priority` | Grouped by priority level |
| `cc_parcel_kpi_month` | Grouped by purchase month |

---

## 📊 Key Findings

### Overall Performance
| Metric | Value |
|---|---|
| Total parcels | 49 |
| Avg shipping time | 2.02 days |
| Avg delivery time | 2.06 days |
| Avg total time | **4.15 days** |

### Transporter Comparison
| Transporter | Parcels | Total Time |
|---|---|---|
| Colissimo | 42 | 4.24 days |
| UPS | 7 | **3.6 days** |

→ UPS is **0.64 days faster** on average, but handles far fewer parcels.

### Priority Level
| Priority | Parcels | Shipping Time | Total Time | Ratio (shipping/total) |
|---|---|---|---|---|
| High | 29 | 1.89 days | 4.0 days | 47% |
| Medium | 12 | 1.92 days | 4.0 days | 48% |
| Low | 8 | **2.71 days** | **4.71 days** | **58%** |

→ Low priority parcels are slower, and the bottleneck is the **shipping step**, not delivery.  
→ High and Medium priority show no meaningful difference in total delivery time.

### Monthly Trend (2022)
| Month | Parcels | Shipping Time | Total Time |
|---|---|---|---|
| April | 3 | 2.33 days | 4.0 days |
| May | 1 | 3.0 days | 5.0 days |
| June | 12 | 2.08 days | 4.27 days |
| July | 7 | 2.0 days | 4.2 days |
| August | 15 | 2.0 days | 4.0 days |
| September | 11 | **1.78 days** | — |

→ Shipping time steadily improves from April to September.  
→ September delivery time is null — parcels were still in transit at analysis time.

---

## 🛠️ SQL Techniques Used

| Technique | Purpose |
|---|---|
| `SELECT *` / `LIMIT` | Data exploration |
| `COUNT` / `GROUP BY` / `HAVING` | Primary key validation, aggregation |
| `ORDER BY` | Sorting results |
| `CASE WHEN` | Deriving the status column |
| `CREATE OR REPLACE TABLE` | Saving KPI tables in BigQuery |
| `PARSE_DATE` / `SAFE.PARSE_DATE` | Converting string dates to DATE type |
| `EXTRACT(MONTH FROM ...)` | Extracting month number from a date |
| `DATE_DIFF` | Calculating days between two dates |
| `AVG` / `ROUND` | Averages and rounding |
| `SAFE_DIVIDE` | Division without zero-division errors |

---

## 🗃️ Output Tables

```
course15/
├── cc_parcel_kpi          ← Row-level table with status + KPI columns
├── cc_parcel_kpi_global   ← Single-row overall summary
├── cc_parcel_kpi_transporter ← KPIs by transporter
├── cc_parcel_kpi_priority ← KPIs by priority level
└── cc_parcel_kpi_month    ← Row-level table with month column (for monthly aggregation)
```

---

## 🔧 Tools

- **Google BigQuery** — SQL engine and data warehouse
- **SQL** — All analysis done in standard SQL with BigQuery-specific functions

---


