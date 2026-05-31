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

## Seasonal TOU Windows

The app automatically detects the month from each row's timestamp and applies the correct rate tier. No manual adjustment is needed when seasons change.

| Season | Months | Peak | Super Off-Peak | Off-Peak |
|---|---|---|---|---|
| Summer | June – September | 4:00 PM – 9:00 PM | *(none)* | All other hours |
| Spring | March – May | 4:00 PM – 9:00 PM | 9:00 AM – 2:00 PM | All other hours |
| Winter | October – February | 4:00 PM – 9:00 PM | *(none)* | All other hours |

---

## How to Update Tariff Rates

When PG&E or CleanPowerSF publish new rates (typically effective **July 1** each year), edit [`rates.json`](./rates.json) in this repository. No other file needs to change.

```json
{
  "_lastUpdated": "2026-07-01",
  "B6": {
    "summerPeak":         0.50981,
    "summerOffPeak":      0.45089,
    "winterPeak":         0.49700,
    "winterOffPeak":      0.43700,
    "springSuperOffPeak": 0.40005
  },
  "BEV1": {
    "summerPeak":         0.49500,
    ...
  }
}
```

All values are the **combined blended rate** in dollars per kWh — PG&E delivery charges plus CleanPowerSF generation charges added together.

**Where to find updated rates:**
- PG&E Schedule B-6: https://www.pge.com/tariffs/assets/pdf/tariffbook/ELEC_SCHEDS_B-6.pdf
- CleanPowerSF rates: https://www.cleanpowersf.org/rates

Update `_lastUpdated` to the effective date when saving changes. The git commit history serves as an audit trail of every rate change.

### In-app rate override

The **Advanced: Update CleanPowerSF / PG&E Rate Metrics** panel in the app allows temporary in-session overrides without editing `rates.json`. These changes are not saved — they reset on page reload. Use this only for spot-checking; permanent changes must go into `rates.json`.

---

## Technical Notes

### Why is `Mains_C` excluded?

The Emporia Vue uses a `Mains_C` channel as a balance/remainder circuit to account for any energy not captured by individual CTs. It does not represent a specific parking space and is excluded from all calculations automatically.

### Data privacy

This app runs entirely in the browser. The Emporia ZIP file is parsed locally in memory using JavaScript — **no data is ever uploaded to a server, sent to GitHub, or stored anywhere outside the browser tab.** The app has no backend. Residents' charging history remains private.

### Files in this repository

| File | Purpose |
|---|---|
| `index.html` | The complete single-page application |
| `rates.json` | TOU rate configuration — edit this annually |
| `arterra-logo.jpg` | HOA logo displayed in the app header |
| `.gitignore` | Prevents accidental commit of `.zip` / `.csv` data exports |

---

## Repository

https://github.com/ArterraSF/evse-billing
