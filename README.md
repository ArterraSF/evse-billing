# Arterra HOA — EVSE Infrastructure Billing Engine

A single-page web application that generates accurate, time-of-use (TOU) based electricity bills for each EV charging parking space at Arterra, using raw energy data exported from the building's Emporia Vue energy monitor.

**Live app:** https://arterrasf.github.io/evse-billing/

---

## Background

Arterra's EVSE infrastructure is monitored by an Emporia Vue 3 energy monitor, which tracks per-circuit kWh consumption. The building is billed by PG&E under a commercial Time-of-Use tariff (Schedule B-6 or BEV1), meaning electricity costs more during peak hours (4–9 PM) than overnight.

The previous billing method divided the total bill proportionally by kWh consumed, ignoring *when* energy was used. This unfairly penalised residents who charge overnight at cheap off-peak rates and subsidised those who charge during expensive peak hours.

This app fixes that by multiplying each circuit's hourly consumption by the exact rate that applied at that hour, then scaling every line item proportionally so the sum matches the actual paper utility bill to the penny — including all fixed fees, taxes, and surcharges.

---

## How to Use (Monthly Routine)

### Step 1 — Export data from Emporia

1. Open the Emporia Web App menu at **https://web.emporiaenergy.com/menu**
2. Select **"Export Raw Data to CSV"** and choose the Arterra Emporia device
3. Set the Start and End Date to match the paper billing period exactly
4. Click **"Export Data"** — you will receive an email with a download link to a ZIP file

### Step 2 — Open the app and upload the ZIP

1. Go to **https://arterrasf.github.io/evse-billing/**
2. Drop the downloaded `.zip` file (do not unzip it) into the upload area in Step 1
3. The app automatically extracts the hourly CSV, detects all circuit columns, and calculates TOU costs

### Step 3 — Enter the paper bill total

1. Look at the final **"Total Amount Due"** on the PG&E / CleanPowerSF paper bill
2. Type that exact dollar amount into the **"Total Paper Bill Amount"** field in Step 2
3. The app scales every parking space's bill proportionally so the grand total matches the paper bill exactly

### Step 4 — Select the tariff

Use the **Step 3 dropdown** to select the active billing schedule:
- **Schedule B-6** — General Commercial TOU (current default)
- **Schedule BEV1** — Dedicated Business EV (if the HOA switches in future)

The results table updates instantly when switching between tariffs.

---

## Understanding the Results Table

| Column | Description |
|---|---|
| Circuit / Parking Space | Auto-detected label from the Emporia export |
| Total kWh | Total energy consumed by that circuit during the billing period |
| Raw TOU Cost | Energy-only cost calculated using the seasonal TOU rates |
| Final Billed Amount | Raw TOU cost scaled by the pro-rata multiplier to include taxes and fees |

### The Pro-Rata Multiplier

The badge at the top right of the results table shows a multiplier, for example `x1.29144`.

This number bridges the gap between the raw energy cost and the actual paper bill. It absorbs all fixed charges, customer fees, San Francisco utility taxes, and PG&E surcharges automatically — without needing to track each line item individually.

**Formula:**

```
Multiplier = Total Paper Bill ÷ Sum of All Raw TOU Costs
Final Bill per Space = Raw TOU Cost × Multiplier
```

If a resident asks why their bill is higher than the raw energy cost, this multiplier is the explanation: it represents their proportional share of the building's fixed overhead charges for that month.

---

## Peak Demand Analysis

When a ZIP file is uploaded, the app also analyses the 15-minute interval data included in the Emporia export to determine the peak simultaneous draw across all EVSE circuits during the billing period.

This is relevant for **Schedule BEV1**, which replaces the traditional per-kW demand charge with a fixed monthly subscription in blocks of 10 kW. The peak demand panel shows:

- **Peak simultaneous draw** in kW — the highest 15-minute interval total across all circuits
- **When it occurred** — the exact timestamp of that interval
- **BEV1 blocks needed** — ceiling of peak kW ÷ 10, with the resulting monthly subscription cost
- **Top 10 peak intervals** — to distinguish a one-off spike from a recurring pattern

The subscription charge ($12.41 per 10 kW block per month) is a fixed cost that appears on the paper bill and is automatically absorbed by the pro-rata multiplier when the paper bill total is entered.

---

## Seasonal TOU Windows

The app automatically detects the month from each row's timestamp and applies the correct rate tier. No manual adjustment is needed when seasons change.

### Schedule B-6

