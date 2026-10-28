# AURUM   MCX Gold Relative Pricing & Arbitrage Analytics Platform

> **Institutional-grade relative-value screener, cost-of-carry fair value engine, calendar spread modeler, and delivery exit guard for Multi Commodity Exchange of India (MCX) Gold derivatives.**

---

##  Product Overview

Gold trading on the Multi Commodity Exchange of India (MCX) spans multiple contract variants **GOLD (Big)**, **GOLDM (Mini)**, **GOLDTEN (Ten)**, **GOLDGUINEA**, and **GOLDPETAL**. Each contract has distinct lot sizes, quotation units, liquidity profiles, and delivery mechanisms:

* `GOLD` & `GOLDM` quote per **10 grams** (1 kg and 100 g lots respectively)
* `GOLDTEN` quotes per **10 grams** (10 g lot)
* `GOLDGUINEA` quotes per **8 grams** (8 g lot)
* `GOLDPETAL` quotes per **1 gram** (1 g lot)

Because of differing quote sizes and liquidity variations across expiries, market mispricings, retail premiums, and calendar spread dislocations frequently emerge. 

**AURUM** solves this fragmentation by:
1. **Normalizing all contracts to a standard ₹ per gram** price ladder.
2. **Computing theoretical fair values** using real-time MCX physical spot prices and annualized cost-of-carry financing rates.
3. **Quantifying execution friction** by calculating round-trip brokerage, exchange fees, sell-side taxes (CTT), and GST against theoretical edge.
4. **Guarding against physical delivery penalties** by tracking trading days remaining before the tender period commences.
5. **Synthesizing automated market briefs and predictive signals** for quantitative traders, hedgers, and bullion desk arbitragers.

---

##  Key Features

* **Normalized Per-Gram Price Ladder**: Seamlessly compare `GOLDM`, `GOLDGUINEA`, `GOLDPETAL`, and `GOLDTEN` side-by-side on an absolute per-gram basis.
* **Cost-of-Carry Fair Value Engine**: Calculates theoretical forward prices from spot gold and annual financing/carry rates:
  $$\text{Fair Value} = \text{Spot} \times \left(1 + \text{Annual Rate} \times \frac{\text{Days to Expiry}}{365}\right)$$
* **Relative-Value Screener & Signal Tagging**:
  * Detects rich and cheap deviations against theoretical fair value.
  * Cross-contract spread analysis with 20-session rolling $Z$-Scores against benchmark `GOLDM`.
  * Automatic classification of term structures (`CONTANGO` vs `BACKWARDATION`).
  * Liquidity warnings (`THIN`) for low-volume contracts and imminent roll notifications (`ROLL`).
* **Daily Market Brief & Quick Prediction Summary**: Built-in market intelligence generating concise session summaries and directional drift indications based on basis deviations.
* **Interactive Pairwise Comparison**: Compare any two contracts across expiries to analyze relative spreads, price ratios, and calendar basis.
* **Futures Term Structure & Curve Visualizer**: Real-time representation of the term structure across all active expiry cycles.
* **Institutional Round-Trip Cost Calculator**: Input custom brokerage rates, exchange turnover charges, sell-side taxes, and GST to verify whether a mispricing is tradable (`COST < GAP`) or eaten by transactional friction (`COST >= GAP`).
* **Exit Guard & Tender Risk Management**: Monitors calendar and trading days to contract expiry, issuing time-sensitive warnings (`EXIT SOON`, `ACT TODAY BY 21:00`, `TENDER OPEN / PAST SAFE EXIT`) before staggered/compulsory delivery windows lock in.
* **Historical Data Replay & Greppable Export**: Replay past Bhavcopy market sessions or export plain-text signals via `/tags.txt` for shell scripts, cron jobs, and terminal workflows.

---

##  Tech Stack

