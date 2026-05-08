# Changelog

All notable changes to **Local Financial Vault** are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) conventions.
Versions are dated rather than semver-tagged, since the project ships as a single HTML file with no package registry.

---

## [Unreleased](https://github.com/Scrince/Local_Financial_Vault/compare/main...HEAD)

Changes staged but not yet formally released go here.

---

## [2.2.0](https://github.com/Scrince/Local_Financial_Vault/releases/tag/v2.2.0) — 2026-05-07

> **Compatible with V2 vault files.** No re-encryption or migration is required.

### Added

- **Inline cell editing** — clicking any data cell in a category table opens it for direct editing without opening the full Add/Edit modal. A pencil icon (✎) appears on hover to indicate editability. Press **Enter** to commit, **Escape** to cancel, or click away to save. Select-type fields (e.g. Card Type) render as a dropdown. Number fields store raw values without currency formatting; the formatted display is restored after saving. Inline editing is suppressed while a search filter is active to prevent row-index mismatches. Notes and computed columns (e.g. Net Equity on Other Assets) are not inline-editable and continue to require the modal.
- **Unsaved changes warning banner** — an amber banner is displayed below the search bar whenever the vault has unsaved changes. The banner includes a **Save Now** shortcut button that triggers the normal save flow. The banner is shown after any data-mutating action (add, edit, delete, duplicate, inline cell edit, drag-to-reorder, snapshot record, snapshot delete) and is dismissed automatically after a successful save or when the vault is locked and cleared.
- **`beforeunload` guard** — if the vault has unsaved changes, attempting to close or refresh the browser tab triggers the browser's standard "Leave site?" confirmation dialog, preventing accidental data loss.

### Fixed

- **Crypto `quantity` field type** — the Quantity field in the Crypto schema has been changed from `number` to `text`. This prevents the live comma-formatting and currency-stripping logic from being applied to a field that holds a unit count rather than a dollar amount (e.g. `0.00473821` BTC). The value is displayed and stored as entered and is not used in any financial calculations.

---

## [2.1.0](https://github.com/Scrince/Local_Financial_Vault/releases/tag/v2.1.0) — 2026-05-07

