# BullionVault — Setup, Usage & Troubleshooting Guide

Because **BullionVault** is engineered as a self-contained, single-file application (`index.html`), it requires zero build tools (`npm`, `webpack`, or `vite` are not required) and zero backend servers.

---

## 🚀 1. How to Run Locally

### Option A: Direct Double-Click (Fastest)
1. Save `index.html` to any folder on your computer or phone.
2. Double-click `index.html` to open it in **Google Chrome**, **Microsoft Edge**, **Brave**, **Firefox**, or **Safari**.
3. *Note:* Ensure you have an active internet connection on initial load so the browser can fetch the CDN scripts (React, Tailwind CSS, and the Tesseract.js OCR WebAssembly worker).

### Option B: Local Web Server (Recommended for Full Web Crypto & PWA Behavior)
Some mobile browsers restrict `window.crypto.subtle` (used for `SHA-256` file hashing and password sync) when opened via `file://` URLs. Running a lightweight local HTTP server guarantees full Web Crypto API support:

* **Using Python 3:**
  ```bash
  python -m http.server 8080
  ```
  Then open `http://localhost:8080` in your browser.

* **Using Node.js (`npx`):**
  ```bash
  npx serve .
  ```

* **Using VS Code:**
  Install the **Live Server** extension, right-click `index.html`, and select **"Open with Live Server"**.

---

## 🌐 2. Hosting for Free (Access on Both PC & Mobile)

To use BullionVault seamlessly across your PC and smartphone, host `index.html` on any free static hosting provider with HTTPS enabled:

### GitHub Pages
1. Create a new GitHub repository and upload `index.html`.
2. Go to **Settings $\rightarrow$ Pages**.
3. Select the `main` branch and `/ (root)` folder, then click **Save**.
4. Your personal tracker will be live at `https://<your-username>.github.io/<repo-name>/`.

### Netlify Drop / Vercel / Cloudflare Pages
1. Visit [app.netlify.com/drop](https://app.netlify.com/drop).
2. Drag and drop the folder containing `index.html` onto the page.
3. Netlify will instantly generate an `https://...netlify.app` URL that you can bookmark on both your PC and mobile home screen.

---

## 📷 3. Step-by-Step: Scanning PhonePe Receipts

1. **Take a Screenshot:** Open PhonePe on your phone, navigate to your Gold or Silver purchase receipt (showing the timestamp at the top, weight, rate/g, GST breakdown, and `NB...` Transaction ID), and take a screenshot.
2. **Upload to BullionVault:**
   * Click **"+ Upload Screenshot"** in the top navigation bar (you can select a single screenshot or batch-select dozens at once).
   * The app will automatically switch to the **📷 Screenshot OCR Queue** tab.
3. **Review Extracted Data:**
   * Watch the real-time progress indicator (`Reading Receipt Text... 0% -> 100%`).
   * Once complete, verify the extracted fields (**Metal**, **Date**, **Time**, **Rate/g**, **Weight**, **Metal Value**, **GST**, **Total Paid**, and **PhonePe Txn ID**).
   * Every field is directly editable in case you want to tweak any value before saving.
4. **Save to Portfolio:**
   * Click **"✓ Save to Portfolio"**.
   * If you have already configured a **Sync Password**, saving an item automatically pushes your updated portfolio to the cloud in the background.

---

## 🔐 4. Setting Up Cross-Device Password Cloud Sync

You can keep your PC and Mobile browsers in sync without creating any accounts:

### On Your Primary Device (e.g., PC)
1. Click the **"🔐 Set Sync Password"** button in the top header.
2. Enter a unique secret password (at least 4 characters, e.g., `MyBullionVault2026!`).
3. Click **"✓ Save & Push Sync"**.
4. Wait for the green confirmation message:  
   `✓ Pushed X transactions to Cloud! Now click "Pull from Cloud" on your mobile.`

### On Your Secondary Device (e.g., Mobile)
1. Open BullionVault on your phone and tap **"🔐 Set Sync Password"**.
2. Enter the **exact same password** (case-sensitive).
3. Tap **"⬇ Pull from Cloud"**.
4. Your entire transaction history, timestamps, and metrics will populate immediately.

---

## 💾 5. Offline JSON Backups

In addition to Cloud Sync, you can create permanent offline backups at any time:
* **Export:** Click **"⬇ Backup JSON"** on the right side of the navigation bar to download a `bullion-vault-backup-YYYY-MM-DD.json` file.
* **Restore:** Click **"⬆ Restore JSON"** and select a previously downloaded `.json` backup file to restore your entire ledger.

---

## 🔧 6. Troubleshooting & FAQ

### Q1: Why is a screenshot flagged as `⛔ Blocked: Already in Portfolio` or `Duplicate PhonePe ID`?
* **Cause:** BullionVault checks the image's `SHA-256` hash, the PhonePe `NB...` Transaction ID, and the composite financial signature (`Metal + Date + Time + Weight + Total Paid`) against your currently saved ledger.
* **Fix:** If you intentionally want to log it anyway, click the amber **"Force Allow & Save"** button on the queued item. If you want to start completely fresh, click the red **"Clear All"** button in the toolbar.

### Q2: Why did the OCR misread a digit in my screenshot?
* **Cause:** Low-resolution screenshots or cropped receipts can occasionally obscure digits.
* **Built-in Fix:** BullionVault automatically runs a math cross-check (`Metal Value + GST = Total` and `Metal Value / Weight = Rate/g`) to self-heal `₹` symbol misreads. You can also manually edit any field inside the OCR queue card before clicking **Save to Portfolio**.

### Q3: Why does "Pull from Cloud" say `No cloud data found for this password yet`?
* **Cause:** Either you haven't clicked **"✓ Save & Push Sync"** on your first device yet, or there is a capitalization/spelling difference in the password between your two devices.
* **Fix:** Ensure both devices use the exact same password (including uppercase/lowercase letters) and click **"✓ Save & Push Sync"** on the device that currently holds your transactions first.