### Frontend
* **Core Framework**: React 18 (`18.3.1`) + React DOM
* **Build Tool & Dev Server**: Vite 6 (`6.0.7`) with `@vitejs/plugin-react`
* **Styling**: Vanilla CSS3 (Custom design system with CSS custom properties, responsive grid/flexbox layouts, smooth transitions, dark/light theme persistence, and pure CSS/SVG 3D gold orb visuals)
* **Visualizations**: Lightweight, responsive SVG sparklines and term structure bar charts
* **HTTP Client**: Native browser `Fetch` API with built-in Vite dev server proxying

### Backend
* **Runtime**: Node.js 20+ (ES Modules)
* **Web Framework**: Express.js (`4.21.2`)
* **Database & ODM**: MongoDB & Mongoose (`8.9.5`)
* **Data Ingestion**: Custom MCX Bhavcopy CSV ingestion parser (supports multi-format headers, lot conversions, and date normalizations)
* **Spot Price Scraper**: Live MCX spot market price integration (`services/spotPrice.js`) with failover handling
* **Configuration & Environment**: `dotenv` (`16.4.7`) and centralized quantitative config (`config.js`)
* **Development**: `nodemon` (`3.1.9`) for hot reloading

---

##  How It Works

```
┌─────────────────────────┐       ┌────────────────────────┐
│  MCX Bhavcopy CSV Files │       │  MCX Live Spot Scraper │
│  (GOLDM, GUINEA, etc.)  │       │  (mcxindia.com feed)   │
└────────────┬────────────┘       └───────────┬────────────┘
             │                                │
             ▼                                ▼
┌──────────────────────────────────────────────────────────┐
│                   AURUM Backend (Express)                │
│                                                          │
│  1. Ingest & Upsert to MongoDB (MarketRow model)         │
│  2. Normalize settle prices to ₹ / gram                  │
│  3. Calculate Days to Expiry (Calendar & Trading Days)   │
│  4. Compute Fair Value: Spot × (1 + r × t / 365)         │
│  5. Calculate Basis % = ((Price/g - Fair) / Fair) × 100  │
│  6. Compute 20-day Rolling Z-Scores (Cross & Basis)      │
│  7. Calculate Brokerage + Exchange + Tax + GST Costs     │
│  8. Evaluate Exit Guard (Tender period countdown)        │
│  9. Generate Tag Matrix & Daily Brief Summary            │
└────────────────────────────┬─────────────────────────────┘
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
┌───────────────────────┐         ┌─────────────────────────┐
│     JSON REST API     │         │ Plain-Text /tags.txt    │
│  (/api/rows, /spot)   │         │ (Greppable CLI Feed)    │
└───────────┬───────────┘         └─────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────┐
│                   AURUM Client (React 18)                │
│                                                          │
│  • Overview & Quick Prediction Summary                   │
│  • Relative-Value Screener (Rank, Search, CSV Export)    │
│  • Pairwise Contract Spread Comparator                   │
│  • Futures Curve & Contango / Backwardation Tracker      │
│  • Real-Time Cost vs Arbitrage Gap Calculator            │
│  • Dataset Audit & Signal Coverage Explorer              │
└──────────────────────────────────────────────────────────┘
```

### 1. Data Ingestion & Standardization
On startup, the server reads historical and session Bhavcopy CSVs from `server/data/` (or synthetic records if starting cold), normalizes dates and symbols, and maps raw settlement values into standard lot specifications:
* $\text{Price per Gram} = \frac{\text{Settle Price}}{\text{Quote Unit in Grams}}$

### 2. Fair Value & Basis Computation
When a spot price is available (either fetched from MCX spot market feeds or overridden via environment/settings), the fair forward price is evaluated based on the financing cost of carry:
* $\text{Basis \%} = \left(\frac{\text{Price per Gram} - \text{Fair Value}}{\text{Fair Value}}\right) \times 100$
* **Tags assigned**:
  * `RICH` if $\text{Basis \%} > +\text{richPct}$ (e.g. $> +0.5\%$)
  * `CHEAP` if $\text{Basis \%} < -\text{cheapPct}$ (e.g. $< -0.5\%$)

