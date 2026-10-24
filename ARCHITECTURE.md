# BullionVault — System Architecture & Function-by-Function Reference

This document provides an exhaustive technical breakdown of **BullionVault (`index.html`)**, detailing its initialization lifecycle, cryptographic utilities, OCR parsing pipeline, mathematical self-healing heuristics, cloud synchronization protocol, and React state architecture.

---

## 1. High-Level System Architecture

BullionVault is architected as a **Zero-Dependency Single-File Application (SFA)**. All HTML, CSS, React components, OCR workers, and cloud synchronization logic reside in `index.html`.

```
+-------------------------------------------------------------------------+
|                           USER BROWSER (CLIENT)                         |
|                                                                         |
|  +-------------------+     +-----------------------------------------+  |
|  | Screenshot Upload | --> | computeFileHash() [Web Crypto SHA-256]  |  |
|  +-------------------+     +--------------------+--------------------+  |
|                                                 |                       |
|                                                 v                       |
|                            +-----------------------------------------+  |
|                            | Tesseract.js v5 Worker (WASM OCR)       |  |
|                            +--------------------+--------------------+  |
|                                                 | Raw OCR String        |
|                                                 v                       |
|  +-------------------------------------------------------------------+  |
|  |                parsePhonePeOCRText(rawText, fileName)             |  |
|  |  1. Metal & Provider Detection (Gold/Silver, MMTC-PAMP/SafeGold)  |  |
|  |  2. PhonePe ID Extraction & Timestamp Decoding (NB/T + YYMMDD...) |  |
|  |  3. Rupee Symbol ('₹' -> '2'/'7') Math Self-Healing Engine        |  |
|  |  4. GST (3%) & Rate/Gram Cross-Verification                       |  |
|  +----------------------------------------------+--------------------+  |
|                                                 |                       |
|                                                 v                       |
|                            +-----------------------------------------+  |
|                            | checkDuplicate() [Hash + TxID + Sig]    |  |
|                            +--------------------+--------------------+  |
|                                                 |                       |
|                                                 v                       |
|  +-------------------------------------------------------------------+  |
|  |               React State (transactions) <-> localStorage         |  |
|  +---------------------+------------------------+--------------------+  |
+------------------------|------------------------|-----------------------+
                         |                        |
            Push / Pull  |                        | Large Payload Backup
                         v                        v
         +-------------------------------+  +-------------------------+
         | ntfy.sh Deterministic Channel |  | jsonblob.com REST Bin   |
         | Topic: bv_sync_bin_<SHA256>   |  | Auto-Provisioned PUT/GET|
         +-------------------------------+  +-------------------------+
```

---

## 2. Legacy Storage Migration & Constants

At application boot (before React mounts), BullionVault executes a one-time storage sanitization routine to clear legacy keys from older prototypes that previously caused false-positive SHA-256 duplicate blocks:

* **`CLEANUP_FLAG` (`"bv_legacy_cleaned_v4"`):** Checks `localStorage`. If absent, iterates over 14 legacy key names (`bullion_vault_txs_v2`, `bv_hashes_v2`, etc.) and removes them via `localStorage.removeItem(k)`.
* **`STORAGE_TX_KEY` (`"bv_fresh_txs_v4"`):** Primary `localStorage` key storing the serialized JSON array of verified transactions.
* **`STORAGE_PWD_KEY` (`"bv_fresh_pwd_v4"`):** Stores the user's plain-text sync password locally so they do not need to re-type it on every refresh.
* **`CLOUD_SYNC_RELAY` (`"https://ntfy.sh"`):** Base URL for the serverless pub/sub JSON relay.

---

## 3. Core Utility & Cryptographic Functions