| Season | Months | Dates | Peak | Super Off-Peak | Off-Peak |
|---|---|---|---|---|---|
| Summer | June – September | Jun 1 – Sep 30 | 4:00 PM – 9:00 PM | *(none)* | All other hours |
| Spring | March – May | Mar 1 – May 31 | 4:00 PM – 9:00 PM | 9:00 AM – 2:00 PM | All other hours |
| Winter | October – February | Oct 1 – Feb 28/29 | 4:00 PM – 9:00 PM | *(none)* | All other hours |

### Schedule BEV1

BEV1 has **no seasonal variation** — the same rates apply year-round, and Super Off-Peak applies every day (not just spring):

| Period | Hours | Rate type |
|---|---|---|
| Peak | 4:00 PM – 9:00 PM, every day | Higher |
| Off-Peak | 9:00 PM – 9:00 AM and 2:00 PM – 4:00 PM, every day | Lower |
| Super Off-Peak | 9:00 AM – 2:00 PM, every day | Lowest |

These dates and windows are defined by PG&E tariff regulation. The season month arrays are stored in the `"seasons"` block in [`rates.json`](./rates.json) and can be updated there if PG&E ever changes them via a CPUC rate case.

---

## How to Update Tariff Rates

When PG&E or CleanPowerSF publish new rates (typically effective **July 1** each year), edit [`rates.json`](./rates.json) in this repository. No other file needs to change.

Rates are stored as two separate sub-objects under each tariff — `delivery` (PG&E) and `generation` (CleanPowerSF) — because the two utilities publish their rates independently and on different schedules. The app adds them together at runtime. This means when only one changes, you update only that sub-object without having to do any arithmetic.

```json
{
  "B6": {
    "delivery": {
      "_source": "PG&E Electric Delivery Charges — Schedule B-6",
      "summerPeak":         0.64253,
      "summerOffPeak":      0.38491,
      "summerSuperOffPeak": 0.00000,
      "winterPeak":         0.39584,
      "winterOffPeak":      0.35225,
      "winterSuperOffPeak": 0.00000,
      "springSuperOffPeak": 0.31617
    },
    "generation": {
      "_source": "CleanPowerSF Electric Generation Charges — Schedule B-6",
      "summerPeak":         0.17108,
      "summerOffPeak":      0.10711,
      "summerSuperOffPeak": 0.00000,
      "winterPeak":         0.11397,
      "winterOffPeak":      0.09864,
      "winterSuperOffPeak": 0.00000,
      "springSuperOffPeak": 0.08388
    }
  }
}
```

**Where to find updated rates:**
- PG&E Schedule B-6: https://www.pge.com/tariffs/assets/pdf/tariffbook/ELEC_SCHEDS_B-6.pdf
- PG&E Schedule BEV: https://www.pge.com/tariffs/assets/pdf/tariffbook/ELEC_SCHEDS_BEV.pdf
- CleanPowerSF rates: https://cleanpowersf.org/commercial-rate-tables

The rates on your paper bill are the easiest source to verify against — PG&E lists delivery charges and CleanPowerSF lists generation charges on separate pages, matching the two sub-objects in `rates.json` exactly.

Update `_lastUpdated` to the effective date when saving changes. The git commit history serves as an audit trail of every rate change.

### In-app rate override

The **Advanced: Update CleanPowerSF / PG&E Rate Metrics** panel in the app allows temporary in-session overrides without editing `rates.json`. These changes are not saved — they reset on page reload. Use this only for spot-checking; permanent changes must go into `rates.json`.

---

## Technical Notes

### Why are `Mains_A`, `Mains_B`, and `Mains_C` excluded?

The Emporia Vue monitors the two physical mains lines (`Mains_A` and `Mains_B`) that together represent the total building supply. They are not individual circuits and would double-count consumption if included. `Mains_C` is a balance/remainder channel used to account for any energy not captured by individual CTs. All three are excluded automatically — only user-defined circuit columns (the individual EVSE parking spaces) are used in calculations.

### Duplicate circuit names

If two Emporia circuits share the same label, the app merges them into a single row by summing their kWh values. This prevents silent data loss and ensures the total is always correct.

### Data privacy

This app runs entirely in the browser. The Emporia ZIP file is parsed locally in memory using JavaScript — **no data is ever uploaded to a server, sent to GitHub, or stored anywhere outside the browser tab.** The app has no backend. Residents' charging history remains private.

### Files in this repository

| File | Purpose |
|---|---|
| `index.html` | The complete single-page application |
| `rates.json` | TOU rate configuration — edit this annually |
| `tailwind.css` | Compiled Tailwind CSS — rebuild after modifying `index.html` |
| `input.css` | Tailwind CSS input file (local only, not committed) |
| `arterra-logo.jpg` | HOA logo displayed in the app header |
| `arterra-logo-square.jpg` | Square version of the logo for GitHub org profile |
| `favicon.ico` | Browser tab icon |
| `.gitignore` | Prevents accidental commit of `.zip` / `.csv` data exports |