### 3. Cross-Contract & Statistical Deviations
For any given session and expiry, contracts are benchmarked against `GOLDM`:
* $\text{Cross Spread} = \text{Price/g}(\text{Symbol}) - \text{Price/g}(\text{GOLDM})$
* A rolling $Z$-score ($z = \frac{x - \mu}{\sigma}$) over a 20-session window identifies statistically stretched spreads:
  * `WIDE` ($z > +2.0$) or `TIGHT` ($z < -2.0$).

### 4. Transaction Cost vs. Edge (`COST>GAP`)
The cost calculator computes two-leg friction:
$$\text{Fees} = 2 \times (\text{Flat Brokerage} + \text{Turnover} \times \text{Exchange \%}) + (\text{Turnover} \times \text{Sell Tax \%})$$
$$\text{Total Cost} = \text{Fees} \times (1 + \text{GST \%})$$
If $\text{Cost per Gram} \ge |\text{Price/g} - \text{Fair Value}|$, AURUM flags `COST>GAP`, preventing false arbitrage entries where transaction friction eliminates the profit margin.

### 5. Exit Guard & Physical Delivery Protection
MCX gold contracts carry physical delivery requirements if held into the tender period (typically 3–5 trading days before expiry). AURUM iterates through valid trading days (excluding weekends) and alerts traders:
* `ACTIVE`: Safe trading window.
* `EXIT SOON`: Less than 5 trading days until tender notice.
* `ACT TODAY BY 21:00`: Final trading day before tender risk begins.
* `TENDER OPEN / PAST SAFE EXIT`: Position is within tender period; risk of delivery allocation and penalty charges.

---

##  MCX Contract Specifications Matrix

| Symbol | Lot Size | Quote Unit | Delivery Mechanism | Default Premium Model |
| :--- | :--- | :--- | :--- | :--- |
| **GOLD** | 1,000 g (1 kg) | 10 g | Staggered Delivery | Benchmark (0.00%) |
| **GOLDM** | 100 g | 10 g | Staggered Delivery | +0.03% |
| **GOLDTEN** | 10 g | 10 g | Staggered Delivery | +0.05% |
| **GOLDGUINEA** | 8 g | 8 g | Staggered Delivery | +0.15% |
| **GOLDPETAL** | 1 g | 1 g | Compulsory Delivery | +0.30% |
| **GOLDPETALM** | 1 g | 1 g | Compulsory Delivery | +0.25% |

---

##  Getting Started

### Prerequisites
* **Node.js**: v20.x or later
* **npm**: v10.x or later
* **MongoDB**: A running local MongoDB instance (`mongodb://127.0.0.1:27017/gold_commodity`) or a MongoDB Atlas connection string.

---

### Installation & Local Setup

#### 1. Clone the repository
```bash
git clone https://github.com/Sumodip-Patra/AURUM.git
cd AURUM
```

#### 2. Backend Setup
```bash
cd server

# Create or verify .env configuration
# (e.g. MONGODB_URI=mongodb://127.0.0.1:27017/gold_commodity)
npm install
npm run dev
```
The server will boot on `http://localhost:5000`, connect to MongoDB, import Bhavcopy data from `server/data/`, and begin listening for API requests.

#### 3. Frontend Setup
In a separate terminal:
```bash
cd client
npm install
npm run dev
```
Open the Vite URL shown in your terminal (typically `http://localhost:5173`). Vite automatically proxies requests to `/api` and `/tags.txt` to the Express backend running on port 5000.

---

##  Environment & Configuration

### Backend Environment Variables (`server/.env`)
```env
PORT=5000
MONGODB_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/gold_commodity
MCX_SPOT_PER_GRAM=               # Optional manual spot override in ₹/gram
MCX_FINANCING_RATE_PCT=6.0       # Annualized risk-free financing rate (default: 6.0%)
MCX_ADDITIONAL_CARRY_PCT=0.0     # Storage, insurance, and carry rate surcharge
```