> **Compatible with V2 vault files.** No re-encryption or migration is required. The `migrateVault()` function automatically upgrades any V2 vault with the older schema layout on first open. See [MIGRATION.md](https://github.com/Scrince/Local_Financial_Vault/blob/main/MIGRATION.md) for details.

### Added

- **Drag-and-drop row reordering** — rows in every category table now have a drag handle (⠿) and can be dragged to reorder entries. Drag handles and highlighting are suppressed while a search filter is active. Reordering updates `vault.metadata.lastModifiedAt` and re-renders the table immediately.
- **Password match progress bar** — a live progress bar and label appear below the "Confirm password" field in the Create Vault modal, showing whether the two password fields match and how much prefix overlap exists.
- **Comma-formatted monetary input** — all `number`-type schema fields in the Add/Edit modal render as formatted text inputs with live comma insertion as the user types (e.g. `1234567.89` displays as `1,234,567.89`). Raw numeric values are stored internally without commas.
- **Expiration date auto-mask** — the Credit Card `expDate` field auto-inserts a `/` separator after two digits, producing `MM/YY` format as the user types.
- **Export redaction options** — the Export modal now includes a "Redact Sensitive Fields" section with three checkboxes:
  - *Bank account numbers* — truncates to last 4 digits in the export output.
  - *Card numbers* — truncates to last 4 digits.
  - *Card CVV & PIN* — replaces both fields with `•••`.
  - Redaction is applied to all three export formats (CSV, JSON, PDF).
- **Charts embedded in PDF export** — if the Snapshots tab has been visited during the current session, the three canvas charts (Net Worth Over Time, Assets vs. Debts, Assets by Category) are captured as PNG images and embedded in the PDF export print view.
- **`cardLimit` field on Credit Cards** — a new "Card Limit ($)" field has been added to the Credit Cards schema, separate from the existing "Balance ($)" field. The two fields serve different purposes: `balance` reflects the current amount owed or available, while `cardLimit` records the total credit limit on the card.
- **`cardType` select on Credit Cards** — a new "Card Type" select field (options: Credit, Debit, Other) has been added to the Credit Cards schema.
- **Net Equity computed column on Other Assets** — a read-only "Net Equity" column is displayed in the Other Assets table and in exports, calculated as `estimatedValue − loanAmount`.
- **`walletId` field on Crypto** — a new "ID" field stores the wallet address or exchange account identifier for each crypto entry.
- **In-session save indicator** — `updateSaveIndicator()` tracks whether the current vault has unsaved changes and reflects this in the UI.

### Changed

- **Stocks schema** — the `purchasePrice` and `currentPrice` fields have been removed from the Stocks entry form. `value` (a single dollar-amount field labelled "Value ($)") and `shares` (now a plain text field) replace the former computed model. This is a within-V2 schema change; `migrateVault()` computes an initial `value` from `shares × currentPrice` for any vault opened that still carries the old fields.
- **Crypto schema** — `purchasePrice` has been removed from the Crypto entry form. The `currentPrice` field label has been renamed to "Value ($)" to reflect that it now stores the total holding value rather than a per-unit price. `migrateVault()` drops `purchasePrice` on open.
- **Credit Cards schema** — `creditLimit` (the old field name in V1 and early V2 vaults) is automatically renamed to `balance` by `migrateVault()` on open if `balance` is not already present. The new `cardLimit` field is separate and distinct.
- **Export modal** — category list is now generated dynamically from `EXPORT_CATEGORIES` constant; "Select All" and "Select None" controls added. Format tabs (CSV / JSON / PDF) updated to reflect new PDF chart-embedding behaviour.
- **KDF progress message wording** — the status message shown during open/save now reads `"🔐 Verifying key & MAC (this takes a moment)…"` to better describe what is happening during the 1M-iteration PBKDF2 operation on open.
- **MIGRATION.md** — updated to document within-V2 schema migrations handled by `migrateVault()`.
- **README.md** — category feature table updated to reflect current schemas; new features section added.
- **SECURITY.md** — export redaction section added; no cryptographic changes.
- **THREAT_MODEL.md** — supply-chain section updated with SHA-256 checksum note; Revision History updated.

### Fixed

- **Duplicate entry on Credit Cards** with `expMonth`/`expYear` fields — `migrateVault()` now correctly merges these into `expDate` (MM/YY) regardless of whether the entry came from a V1-era export JSON or an early V2 vault.
- **Counter encoding in HMAC-SHA512-CTR** — the counter byte-encoding loop previously had a redundant first pass; the correct big-endian encoding now runs only once per block.

---

## [2.0.0](https://github.com/Scrince/Local_Financial_Vault/releases/tag/v2.0.0) — 2025-05-06

### ⚠ Breaking Change — Vault Format V2

**V2 `.lfv` files cannot be opened by V1 builds of the app, and V1 files cannot be opened by V2 builds.** Before updating, follow the [Migration Guide](https://github.com/Scrince/Local_Financial_Vault/blob/main/MIGRATION.md) to export and re-import your data.

---

### Security — Encryption Overhaul (V1 → V2)

The entire encryption scheme has been replaced. The previous scheme (PBKDF2-SHA256 + AES-GCM, 100,000 KDF iterations) has been retired and is no longer supported.

**V2 scheme — `LOCALFINANCIALVAULT-V2`:**

| Component | V1 | V2 |
|---|---|---|
| KDF | PBKDF2-HMAC-**SHA-256** | PBKDF2-HMAC-**SHA-512** |
| KDF iterations | 100,000 | **1,000,000** |
| Key derivation | Single key | **Domain-separated** enc key + auth key |
| Cipher | **AES-256-GCM** | **HMAC-SHA-512-CTR** stream cipher |
| Authentication | GCM built-in tag | Explicit **HMAC-SHA-512 MAC** |
| Auth ordering | Encrypt-and-MAC | **Encrypt-then-MAC** |
| Salt size | 128-bit | **256-bit** |
| Nonce/IV size | 96-bit (GCM IV) | **256-bit** |
| File format | Raw Base64 blob | **JSON armor** with magic header |

**Key derivation details:**
- PBKDF2-HMAC-SHA-512 produces 512-bit input key material (IKM).
- `encKey  = HMAC-SHA-512(IKM, "encryption")` — 512-bit encryption key.
- `macKey  = HMAC-SHA-512(IKM, "authentication")` — 512-bit authentication key.
- Domain labels prevent any cryptographic cross-context key reuse.

**Encryption details:**
- HMAC-SHA-512-CTR generates a 64-byte keystream block per counter: `pad[i] = HMAC-SHA-512(encKey, nonce ‖ counter_i)`.
- Plaintext is XORed against the keystream. No block cipher, no padding oracle surface.

**Authentication details:**
- `MAC = HMAC-SHA-512(macKey, magic ‖ salt ‖ nonce ‖ ciphertext)`.
- MAC covers the full ciphertext and all envelope parameters.
- MAC is **verified before decryption**. A wrong password or any byte-level tampering is detected immediately with no plaintext exposure.

**File format details:**
- `.lfv` files are now UTF-8 JSON with a `"magic": "LOCALFINANCIALVAULT-V2"` header.
- All cryptographic parameters are embedded in the file — fully self-describing.
- Fresh 256-bit salt and 256-bit nonce are generated on every save.

---

### Added

- **V2 file armor** — `.lfv` files now parse and validate as JSON.
- **KDF progress indicator** — status message during save/open operations.
- **Richer error messages** — decryption failures surface the underlying error detail.
- **`format` field in vault metadata** — `"format": "LOCALFINANCIALVAULT-V2"`.
- **`version: "2.0"` in vault metadata** — bumped from `"1.0"`.

### Changed

- **Save file MIME type** — `.lfv` files served as `application/json` in download fallback mode.
- **Open error handling** — status message reset on failure.
- **SECURITY.md** — fully rewritten to document the V2 scheme.
- **README.md** — encryption model, file format, and data model sections updated.
- **CONTRIBUTING.md** — cryptographic contribution guidelines added.

### Removed

- **PBKDF2-SHA-256 + AES-GCM** encryption (V1 scheme) — entirely removed.
- **`getKey()` function** — replaced by `deriveKeys()`.
- **`joinBufs()` helper** — no longer needed.
- **Raw Base64 vault format** — V1 files will fail the magic-header check with a clear error message.

---

## [1.0.0](https://github.com/Scrince/Local_Financial_Vault/releases/tag/v1.0.0) — 2025 (initial release)

### Added

- Single-file offline financial vault application.
- Encrypted vault files using PBKDF2-SHA-256 (100,000 iterations) + AES-256-GCM.
- Vault file format: raw Base64-encoded binary (salt + IV + ciphertext concatenated).
- Financial categories: Bank Accounts, Credit Cards, Retirement Accounts, Stocks, Crypto, Other Assets, Debts.
- Add / Edit / Duplicate / Delete entries per category.
- Global search with live highlighting across all fields.
- Net worth calculation (assets − debts) displayed in real time.
- Financial snapshots with timestamp, total assets, total debts, net worth.
- Charts: net worth over time (line), assets vs. debts (bar), assets by category (pie) — pure HTML Canvas, no libraries.
- CSV, JSON, and PDF (print) export with per-category selection.
- Dark mode (default) and light mode; preference persisted in `localStorage`.
- 10-minute inactivity auto-lock.
- File System Access API (FSAA) in-place save for Chromium-based browsers.
- Download fallback for non-FSAA browsers.
- Settings modal: dark/light mode toggle, change vault password.
- MIT License.
- README, SECURITY, CONTRIBUTING documentation.

---

## Format Version Reference

| Format version | Magic header | KDF | Cipher | Status |
|---|---|---|---|---|
| 1 | *(none — raw Base64)* | PBKDF2-HMAC-SHA-256, 100k iter | AES-256-GCM | ❌ Retired |
| 2 | `LOCALFINANCIALVAULT-V2` | PBKDF2-HMAC-SHA-512, 1M iter | HMAC-SHA-512-CTR | ✅ Current |