### 3.1 `computeFileHash(file)`
* **Purpose:** Generates a unique cryptographic fingerprint of an uploaded screenshot image file to prevent duplicate uploads of the exact same image.
* **Mechanism:**
  1. Reads the `File` object into an `ArrayBuffer` using `await file.arrayBuffer()`.
  2. Passes the buffer to `crypto.subtle.digest("SHA-256", buffer)`.
  3. Converts the resulting `ArrayBuffer` into a `Uint8Array`, maps each byte to a 2-character zero-padded hexadecimal string, and returns the 64-character hex hash.

### 3.2 `decodePhonePeTxTimestamp(txId)`
* **Purpose:** Extracts the exact purchase date and minute-level timestamp embedded inside PhonePe's proprietary transaction IDs, ensuring 100% timestamp accuracy even if the top of the receipt screenshot is cropped.
* **PhonePe ID Format:** PhonePe digital bullion IDs begin with `NB` or `T` followed by `YYMMDDHHMMSS` and random trailing digits (e.g., `NB2610231745489182736451`).
* **RegEx Pattern:** `/^(?:NB|T)(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})\d+$/`
* **Capture Groups:**
  * `m[1]`: 2-digit Year (`26` $\rightarrow$ `2000 + 26 = 2026`)
  * `m[2]`: 2-digit Month (`01`–`12`)
  * `m[3]`: 2-digit Day (`01`–`31`)
  * `m[4]`: 2-digit 24-Hour (`00`–`23`)
  * `m[5]`: 2-digit Minute (`00`–`59`)
  * `m[6]`: 2-digit Second
* **Validation & Output:** Verifies bounds (`month` 1–12, `day` 1–31, `hour24` 0–23, `minute` 0–59). Converts 24-hour time into 12-hour `hh:mm am/pm` format and returns `{ date: "YYYY-MM-DD", time: "hh:mm am/pm" }`.

### 3.3 `decodeFilenameTimestamp(fileName)`
* **Purpose:** Secondary fallback timestamp extractor when an image lacks a readable header date or `NB` transaction ID.
* **RegEx Pattern:** `/(\d{4})-(\d{2})-(\d{2})-(\d{2})-(\d{2})-(\d{2})/` (matches standard Android screenshot naming conventions like `Screenshot_2026-10-23-17-45-48.png`).
* **Output:** Returns `{ date: "YYYY-MM-DD", time: "hh:mm am/pm" }` or `null`.

### 3.4 `to24HourTime(timeStr)`
* **Purpose:** Normalizes 12-hour time strings (e.g., `"05:45 pm"`) into 24-hour ISO-sortable strings (`"17:45"`) so chronological sorting across the SVG chart and ledger works accurately within the same day.

### 3.5 `repairSavedTimestamps(txList)`
* **Purpose:** Self-heals existing or imported transaction lists where the time defaulted to `"12:00 pm"`.
* **Mechanism:** Iterates over `txList`, runs `decodePhonePeTxTimestamp(tx.transactionId)`, and overwrites `date` and `time` if a valid embedded timestamp is discovered.

---

## 4. The OCR & Mathematical Self-Healing Engine

### `parsePhonePeOCRText(rawText, fileName)`
This is the core intelligence function of BullionVault. It transforms noisy, unstructured text output from `Tesseract.js` into a mathematically verified financial record.

#### Step 1: Metal & Vault Provider Classification
* **Metal Detection:**
  * Checks `/Gold\s*Purchase|Rate\s*of\s*gold|Gold\s*Weight|Gold\s*value/i` $\rightarrow$ `"Gold"`
  * Checks `/Silver\s*Purchase|Rate\s*of\s*silver|Silver\s*Weight|Silver\s*value/i` $\rightarrow$ `"Silver"` (defaults to `"Silver"` if ambiguous).
* **Provider Detection:**
  * Checks `/MMTC[\s-]*PAMP/i` $\rightarrow$ `"MMTC-PAMP"`, otherwise `"SafeGold"`.

