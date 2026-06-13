# Solis Solar Dashboard — n8n Workflow

A self-hosted, real-time battery dashboard for **Solis / SolisCloud** solar inverters, built as an n8n workflow. No extra servers or databases — it runs entirely inside n8n and serves a live HTML page on demand.

---

## Features

- **Live SoC** — battery state-of-charge with color-coded progress bar (green → amber → red)
- **Time-to-empty** — how long until the battery hits your reserve floor, based on current discharge rate
- **Time-to-full** — how long until the battery is fully charged, based on current charge rate
- **ETA clock** — exact local time the battery will reach empty or full (e.g. `~11:30 PM`)
- **SoC sparkline** — SVG line chart of the last 24 h of battery data per inverter
- **Solar / Load** — live solar output and computed household load
- **Online / Offline / Alarm** status badge per inverter
- **Auto-refresh every 5 min** with countdown timer — matches SolisCloud's data update rate
- **Hard refresh** — bypass browser cache on every reload (button and auto-refresh both use a cache-busting URL)
- Supports **multiple inverters** in a single dashboard

---

## Files

| File | Purpose |
|------|---------|
| `solis-dashboard.template.json` | Dashboard workflow — **import this** (no credentials inside) |
| `solis-n8n.json` | Alert workflow — Pushover push notifications on low-battery events |

---

## Prerequisites

- **n8n** (self-hosted, any version that supports webhook + code + HTTP Request nodes)
- A **SolisCloud account** with API access enabled
- Your inverter **ID** and **SN** (serial number) from the SolisCloud portal

---

## Setup

### 1 — Get your SolisCloud API credentials

1. Log in to [soliscloud.com](https://www.soliscloud.com)
2. Go to **Account → Basic Settings → API Management**
3. Generate or copy your **Key ID** and **Key Secret**

### 2 — Find your inverter IDs and serial numbers

Each inverter has two identifiers you need:

- **ID** — a long numeric string (e.g. `1308675217950238097`)
- **SN** — the hardware serial number (e.g. `1031730257030353`)

Find them in SolisCloud under **Plant → Inverter → Device Details**, or call the `/v1/api/inverterList` endpoint manually to see all your inverters in one response.

### 3 — Import the workflow into n8n

1. In n8n, click **+ New Workflow → Import from file**
2. Select `solis-dashboard.template.json`
3. Open the **Build Auth** node and fill in the config block at the top:

```js
const KEY_ID     = 'YOUR_SOLIS_KEY_ID';
const KEY_SECRET = 'YOUR_SOLIS_KEY_SECRET';
const INVERTERS = [
  { id: 'YOUR_INVERTER_1_ID', sn: 'YOUR_INVERTER_1_SN' },
  { id: 'YOUR_INVERTER_2_ID', sn: 'YOUR_INVERTER_2_SN' },
  // add more if you have more inverters
];
```

4. Open the **Process + Render** node and update the capacity config block:

```js
const RESERVE_SOC = 20;  // % floor — match your inverter's discharge reserve setting
const CAPACITY_KWH = {
  'YOUR_INVERTER_1_ID': 10.24,  // total usable kWh for this inverter's battery
  'YOUR_INVERTER_2_ID': 15.36,
};
const INVERTER_ORDER = [
  'YOUR_INVERTER_1_ID',
  'YOUR_INVERTER_2_ID',
];
```

> **Tip — battery capacity:** Dyness LV modules are 5.12 kWh each. Multiply by the number of parallel modules (`bmsPNum` in the API). Other battery brands will have their own specs.

5. **Activate** the workflow and open the webhook URL in your browser:
   ```
   https://YOUR_N8N_HOST/webhook/solis-dashboard
   ```

---

## How it works

```
[Webhook GET /solis-dashboard]
  → [Build Auth]        — signs requests with HMAC-SHA1 for each inverter
  → [Fetch All]         — one HTTP call to /inverterList + one per inverter to /inverterDay
  → [Process + Render]  — computes load, TTE, TTF, ETAs; renders HTML with SVG sparklines
  → [Return Dashboard]  — responds with text/html + no-cache headers
```

All logic lives in the two Code nodes — no external dependencies, no npm packages beyond Node's built-in `crypto`.

### Energy balance (load calculation)

SolisCloud's `inverterList` endpoint does not return `familyLoadPower`, so load is estimated:

```
load = max(0, solar_kW − battery_kW − grid_kW)
```

### Time estimates

| Mode | Formula |
|------|---------|
| Discharging | `(soc − reserve) / 100 × capacityKwh ÷ dischargekW` |
| Charging | `(100 − soc) / 100 × capacityKwh ÷ chargeKw` |

Estimates assume the current power rate stays constant. They update every 5 minutes with the next data refresh.

---

## SolisCloud API notes

- **Base URL:** `https://www.soliscloud.com:13333`
- **Auth:** HMAC-SHA1 signature per request — see Build Auth node for the exact algorithm
- **Rate limit:** 2 calls/sec; this workflow makes `1 + N` calls per page load (N = number of inverters)
- **Data refresh rate:** every 5 minutes

Key fields used from `inverterList`:

| Field | Meaning |
|-------|---------|
| `batteryCapacitySoc` | SoC % (0–100) |
| `batteryPower` | kW — negative = discharging, positive = charging |
| `pac` | Solar output (kW) |
| `psum` | Grid import/export (kW) |
| `state` | 1 = online, 2 = offline, 3 = alarm |

> **Note:** `batteryPower` in `inverterDay` (used for sparklines) is in **watts**, not kW.

---

## Adding a third inverter

1. Add an entry to `INVERTERS` in **Build Auth**
2. Add the same ID to `CAPACITY_KWH` and `INVERTER_ORDER` in **Process + Render**

The workflow is designed so that adding inverters here is the only change needed.

---

## License

MIT — use freely, no attribution required.
