# PhonePe Data Analysis Dashboard

A Power BI dashboard analyzing ~300,000 simulated PhonePe transactions across 107,658 users, covering transaction volume, value, success/failure rates, service-level breakdowns, and user demographics for the period Jan 2024 – Dec 2024.

> Built in Power BI Desktop. Data modeled with Power Query (M) and DAX on a two-table star schema.

---


## Data Source

| | |
|---|---|
| **Source file** | `Phonepe-Final-Dataset.xlsx` |
| **Sheets** | `All_Users`, `All_Transactions` |
| **Load method** | Power Query (`Excel.Workbook`) |
| **Date range** | 2024-01-01 to 2024-12-30 |

The workbook is not bundled inside the `.pbix`. To make this repo reproducible for anyone who clones it:
1. Place `Phonepe-Final-Dataset.xlsx` in a `/data` folder in this repo (or link to wherever you're hosting it, if it's too large/sensitive for git).
2. In Power BI Desktop: **Transform Data → Data Source Settings → Change Source**, and point both `All_Users` and `All_Transactions` queries to the relative path, e.g. `.\data\Phonepe-Final-Dataset.xlsx`.
3. Refresh.

Without step 2, the queries will still point at whatever machine/path they were last saved on — do this before pushing, not after.

---

## Data Model

Star schema, one active relationship:

```
All_Transactions (fact)  --[User_ID, M:1]-->  All_Users (dimension)
```

Plus a calculated `Date_Table` (built from `MIN`/`MAX` of `All_Transactions[Date]`) used for time intelligence, not physically stored in the source file.

### `All_Transactions` — 300,000 rows

| Column | Type | Notes |
|---|---|---|
| Transaction_ID | Text | Unique key |
| Amount | Decimal | |
| User_ID | Text | FK to All_Users |
| Service | Text | 4 categories: Money Transfer, Recharge Bills, Loans, Insurance |
| Service Type | Text | 16 sub-categories (e.g. To UPI ID, Mobile Recharge, Gold Loan, Mutual Fund) |
| Payment_Status | Text | Successful / Failed / Pending (standardized in Power Query) |
| Reason | Text | Granular outcome: Successful, Server error, Wrong PIN, Insufficient amount, Wrong Info, Bank Denied |
| Date | Date | |

### `All_Users` — 107,658 rows

| Column | Type | Notes |
|---|---|---|
| User_ID | Text | Unique key |
| Name | Text | |
| Age | Whole number | Range 18–60 |
| Join_Date | Date | |
| Age Segment | Text (calculated) | Gen Z ≤26, Millennials ≤42, Gen X ≤58, Boomers >58 |

---

## Power Query Transformations

**All_Users**
- Promote headers, type casting
- Remove duplicate `User_ID`, remove fully blank rows
- Added conditional column `Age Segment` (age-band logic above)

**All_Transactions**
- Promote headers, type casting
- Remove duplicate `Transaction_ID`
- Standardized category labels: `Recharge_Bills` → `Recharge Bills`, `Money_Transfer` → `Money Transfer`
- Collapsed three failure reasons (`Wrong PIN`, `Insufficient amount`, `Server error`) into `Payment_Status = "Pending"` — the original, more granular reason is preserved separately in the `Reason` column

---

## DAX Measures

All measures live in a dedicated `Measures` table.

```dax
Total Transaction Amount = SUM(All_Transactions[Amount])

Total Transactions = COUNT(All_Transactions[Transaction_ID])

Successful Transactions =
CALCULATE([Total Transactions], All_Transactions[Payment_Status] = "Successful")

Success Rate = DIVIDE([Successful Transactions], [Total Transactions])

Total Users = DISTINCTCOUNT(All_Users[User_ID])

Trans Value PM =
CALCULATE([Total Transaction Amount], DATEADD(Date_Table[Date], -1, MONTH))

Trans Value MoM% =
DIVIDE([Total Transaction Amount] - [Trans Value PM], [Trans Value PM], 0)

Total Trans PM =
CALCULATE([Total Transactions], DATEADD(Date_Table[Date], -1, MONTH))

Total Trans MoM% =
DIVIDE([Total Transactions] - [Total Trans PM], [Total Trans PM], 0)
```

---

## Report Pages

**Page 1 — Main Dashboard**
- KPI cards: Total Transaction Amount, Total Transactions, Success Rate, Total Users, Trans Value MoM%, Total Trans MoM%
- Slicers: Month, Payment Status
- Line charts: Transaction Amount & Count trend by month, Success Rate trend by month, Total Users by join year
- Clustered bar chart: Transaction Amount by Service
- Column chart: Transaction Amount by user (Name)
- Donut charts: User count by Age Segment; Transaction Amount split by Weekday vs. Weekend
- Dynamic insight text boxes, e.g. auto-updating "on weekdays the number of transactions are maximum" and a top-service callout that recalculates against filters

**Tooltip pages (2)**
- Custom hover tooltips showing Transaction Amount by Service Type, triggered from the Service bar chart

---

## Tech Stack

- Power BI Desktop
- Power Query (M) — data cleaning & shaping
- DAX — measures & calculated table

---

## Suggested Repo Structure

```
phonepe-data-analysis-dashboard/
├── PhonePe_analysis.pbix
├── data/
│   └── Phonepe-Final-Dataset.xlsx
├── preview.png
└── README.md
```

---

## How to Run

1. Clone this repo.
2. Open `PhonePe_analysis.pbix` in Power BI Desktop.
3. If prompted, update the data source path to point at `./data/Phonepe-Final-Dataset.xlsx`.
4. Click **Refresh** on the Home ribbon.

---

## Known Limitations

- `.pbix` is a binary format — git will not produce meaningful diffs between versions; each commit effectively replaces the whole file.
- The dataset appears synthetic/simulated (round category splits: Money Transfer 150K, Recharge/Loans/Insurance 50K each) — worth stating that explicitly if this goes on a portfolio, so it doesn't get mistaken for real PhonePe transaction data.
