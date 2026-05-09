# 🔐 Local Financial Vault

**Local Financial Vault** is a secure, fully offline, single-page web application for managing all your financial accounts, assets, debts, and net-worth history in one encrypted vault file. All data is stored locally, encrypted with modern cryptography, and never leaves your device.

> **No servers. No cloud. No telemetry. No dependencies. One HTML file.**

---

## Features

### 🔐 Secure Encrypted Vault (V2 Format)

- Create a new vault protected by a **master password** (minimum 12 characters recommended).
- Vaults are encrypted using the **LOCALFINANCIALVAULT-V2** format — a hardened scheme inspired by [Arculus Recovery](https://github.com/Scrince/Arculus_Recovery):
  - **PBKDF2-HMAC-SHA512** with **1,000,000 iterations** for key derivation
  - **Domain-separated keys** — a dedicated encryption key and a separate authentication key, each derived via HMAC from the master key material
  - **HMAC-SHA512-CTR stream cipher** for encryption (no block cipher modes, no padding oracles)
  - **HMAC-SHA512 MAC** over the full ciphertext for authentication (encrypt-then-MAC)
  - **256-bit random salt** and **256-bit random nonce** per save — no IV reuse possible
- Vault files are **JSON-armored** with a magic header, version tag, and all cryptographic parameters embedded — fully self-describing and verifiable.
- No plaintext financial data is ever written to disk.
- Fully offline — **no servers, no cloud, no external APIs or CDNs**.

### 🔑 Optional Keyfile Protection

- Toggle **Require a keyfile** during vault creation to add a second authentication factor.
- Choose between a **raw keyfile** (`.key`) — 64 bytes of cryptographically random data — or an **encrypted keyfile** (`.enc.key`) protected by its own separate password.
- The keyfile is mixed into the password via `HMAC-SHA512(keyfileBytes, UTF8(password))` before KDF, cryptographically binding both factors. Either alone is insufficient to open the vault.
- After generating a keyfile, the file input is **automatically populated** with the just-generated file — no need to manually re-select it before clicking Create.
- A **live password match bar** is shown when setting an encrypted keyfile password, identical to the one used for the vault master password.
- A warning is shown at creation time: losing the keyfile makes the vault permanently unrecoverable. Keep a backup in a separate, safe location.

### 📁 Vault Lifecycle

- **Auto-save on creation** — clicking **Create** immediately encrypts and saves the vault to disk. On Chromium-based browsers the File System Access API picker opens pre-filled with the vault name; on Firefox/Safari the file downloads automatically. No separate Save step is required after creation.
- In-place overwrite via the **File System Access API** (Chromium-based browsers) — no re-download required on subsequent saves.
- Fallback download for all other browsers — saves use the same file name automatically.
- Opening a vault replaces all in-memory data.
- Automatic lock after **10 minutes of inactivity**.
- Manual lock/clear at any time.
- **Unsaved changes banner** — an amber warning banner appears whenever there are unsaved changes, with a one-click **Save Now** shortcut. Closing or refreshing the tab with unsaved changes triggers the browser's "Leave site?" confirmation.

### 🧾 Structured Financial Data

Manage entries across all major financial categories:

| Category | Key Fields |
|---|---|
| **Bank Accounts** | Institution, type, account #, routing #, balance, notes |
| **Credit Cards** | Issuer, card type, card #, expiry (MM/YY), CVV, PIN, balance, card limit, notes |
| **Retirement Accounts** | Institution, type (401k/IRA…), account #, current balance, notes |
| **Stocks** | Ticker, company, shares, value ($), brokerage, notes |
| **Cryptocurrency** | Coin, symbol, quantity, value ($), wallet/exchange, wallet ID, notes |
| **Other Assets** | Category, description, estimated value, loan remaining, net equity (computed), notes |
| **Debts** | Type, lender, account #, balance, interest rate, minimum payment, notes |

Supported operations per entry: **Add · Edit · Duplicate · Delete**

### ✏ Inline Cell Editing

- Click any data cell in a table row to edit it directly — no modal required.
- A pencil icon (✎) appears on hover to indicate editable cells.
- Press **Enter** to commit, **Escape** to cancel, or click away to save.
- Select-type fields (e.g. Card Type) render as a dropdown inline.
- Suppressed while a search filter is active. Notes and computed columns (e.g. Net Equity) continue to require the full modal.

### 🔍 Global Search & Filtering

- Real-time, case-insensitive substring search across **all fields in all categories**.
- Result count displayed live; matched text highlighted in the table.

### 📊 Net Worth & Financial Calculations

Automatically computes and displays in the header bar:

- **Total Assets** — bank balances + retirement balances + stock values + crypto values + other asset equity
- **Total Debts** — sum of all debt balances
- **Net Worth** — assets minus debts

### 📈 Snapshots & Visualizations

Record point-in-time financial snapshots and view trends:

- **Net Worth Over Time** — line chart with gradient fill
- **Assets vs. Debts** — side-by-side bar chart
- **Assets by Category** — pie chart breakdown

Charts are rendered in pure **HTML Canvas** — no Chart.js, no external libraries.

### ↕ Row Reordering

Drag the ⠿ handle on any row to reorder entries within a category. Reordering is suppressed while a search filter is active.

### ☀/🌙 Appearance

- Dark mode (default) and light mode, toggled in Settings.
- Preference persisted in `localStorage`.

### ⬇ Export

Export any combination of categories to:

- **CSV** — spreadsheet-compatible, one section per category
- **JSON** — structured data dump
- **PDF** — formatted print view via browser print dialog, with snapshot charts embedded if the Snapshots tab was visited during the session

Optional redaction on export:
- Bank account numbers (last 4 only)
- Card numbers (last 4 only)
- Card CVV & PIN

---

## Getting Started

1. **Download** `LocalFinancialVault.html` from this repository.
2. **Open** it in any modern browser — no installation, no build step, no internet required.
3. Click **＋ Create Vault**, set a strong password (12+ characters), and start adding entries.
4. The vault is **saved automatically** when you click Create — choose a location when prompted (Chromium) or find the download in your Downloads folder (Firefox/Safari).
5. To re-open later, use the **Open vault file** input and enter your password.

> **Tip (Chromium browsers):** After the initial save, subsequent saves overwrite the file in-place without prompting for a download location.

> **Tip (Keyfile):** Enable the keyfile option during vault creation for a second authentication factor. Generate the keyfile first — it will auto-populate the file input — then click Create. Store the keyfile separately from the vault file.

---

## File Format — LOCALFINANCIALVAULT-V2

`.lfv` files are UTF-8 JSON with the following structure:

```json
{
  "magic":          "LOCALFINANCIALVAULT-V2",
  "version":        2,
  "kdf":            "PBKDF2-HMAC-SHA512",
  "kdf_iterations": 1000000,
  "cipher":         "HMAC-SHA512-CTR",
  "mac_algo":       "HMAC-SHA512",
  "has_keyfile":    false,
  "salt_b64":       "<256-bit random salt, base64>",
  "nonce_b64":      "<256-bit random nonce, base64>",
  "ciphertext_b64": "<encrypted vault JSON, base64>",
  "mac_b64":        "<HMAC-SHA512 over magic+salt+nonce+ciphertext, base64>"
}
```

The file is fully self-describing. Every cryptographic parameter needed to decrypt and verify the vault is stored alongside the ciphertext — no external configuration required. The `has_keyfile` flag allows the Open Vault modal to detect keyfile requirement before any decryption is attempted.

---

## Encryption Model (V2)

```
Password [+ Keyfile (optional)]
      │
      │  If keyfile present:
      │  effectivePassword = HMAC-SHA512(keyfileBytes, UTF8(password))
      │
      ▼
PBKDF2-HMAC-SHA512 (1,000,000 iterations) → 512-bit IKM
      │
      ├─ HMAC-SHA512(IKM, "encryption")     → 512-bit Encryption Key
      └─ HMAC-SHA512(IKM, "authentication") → 512-bit Authentication Key

Plaintext (vault JSON)
      │
      ▼  [HMAC-SHA512-CTR stream cipher, 64-byte blocks]
Ciphertext
      │
      ▼  [HMAC-SHA512(AuthKey, magic ‖ salt ‖ nonce ‖ ciphertext)]
MAC

→ Stored as JSON armor with magic header LOCALFINANCIALVAULT-V2
```

MAC is verified **before** any decryption attempt (encrypt-then-MAC pattern). A wrong password, wrong keyfile, or corrupted file is detected immediately without exposing any plaintext.

When a keyfile is used, the effective entropy floor is **512 bits** (the HMAC-SHA512 output), regardless of password strength. Both factors are required simultaneously — neither alone provides any information about the derived key.

---

## Keyfile Format

Keyfiles are JSON with one of two structures:

**Raw keyfile (`.key`):**
```json
{ "magic": "KEYFILE", "v": 1, "len": 64, "key": "<64 random bytes, base64>" }
```

**Encrypted keyfile (`.enc.key`):**
```json
{
  "magic": "KEYFILE-ENC", "v": 1,
  "kdf":   { "algo": "PBKDF2-SHA512", "i": 200000, "s": "<salt, base64>" },
  "n":     "<nonce, base64>",
  "ct":    "<encrypted key bytes, base64>",
  "mac":   "<HMAC-SHA512 integrity tag, base64>"
}
```

Encrypted keyfiles use PBKDF2-SHA512 (200,000 iterations) + HMAC-SHA512-CTR + HMAC-SHA512 MAC — the same construction as the vault itself, with a 4-byte counter matching the Keyfile Generator tool.

---

## Vault Data Model

The vault JSON (before encryption) follows this schema:

```json
{
  "metadata": {
    "version": "2.0",
    "name": "my-vault",
    "format": "LOCALFINANCIALVAULT-V2",
    "createdAt": "2025-01-01T00:00:00.000Z",
    "lastModifiedAt": "2025-01-01T00:00:00.000Z"
  },
  "accounts": {
    "bankAccounts": [],
    "creditCards": [],
    "retirementAccounts": [],
    "stocks": [],
    "crypto": [],
    "otherAssets": [],
    "debts": []
  },
  "snapshots": [
    {
      "timestamp": "2025-01-01T00:00:00.000Z",
      "totalAssets": 0,
      "totalDebts": 0,
      "netWorth": 0
    }
  ]
}
```

---

## Security Considerations

- **Your password is never stored or transmitted.** It exists only in-memory for the duration of your session.
- **Each save generates a fresh random salt and nonce.** The same password will produce a different ciphertext every time.
- **The 1M-iteration KDF is intentional.** It makes brute-force attacks approximately 10× harder than the previous 100k-iteration scheme. Expect 2–5 seconds for key derivation on modern hardware — a status message is displayed during this time.
- **No metadata leakage.** The encrypted payload is the entire vault object. Category names, entry counts, and field values are all encrypted.
- **Keyfile security.** The keyfile and vault file should be stored separately. If both are on the same device and an attacker obtains both, security reduces to the strength of the password alone. The keyfile provides the strongest protection when kept on a separate physical medium (USB drive, printed QR code).
- **This tool does not protect against a compromised device.** If your OS, browser, or file system is compromised, no client-side encryption can fully protect your data.

See [SECURITY.md](SECURITY.md) for the full security policy and vulnerability reporting process.

---

## Browser Compatibility

| Browser | Save (in-place) | Save (download) | Open | Charts |
|---|:---:|:---:|:---:|:---:|
| Chrome / Edge / Brave | ✅ | ✅ | ✅ | ✅ |
| Firefox | ❌ | ✅ | ✅ | ✅ |
| Safari | ❌ | ✅ | ✅ | ✅ |

In-place save requires the [File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API), available in Chromium-based browsers. All other functionality works in any modern browser with Web Crypto API support.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on submitting issues, pull requests, and security changes.

---

## License

[MIT](LICENSE.md) — free to use, modify, and distribute.

---

## Acknowledgements

The V2 encryption scheme is inspired by the [Arculus Recovery](https://github.com/Scrince/Arculus_Recovery) project's `.arc` file format, adapted for the financial vault use case with a custom magic header and `.lfv` file extension.
