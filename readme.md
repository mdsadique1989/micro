# BullionVault — Zero-API Micro Gold & Silver Portfolio Tracker

**BullionVault** (Fresh Clean v4) is a single-file, zero-backend, privacy-first web application built to track micro-investments in Digital Gold and Silver (such as PhonePe receipts from **MMTC-PAMP** and **SafeGold**). 

It uses **on-device Optical Character Recognition (Tesseract.js)** combined with a **Mathematical Cross-Verification Engine** and **Cryptographic Transaction Timestamp Decoding** to automatically extract purchase details from screenshots without ever uploading your financial images to an external server.

---

## 🌟 Core Features

### 1. 100% On-Device OCR & Mathematical Self-Healing
* **Browser-Native OCR:** Uses `Tesseract.js v5` WebAssembly workers to read PhonePe Gold/Silver purchase screenshots directly in the client browser.
* **Rupee Symbol (`₹`) Hallucination Correction:** OCR engines frequently misread the Indian Rupee symbol (`₹`) as the digit `2` or `7` (e.g., reading `₹10` as `210` or `₹0.30` GST as `20.30`). BullionVault automatically cross-checks `Metal Value + 3% GST = Total Amount` and `Metal Value / Weight = Rate per Gram` to strip hallucinated leading digits.
* **3-Tier Timestamp Extraction:**
  1. **Receipt Header RegEx:** Extracts time and date directly from the receipt banner (e.g., `05:45 pm on 23 Oct 2026`).
  2. **PhonePe Transaction ID Decoder:** Decodes the exact `YYMMDDHHMMSS` timestamp embedded inside PhonePe's `NB...` or `T...` transaction IDs.
  3. **Android Screenshot Filename Fallback:** Parses `YYYY-MM-DD-HH-MM-SS` from standard mobile screenshot filenames.

### 2. Multi-Layer Duplicate Detection Engine
Prevents double-counting of receipts both across your saved portfolio and within multi-file upload batches using four strict checks:
1. **Cryptographic File Hash (`SHA-256`):** Blocks identical image files even if renamed.
2. **PhonePe Transaction ID Match:** Blocks receipts sharing the same `NB...` or `T...` identifier.
3. **Composite Financial Signature:** Blocks entries matching the exact same `Metal + Date + Time + Weight (±0.00001g) + Total Amount (±₹0.01)`.
4. **Intra-Batch Deduplication:** Tracks hashes and Transaction IDs inside a single multi-file drag-and-drop operation.
* *Override Capability:* Includes a **"Force Allow & Save"** button in case a user intentionally wants to log two identical micro-purchases made within the same minute.

### 3. Zero-API Password Cloud Sync (Hybrid Relay + Blob Storage)
* **No Sign-ups or API Keys:** Syncs between PC and Mobile using only a user-chosen secret password.
* **Deterministic Channel Derivation:** Computes `SHA-256("bullion-vault-salt-v2-" + password)` to derive a unique, unguessable 32-character sync channel.
* **Dual-Layer Cloud Persistence:**
  * **Primary Large-Payload Store:** Automatically provisions a REST storage bin via `jsonblob.com` for large transaction histories.
  * **Instant Pub/Sub Relay:** Broadcasts the bin pointer and full transaction JSON backup over `ntfy.sh` (`bv_sync_bin_<hash>`), guaranteeing zero `404` errors and 100% CORS compatibility across desktop and mobile browsers.

### 4. Financial Analytics & Dip vs. High Detector
* **Weighted Average Buy Rate:** Calculates your true pure-metal cost basis ($\frac{\sum \text{Metal Value}}{\sum \text{Weight}}$).
* **GST Break-Even Rate:** Calculates the exact market rate required to break even after accounting for the 3% GST entry cost ($\frac{\sum \text{Total Paid}}{\sum \text{Weight}}$).
* **Interactive SVG Dip Detector Chart:** Plots every historical purchase against your weighted average rate line. Purchases made below your average are highlighted as **🟢 DIP BUYS**, while purchases above average are flagged as **🔴 HIGH BUYS** with exact percentage differentials.
* **Micro-SIP Accumulation & Rate Forecaster:** Projects future portfolio weight, linear rate trajectory, and estimated valuation based on customizable monthly SIP budgets (`₹50` to `₹5,000/mo`) and time horizons (`1` to `24` months).

### 5. Flexible Unit Formatting (`g`, `mg`, `g + mg`)
* Switch seamlessly between **Grams (`0.0495 g`)**, **Milligrams (`49.50 mg`)**, or **Combined (`0.0495 g (49.5 mg)`)** across all cards, charts, and ledger tables.

---

## 📊 Financial Formulas Used

| Metric | Formula | Purpose |
| :--- | :--- | :--- |
| **Total Weight ($W_{\text{total}}$)** | $\sum_{i=1}^{n} w_i$ | Total gold or silver accumulated in grams. |
| **Weighted Avg Rate ($R_{\text{avg}}$)** | $\frac{\sum \text{MetalValue}_i}{W_{\text{total}}}$ | True pre-tax average purchase rate per gram. |
| **Break-Even Rate ($R_{\text{be}}$)** | $\frac{\sum \text{TotalPaid}_i}{W_{\text{total}}}$ | Market rate required to recover the 3% GST paid on entry. |
| **Current Market Value ($V_{\text{curr}}$)** | $W_{\text{total}} \times R_{\text{live}}$ | Real-time liquidation value at live market price. |
| **Net Profit / Loss ($\text{PnL}$)** | $V_{\text{curr}} - \sum \text{TotalPaid}_i$ | Net unrealized gain or loss after 3% GST. |
| **Dip Differential ($\%$)** | $\left(\frac{R_i - R_{\text{avg}}}{R_{\text{avg}}}\right) \times 100$ | Determines whether a specific purchase lowered or raised your cost basis. |

---

## 🛠️ Technology Stack

* **Frontend Framework:** React 18 (UMD Production Build) + Babel Standalone (In-browser JSX compilation)
* **Styling:** Tailwind CSS CDN (Dark-first responsive UI)
* **OCR Engine:** Tesseract.js v5 (`tesseract.min.js`)
* **Cryptography:** Web Crypto API (`window.crypto.subtle.digest('SHA-256', ...)`)
* **Cloud Sync Transport:** Hybrid `ntfy.sh` JSON stream + `jsonblob.com` REST API
* **Storage:** Browser `localStorage` (`bv_fresh_txs_v4`, `bv_fresh_pwd_v4`) + Full JSON Export/Import

---

## 📁 Documentation Index

1. **[`README.md`](./README.md):** Overview, core capabilities, financial formulas, and tech stack.
2. **[`ARCHITECTURE.md`](./ARCHITECTURE.md):** Deep-dive into every function, regular expression, OCR self-healing heuristic, state hook, and cloud sync protocol.
3. **[`SETUP_GUIDE.md`](./SETUP_GUIDE.md):** Step-by-step instructions for local execution, static hosting, mobile-to-PC cloud syncing, and troubleshooting.