#### Step 2: OCR-Tolerant Transaction ID Extraction
* Tesseract frequently inserts spaces after `NB` or misreads the number `0` as the capital letter `O` inside long numeric strings.
* **Sanitization Pipeline:**
  ```javascript
  const cleanedForId = text
    .replace(/NB\s+/gi, "NB")
    .replace(/NB([0-9O]{18,28})/gi, (full) => full.replace(/O/gi, "0"));
  ```
* Extracts match via `/\b(NB\d{18,28}|T\d{18,28})\b/i`.

#### Step 3: Multi-Source Date & Time Resolution
1. Initializes `dateStr` and `timeStr` from `decodePhonePeTxTimestamp(transactionId)` or `decodeFilenameTimestamp(fileName)`.
2. Normalizes common OCR leading-zero errors in the receipt banner (`/\b[QO](\d[:.]\d{2})/g` $\rightarrow$ `"0$1"`).
3. Searches for explicit banner time (`05:45 pm on 23 Oct 2026`).
4. Searches for explicit banner date (`(\d{1,2})\s+(Jan|Feb|...|Dec)[a-z]*\s+(\d{4})`).
5. If either banner regex fails, falls back to the cryptographically decoded timestamp from the PhonePe Transaction ID.

#### Step 4: Weight Extraction (`weightGrams`)
* Primary RegEx: `/(?:Weight|weight)[^\d]*?(0\.\d{2,5})\s*g/gi`
* Fallback RegEx: `/\b(0\.\d{3,5})\s*g\b/i`

#### Step 5: Rupee Symbol (`₹`) Hallucination Stripping (`cleanCurrency`)
* **The OCR Problem:** In PhonePe's font, the `₹` symbol placed immediately before a number (e.g., `₹9.71` or `₹201.89`) is regularly misread by Tesseract as `29.71` or `7201.89`.
* **The `cleanCurrency(rawNumStr, expectedVal)` Solution:**
  * Parses both the full string (`num`) and the string with its first digit sliced off (`strippedFirst = parseFloat(cleaned.slice(1))`).
  * If an `expectedVal` is known from other fields, it compares `|strippedFirst - expectedVal|` against `|num - expectedVal|` and automatically selects the value that satisfies the accounting identity.
* **Total Amount Guard:** Common micro-SIP amounts like `₹10`, `₹20`, `₹30`, `₹40`, `₹50` often scan as `210`, `220`, ..., `750`. If `totalAmount` lands in `[210, 220, 230, 240, 250, 710, 720, 730, 740, 750]`, `totalAmount % 100` is applied to recover the true `10–50` Rupee amount.
* **GST Guard:** If extracted GST exceeds `₹10` and begins with `"2"` or `"7"` (e.g., `20.30` instead of `₹0.30`), the leading digit is stripped.

#### Step 6: Accounting Identity Cross-Verification
1. **Total vs. Component Reconciliation:**
   $$\text{Total Amount} = \text{Metal Value} + \text{GST}$$
   If $\text{Metal Value} > 0$ and $\text{GST} > 0$, and $|(\text{Metal Value} + \text{GST}) - \text{Total Amount}| > 1$, `totalAmount` is recalculated as $\text{ round}(\text{Metal Value} + \text{GST})$.
2. **Reverse 3% GST Derivation:**
   If `totalAmount` was found but `metalValue` failed to scan:
   $$\text{Metal Value} = \text{round}\left(\frac{\text{Total Amount}}{1.03}\right), \quad \text{GST} = \text{Total Amount} - \text{Metal Value}$$
3. **Implied Rate per Gram Verification:**
   $$\text{Expected Rate} = \frac{\text{Metal Value}}{\text{Weight (g)}}$$
   If the OCR-read `ratePerGram` deviates from `Expected Rate` by more than $8\%$, `ratePerGram` is automatically self-healed to `Expected Rate`.

---

## 5. React Component (`App`) State & Lifecycle Hooks