### Quantitative Model Configuration (`server/config.js`)
Model parameters can be adjusted directly in `server/config.js`:
```javascript
export const CONFIG = {
  dutyPct: 0.06,      // 6% Import Duty assumption
  carryPct: 0.06,     // 6% Annual Carry rate
  richPct: 0.5,       // +0.5% threshold for RICH signal
  cheapPct: 0.5,      // -0.5% threshold for CHEAP signal
  zWindow: 20,        // 20-session rolling window for Z-Scores
  zLimit: 2,          // 2.0 Sigma limit for WIDE/TIGHT tags
  rollDays: 5,        // Threshold days to trigger ROLL flag
  minVolume: 50,      // Minimum lot volume to prevent THIN flag
  broker: {
    flat_brokerage_per_order: 20,
    pct_brokerage: 0,
    exchange_charge_pct: 0.0026,
    sell_side_tax_pct: 0.01,
    gst_on_fees_pct: 18
  },
  exit_guard: {
    tender_period_trading_days: 3,
    warn_days: 5,
    square_off_time: "21:00"
  }
};
```

---

##  API Reference

| Endpoint | Method | Description | Query Parameters |
| :--- | :--- | :--- | :--- |
| `/api/health` | `GET` | Health check endpoint returning `{ status: 'ok' }` | None |
| `/api/sessions` | `GET` | List of all available trading session dates in DB | None |
| `/api/rows` | `GET` | Full relative-value analysis rows for a session | `?date=YYYY-MM-DD` (defaults to latest) |
| `/api/spot` | `GET` | Real-time scraped MCX Gold spot price | None |
| `/api/history` | `GET` | Historical settlement observations for charting | `?symbol=GOLDM` |
| `/tags.txt` | `GET` | Greppable plain-text tag export for terminal workflows | `?date=YYYY-MM-DD` |

### Sample Output: `/tags.txt`
```text
2022-10-27 GOLDM 2022-11-04 [ROLL] [THIN] [EXIT GUARD] 8 days left; Volume 12; EXIT SOON
2022-10-27 GOLDPETAL 2022-11-30 [RICH] [WIDE] +0.82% vs fair; Same-expiry GOLDM deviation z=2.4
```

---

##  Project Structure

```
AURUM/
├── client/                     # React 18 + Vite Frontend
│   ├── index.html              # Single page application entry
│   ├── package.json            # Client dependencies & scripts
│   ├── vite.config.js          # Vite config & API reverse proxy
│   └── src/
│       ├── main.jsx            # Core UI, Screener, Cost Calculator & Brief
│       └── styles.css          # Design system & dark/light theme tokens
├── server/                     # Node.js + Express Backend
│   ├── config.js               # Contract specs, broker fees & quant thresholds
│   ├── package.json            # Server dependencies & scripts
│   ├── .env                    # Database connection & runtime config
│   ├── data/                   # MCX Bhavcopy CSV files (GOLDM, GUINEA, etc.)
│   └── src/
│       ├── index.js            # Express server initialization & MongoDB connection
│       ├── models/
│       │   └── MarketRow.js    # Mongoose schema for market observations
│       ├── routes/
│       │   └── api.js          # REST endpoints (/rows, /sessions, /spot, /history)
│       └── services/
│           ├── marketData.js   # Pricing models, Z-scores, exit guards & tagging
│           └── spotPrice.js    # Live MCX spot market price scraper
└── README.md                   # Project documentation
```

---

##  Disclaimer

AURUM is designed as an analytical research and educational tool for commodity derivatives traders and quantitative researchers. While it calculates real cost-of-carry models, exit guards, and execution fees, live market orders should always be confirmed against official broker execution engines and actual [MCX India](https://www.mcxindia.com) contract circulars.

