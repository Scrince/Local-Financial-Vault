# Changelog

All notable changes to **Local Financial Vault** are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) conventions.
Versions are dated rather than semver-tagged, since the project ships as a single HTML file with no package registry.

---

## [Unreleased]

Changes staged but not yet formally released go here.

---

## [2.0.0] — 2025-05-06

### ⚠ Breaking Change — Vault Format V2

**V2 `.lfv` files cannot be opened by V1 builds of the app, and V1 files cannot be opened by V2 builds.** Before updating, follow the [Migration Guide](MIGRATION.md) to export and re-import your data.

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
- All cryptographic parameters (`kdf`, `kdf_iterations`, `cipher`, `mac_algo`, `salt_b64`, `nonce_b64`, `ciphertext_b64`, `mac_b64`) are embedded in the file — fully self-describing with no external configuration.
- Fresh 256-bit salt and 256-bit nonce are generated on every save, regardless of whether the password changed.

---

### Added

- **V2 file armor** — `.lfv` files now parse and validate as JSON. The magic header `LOCALFINANCIALVAULT-V2` and `version: 2` field are checked on open before any decryption is attempted.
- **KDF progress indicator** — a status message (`🔐 Deriving key (this takes a moment)…`) is displayed during save and open operations while the 1M-iteration KDF runs, so users are not left wondering if the app has frozen.
- **Richer error messages** — decryption failures now surface the underlying error detail (wrong password vs. corrupted file vs. format mismatch) rather than a single generic alert.
- **`format` field in vault metadata** — the decrypted vault JSON now includes `"format": "LOCALFINANCIALVAULT-V2"` in its metadata block for forward-compatibility identification.
- **`version: "2.0"` in vault metadata** — internal vault schema version bumped from `"1.0"` to `"2.0"`.

### Changed

- **Save file MIME type** — `.lfv` files are now served as `application/json` instead of `text/plain` in download fallback mode.
- **Open error handling** — both the standard file-input open path and the File System Access API path now display a status message reset on failure, preventing the UI from showing a stale "decrypting…" message after a bad password attempt.
- **SECURITY.md** — fully rewritten to document the V2 scheme, updated threat model, and formal vulnerability reporting process.
- **README.md** — encryption model section, file format section, and data model section updated to reflect V2.
- **CONTRIBUTING.md** — cryptographic contribution guidelines added; commit message format specified.

### Removed

- **PBKDF2-SHA-256 + AES-GCM** encryption (V1 scheme) — entirely removed from the codebase. No V1 compatibility shim is provided. See [MIGRATION.md](MIGRATION.md).
- **`getKey()` function** — replaced by `deriveKeys()`.
- **`joinBufs()` helper** — no longer needed; the V2 format uses JSON fields rather than a concatenated binary blob.
- **Raw Base64 vault format** — V1 files (bare Base64 strings with no JSON wrapper) will fail the magic-header check and produce a clear error message.

---

## [1.0.0] — 2025 (initial release)

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
|:---:|---|---|---|:---:|
| 1 | *(none — raw Base64)* | PBKDF2-HMAC-SHA-256, 100k iter | AES-256-GCM | ❌ Retired |
| 2 | `LOCALFINANCIALVAULT-V2` | PBKDF2-HMAC-SHA-512, 1M iter | HMAC-SHA-512-CTR | ✅ Current |

[Unreleased]: https://github.com/Scrince/Local-Financial-Vault/compare/main...HEAD
[2.0.0]: https://github.com/Scrince/Local-Financial-Vault/releases/tag/v2.0.0
[1.0.0]: https://github.com/Scrince/Local-Financial-Vault/releases/tag/v1.0.0
