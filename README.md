# 🔐 Local Financial Vault

**Local Financial Vault** is a secure, fully offline, single-page web application for managing all your financial accounts, assets, debts, and net-worth history in one encrypted vault file. All data is stored locally, encrypted with modern cryptography, and never leaves your device.

> **No servers. No cloud. No telemetry. No dependencies. One HTML file.**

---

## Features

### 🔐 Secure Encrypted Vault (V2 Format)

- Create a new vault protected by a **master password** (minimum 12 characters).
- Vaults are encrypted using the **LOCALFINANCIALVAULT-V2** format — a hardened scheme inspired by [Arculus Recovery](https://github.com/Scrince/Arculus_Recovery):
  - **PBKDF2-HMAC-SHA512** with **1,000,000 iterations** for key derivation
  - **Domain-separated keys** — a dedicated encryption key and a separate authentication key, each derived via HMAC from the master key material
  - **HMAC-SHA512-CTR stream cipher** for encryption (no block cipher modes, no padding oracles)
  - **HMAC-SHA512 MAC** over the full ciphertext for authentication (encrypt-then-MAC)
  - **256-bit random salt** and **256-bit random nonce** per save — no IV reuse possible
- Vault files are **JSON-armored** with a magic header, version tag, and all cryptographic parameters embedded — fully self-describing and verifiable.
- No plaintext financial data is ever written to disk.
- Fully offline — **no servers, no cloud, no external APIs or CDNs**.

### 📁 Vault Lifecycle

- Create, open, and save encrypted `.lfv` vault files.
- In-place overwrite via the **File System Access API** (Chromium-based browsers) — no re-download required.
- Fallback download for all other browsers — saves use the same file name automatically.
- Opening a vault replaces all in-memory data.
- Automatic lock after **10 minutes of inactivity**.
- Manual lock/clear at any time.

### 🧾 Structured Financial Data

Manage entries across all major financial categories:

| Category | Key Fields |
|---|---|
| **Bank Accounts** | Institution, type, account #, routing #, balance, PIN, notes |
| **Credit Cards** | Issuer, card #, expiry, CVV, credit limit, notes |
| **Retirement Accounts** | Institution, type (401k/IRA…), account #, balance, notes |
| **Stocks** | Ticker, company, shares, purchase price, current price, brokerage, notes |
| **Cryptocurrency** | Coin, symbol, quantity, purchase price, current price, wallet/exchange, notes |
| **Other Assets** | Category, description, estimated value, notes |
| **Debts** | Type, lender, account #, balance, interest rate, minimum payment, notes |

Supported operations per entry: **Add · Edit · Duplicate · Delete**

### 🔍 Global Search & Filtering

- Real-time, case-insensitive substring search across **all fields in all categories**.
- Result count displayed live; matched text highlighted in the table.

### 📊 Net Worth & Financial Calculations

Automatically computes and displays in the header bar:

- **Total Assets** — bank balances + retirement + stocks (shares × current price) + crypto (quantity × current price) + other assets
- **Total Debts** — sum of all debt balances
- **Net Worth** — assets minus debts

### 📈 Snapshots & Visualizations

Record point-in-time financial snapshots and view trends:

- **Net Worth Over Time** — line chart with gradient fill
- **Assets vs. Debts** — side-by-side bar chart
- **Assets by Category** — pie chart breakdown

Charts are rendered in pure **HTML Canvas** — no Chart.js, no external libraries.

### ☀/🌙 Appearance

- Dark mode (default) and light mode, toggled in Settings.
- Preference persisted in `localStorage`.

### ⬇ Export

Export any combination of categories to:

- **CSV** — spreadsheet-compatible, one section per category
- **JSON** — structured data dump
- **PDF** — formatted print view via browser print dialog

---

## Getting Started

1. **Download** `LocalFinancialVault.html` from this repository.
2. **Open** it in any modern browser — no installation, no build step, no internet required.
3. Click **＋ Create Vault**, set a strong password (12+ characters), and start adding entries.
4. Click **💾 Save Vault** to save an encrypted `.lfv` file to your disk.
5. To re-open later, use the **Open vault file** input and enter your password.

> **Tip (Chromium browsers):** After saving once, subsequent saves overwrite the file in-place without prompting for a download location.

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
  "salt_b64":       "<256-bit random salt, base64>",
  "nonce_b64":      "<256-bit random nonce, base64>",
  "ciphertext_b64": "<encrypted vault JSON, base64>",
  "mac_b64":        "<HMAC-SHA512 over magic+salt+nonce+ciphertext, base64>"
}
```

The file is fully self-describing. Every cryptographic parameter needed to decrypt and verify the vault is stored alongside the ciphertext — no external configuration required.

---

## Encryption Model (V2)

```
Password + Salt
      │
      ▼
PBKDF2-HMAC-SHA512 (1,000,000 iterations) → 512-bit IKM
      │
      ├─ HMAC-SHA512(IKM, "encryption")   → 512-bit Encryption Key
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

MAC is verified **before** any decryption attempt (encrypt-then-MAC pattern). A wrong password or corrupted file is detected immediately without exposing any plaintext.

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
