# Dashboard improvement plan: unpaid purchases (`Ärasõidud`)

This document proposes a clearer dashboard structure and a practical metric set for separating **human errors** from **intentional abuse/fraud**.

## 1) Main design goals

1. Help operations quickly answer: **"What needs action today?"**
2. Help analysts answer: **"Where is risk increasing and why?"**
3. Help management answer: **"How much money is at risk and how effective are actions?"**

## 2) Recommended page structure

## Page A — Executive overview (one-screen summary)

Top KPI cards:
- Total unpaid amount
- Unpaid amount % of total sales
- Number of unpaid cases
- Number of unique clients with unpaid cases
- Share flagged as suspicious (`is_malicious`)
- Recovery rate (paid later / initially unpaid)

Visuals:
- Trend line: unpaid amount by week/month
- Stacked column: reason groups (`Tehingu sisu`) split into **Human mistake** vs **Intentional**
- Top 10 units (`Kaup`/`unit_id`) by unpaid amount
- Heatmap: weekday × hour unpaid count (to detect process issues)

UX notes:
- Keep max 6-8 visuals on this page
- Use color semantics consistently:
  - Gray = neutral
  - Amber = process error
  - Red = suspected intentional abuse
  - Green = recovered/solved

## Page B — Root-cause & segmentation

Visuals:
- Decomposition tree for unpaid amount:
  - `Tehingu sisu` → `Lahendus` → unit/client type
- Matrix by `Tehingu sisu` with columns:
  - cases, amount, avg amount/case, suspicious rate, statement coverage
- Client segmentation scatter:
  - X = number of cases
  - Y = unpaid amount
  - Size = average delay to solution
  - Color = suspicious rate

Drillthrough target:
- Click any client/unit to open detail page with transaction history.

## Page C — Case management / operations

Visuals:
- Aging buckets (0-7, 8-30, 31-60, 61-90, 90+ days)
- Open vs solved cases over time
- SLA compliance (solved within target days)
- Table with action priority score and owner

Include action filters:
- "Only no statement"
- "Only no solution"
- "Only high value > threshold"

## Page D — Data quality & controls

Visuals:
- Missing key fields by source/system (card number, client, invoice link)
- Late ingestion checks
- Duplicate transaction checks
- % transactions with unclear reason mapping

This page reduces false positives and builds trust in the dashboard.

## 3) Core measure set to implement

Below are practical DAX-style measures to include in your model (field names may need adaptation):

```DAX
-- Volume & value
Unpaid Cases = COUNTROWS('Query2')

Unpaid Amount = SUM('Query2'[transaction_sum])

Unpaid Qty = SUM('Query2'[qty])

Average Unpaid per Case = DIVIDE([Unpaid Amount], [Unpaid Cases])

-- Coverage / prevalence
Total Sales Amount = SUM('AllSales'[amount])

Unpaid Amount % of Sales = DIVIDE([Unpaid Amount], [Total Sales Amount])

Unique Impacted Clients = DISTINCTCOUNT('Query2'[client_id])

Unique Impacted Units = DISTINCTCOUNT('Query2'[unit_id])

-- Suspicion & fraud-related
Suspicious Cases = CALCULATE([Unpaid Cases], 'Query2'[is_malicious] = TRUE())

Suspicious Amount = CALCULATE([Unpaid Amount], 'Query2'[is_malicious] = TRUE())

Suspicious Case Rate = DIVIDE([Suspicious Cases], [Unpaid Cases])

Suspicious Amount Rate = DIVIDE([Suspicious Amount], [Unpaid Amount])

-- Resolution effectiveness
Resolved Cases = CALCULATE([Unpaid Cases], NOT(ISBLANK('Query2'[moved_to_client])))

Resolution Rate = DIVIDE([Resolved Cases], [Unpaid Cases])

No Statement Cases = CALCULATE([Unpaid Cases], ISBLANK('Query2'[statement_no]))

No Solution Cases = CALCULATE([Unpaid Cases], ISBLANK('Query2'[moved_to_client]))

-- Aging / speed
Average Days to Resolution =
AVERAGEX(
    FILTER('Query2', NOT(ISBLANK('Query2'[moved_to_client]))),
    DATEDIFF('Query2'[sale_date], 'Query2'[moved_to_client], DAY)
)

Open 30+ Days Cases =
CALCULATE(
    [Unpaid Cases],
    FILTER('Query2', DATEDIFF('Query2'[sale_date], TODAY(), DAY) > 30 && ISBLANK('Query2'[moved_to_client]))
)

-- Operational priority
Priority Score =
0.45 * [Normalized Amount] +
0.30 * [Normalized Age] +
0.25 * [Normalized Suspicion]
```

## 4) Needed dimension tables / data model enrichments

Add or validate these dimensions for better slicing and stable reporting:
- Date dimension (single active relationship to transaction date)
- Unit dimension (region, type, ownership)
- Client dimension (segment, risk category, tenure)
- Reason mapping table (`Tehingu sisu` -> high-level reason group)
- Resolution mapping table (`Lahendus` -> solved/unsolved/partial)
- Risk classification table (rule-based + analyst override)

Modeling tips:
- Prefer star schema over many direct joins between fact-like tables.
- Keep a single fact table for unpaid-case grain (one row per case/transaction).
- Add surrogate keys where source IDs are inconsistent.

## 5) Classification logic: human mistake vs intentional

Create a transparent rule set (first version):

Human mistake indicators:
- Very low amount
- First occurrence for client
- Immediate correction/payment
- Clear process reason (e.g., card read error)

Intentional indicators:
- Repeated unpaid behavior by same client/card
- Multiple units impacted in short period
- High value and no contact response
- Linked suspicious flag (`is_malicious` = true)

Output fields:
- `CaseType` = Human / Intentional / Unknown
- `CaseTypeConfidence` (0-100)

This allows analysts to focus on high-confidence intentional cases first.

## 6) Alerting & workflow integration

Set threshold alerts in Power BI service:
- Daily unpaid amount > X
- Suspicious amount rate > Y%
- No-statement cases spike > baseline + 2σ

Workflow actions:
- Auto-assign owner by region/unit
- Create weekly exception export for finance/security
- Track feedback loop: was case confirmed fraud or process mistake?

## 7) Suggested implementation roadmap

Phase 1 (1-2 weeks):
- Define metric dictionary
- Build Executive page + basic trend and top contributors
- Validate totals against source system

Phase 2 (2-3 weeks):
- Add root-cause and case-management pages
- Add aging and SLA measures
- Implement rule-based case classification

Phase 3 (ongoing):
- Tune thresholds and alerts
- Add prediction scoring (optional)
- Introduce monthly model governance review

## 8) Dashboard usability improvements for current screenshot

Specific upgrades based on current layout:
- Replace two similar tables ("Tehinguid" and "Eurod") with a **single matrix + field parameter toggle** for metric (cases/qty/amount/unpaid/statement count).
- Add KPI cards above tables so users see risk level before details.
- Move long detail table to a drillthrough page; keep overview page focused.
- Add tooltips with mini-trend and last action date per row.
- Sort defaults by unpaid amount desc, not quantity.
- Rename labels to business language (e.g., "Unpaid amount" instead of technical terms where possible).

---

If you want, next step can be a concrete **Power BI build checklist** (which visuals to create first, exact measure names, and how to set interactions/bookmarks).