### State Variables
| State Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `transactions` | `Array<Object>` | `localStorage` or `[]` | Master list of verified Gold & Silver purchases. |
| `syncPassword` | `String` | `localStorage` or `""` | User's secret phrase for cross-device cloud sync. |
| `showPasswordModal`| `Boolean` | `false` | Controls visibility of the Cloud Sync modal dialog. |
| `syncStatus` | `String` | `"idle"` | `"idle"` \| `"syncing"` \| `"success"` \| `"error"`. |
| `syncMsg` | `String` | `""` | Status message displayed inside the Cloud Sync modal. |
| `activeMetal` | `String` | `"Silver"` | Currently viewed metal tab (`"Silver"` or `"Gold"`). |
| `unitMode` | `String` | `"both"` | Weight display mode: `"both"`, `"g"`, or `"mg"`. |
| `activeTab` | `String` | `"dashboard"` | Active main view: `"dashboard"`, `"scanner"`, or `"ledger"`. |
| `liveRates` | `Object` | `{ Silver: 201.89, Gold: 14487.58 }` | Editable live market rates per gram for PnL calculation. |
| `scanQueue` | `Array<Object>` | `[]` | Queue of uploaded screenshots currently being OCR-processed or reviewed. |
| `monthlySip` | `Number` | `500` | Monthly micro-SIP investment amount (₹) for the Forecaster. |
| `forecastMonths` | `Number` | `6` | Forecast time horizon in months (`1`–`24`). |

### Memoized Selectors (`useMemo`)
* **`metalTxs`:** Filters `transactions` by `activeMetal` and sorts chronologically ascending using `date + "T" + to24HourTime(time)`.
* **`stats`:** Computes real-time portfolio metrics (`totalWeight`, `totalPaid`, `totalMetalValue`, `totalGst`, `weightedAvgRate`, `breakEvenRate`, `currentVal`, `netPnL`, `count`).
* **`forecast`:** Computes linear rate slope across historical purchases (`(rates[n-1] - rates[0]) / (n - 1)`), dampened by bounds (`Math.max(avgRate * 0.7, ...)`), and projects future accumulated grams after deducting 3% GST (`futureInvested * 0.97`).

---

## 6. Cloud Synchronization Protocol (`syncWithPasswordCloud`)

BullionVault implements a resilient, zero-configuration cloud sync mechanism that requires no user accounts or API keys.

### 6.1 Key Derivation (`getPasswordHash`)
1. Trims the user's password and prepends a versioned application salt: `"bullion-vault-salt-v2-" + cleanPwd`.
2. Hashes the string with `SHA-256` and slices the first 32 hex characters (`binKey`).
3. Constructs the deterministic relay topic: `bv_sync_bin_${binKey}`.

### 6.2 Push Flow (`mode === "push"`)
1. Runs `repairSavedTimestamps(txs)` to ensure clean data before upload.
2. **Tier 1 (JSONBlob Storage):** Attempts to `PUT` (if a `bv_blob_url_${binKey}` already exists in `localStorage`) or `POST` to `https://jsonblob.com/api/jsonBlob` to store large transaction arrays without size limits. Extracts the returned blob URL from the `Location` or `X-jsonblob` header.
3. **Tier 2 (Ntfy.sh Instant Relay):** Sends an HTTP `POST` to `https://ntfy.sh/bv_sync_bin_${binKey}` containing both the `blobUrl` pointer and the inline `transactions` array.

### 6.3 Pull Flow (`mode === "pull"`)
1. Sends an HTTP `GET` request to `https://ntfy.sh/bv_sync_bin_${binKey}/json?poll=1&since=all`.
2. Parses the newline-delimited JSON (`NDJSON`) stream in reverse order (newest message first) until it finds the latest valid `event === "message"` packet.
3. If `latestData.blobUrl` is present, fetches the full JSON payload from `jsonblob.com`, falling back seamlessly to `latestData.transactions` if `jsonblob.com` is unreachable.
4. Updates React state (`setTransactions`) and persists to `localStorage`.