---

## How Residents Can Verify the Calculations

The source code is fully open and auditable. Any resident who wants to verify their bill can do so at any level of technical depth.

### 1. Verify the TOU rate assignments

Open [`rates.json`](./rates.json) in the repository. Rates are stored as two separate sub-objects — `delivery` (PG&E) and `generation` (CleanPowerSF) — which the app adds together at runtime. Cross-check each against your paper bill:
- PG&E delivery charges appear on page 3 of the bill under **"Details of PG&E Electric Delivery Charges"**
- CleanPowerSF generation charges appear on page 4 under **"Details of CleanPowerSF Electric Generation Charges"**

You can also verify against the published tariff sheets:
- [PG&E Schedule B-6 tariff sheet](https://www.pge.com/tariffs/assets/pdf/tariffbook/ELEC_SCHEDS_B-6.pdf)
- [PG&E Schedule BEV tariff sheet](https://www.pge.com/tariffs/assets/pdf/tariffbook/ELEC_SCHEDS_BEV.pdf)
- [CleanPowerSF commercial rate tables](https://cleanpowersf.org/commercial-rate-tables)

### 2. Verify the seasonal hour windows

The hour boundaries are defined in `index.html` and match the PG&E tariff exactly:

| Window | Condition in code | Hours |
|---|---|---|
| Peak (all seasons) | `hour >= 16 && hour < 21` | 4:00 PM – 8:59 PM |
| Super Off-Peak (spring B-6) | `hour >= 9 && hour < 14` | 9:00 AM – 1:59 PM |
| Super Off-Peak (BEV1 year-round) | `hour >= 9 && hour < 14` | 9:00 AM – 1:59 PM |
| Off-Peak | everything else | remaining hours |

Note: spring peak hours on B-6 use the same rate as winter peak — this is correct per the PG&E B-6 tariff, which defines a single non-summer peak rate applying October through May.

### 3. Verify the pro-rata multiplier

The multiplier is simple arithmetic you can check with a calculator:

```
Multiplier = Total Paper Bill ÷ Sum of All Raw TOU Costs
```

The "Raw TOU Cost" column in the results table shows each circuit's energy-only cost before the multiplier is applied. Sum that column, divide the paper bill total by that sum, and you get the multiplier shown in the badge.

### 4. Verify a specific circuit manually

To spot-check a single parking space:

1. Export the hourly CSV from Emporia and open it in Excel or Google Sheets
2. Filter rows to your circuit's column
3. For each row, look up the timestamp hour and month, determine the correct rate from `rates.json`, and multiply kWh × rate
4. Sum all rows — this should match the **Raw TOU Cost** shown in the app
5. Multiply by the pro-rata multiplier — this should match the **Final Billed Amount**

### 5. Audit the source code directly

The entire application is a single file — [`index.html`](./index.html). The billing logic is in the `processCSV()` and `calculateBilling()` functions, and the peak demand logic is in `analysePeakDemand()`. There is no backend, no database, and no external service involved. Everything that happens to your data happens in those functions, in your browser.

---

## Repository

https://github.com/ArterraSF/evse-billing

---

## Development Notes

### Rebuilding the CSS

This project uses the [Tailwind CSS standalone CLI](https://github.com/tailwindlabs/tailwindcss/releases/latest) — no Node.js or npm required.

**First-time setup:** Download the binary for your platform from the [latest Tailwind CSS release](https://github.com/tailwindlabs/tailwindcss/releases/latest) and place it in the repo root:

```bash
# macOS (Apple Silicon)
curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-macos-arm64
chmod +x tailwindcss-macos-arm64

# macOS (Intel)
curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-macos-x64
chmod +x tailwindcss-macos-x64

# Linux
curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-linux-x64
chmod +x tailwindcss-linux-x64

# Windows
curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-windows-x64.exe
```

**Whenever you modify `index.html`**, rebuild `tailwind.css` before committing:

```bash
# macOS (Apple Silicon)
./tailwindcss-macos-arm64 -i input.css -o tailwind.css --minify

# macOS (Intel)
./tailwindcss-macos-x64 -i input.css -o tailwind.css --minify

# Linux
./tailwindcss-linux-x64 -i input.css -o tailwind.css --minify

# Windows
.\tailwindcss-windows-x64.exe -i input.css -o tailwind.css --minify
```

The compiled `tailwind.css` is committed to the repo so GitHub Pages can serve it directly with no build pipeline.
