# Migration Guide

This document covers two distinct migration scenarios:

1. **V1 → V2** — upgrading a vault encrypted with the old PBKDF2-SHA256 + AES-GCM scheme to the current V2 format. This is a one-time manual process.
2. **Within-V2 schema migrations** — the app automatically handles these on open via the built-in `migrateVault()` function. No user action is required.

---

## Part 1 — V1 → V2 Migration (Manual)

### Am I Affected?

You are affected if your `.lfv` file was saved by a version of the app that used **PBKDF2-SHA-256 + AES-GCM** encryption. You can tell which format your file is in by opening it in a plain text editor:

**V1 file** — looks like a raw Base64 string (a long block of letters, numbers, `+`, `/`, and `=` with no line breaks or structure):

```
c29tZWxvbmdicmFzZTY0c3RyaW5nd2l0aG5vanNvbnN0cnVjdHVyZWF0YWxs...
```

**V2 file** — looks like formatted JSON with a clear magic header:

```json
{
  "magic": "LOCALFINANCIALVAULT-V2",
  "version": 2,
  ...
}
```

If your file is V2, you are already on the current format — no V1→V2 migration needed.

---

### Why Is Migration Required?

The V1 and V2 encryption schemes are fundamentally incompatible. The new build cannot decrypt a V1 file, and the old build cannot open a V2 file. This is by design: the format change involves not just new parameters but an entirely different cryptographic construction (stream cipher vs. block cipher, HMAC authentication vs. GCM tag, domain-separated keys vs. a single derived key).

A silent compatibility shim would have required shipping V1 decryption code alongside V2, keeping old cryptographic surface area alive indefinitely. Instead, migration is a one-time manual step.

---

### Migration Steps (V1 → V2)

The migration process is: open your vault with the **old** app → export your data as JSON → open the new app → re-enter your data → save as a V2 vault.

#### Step 1 — Get the Old App Version

You need the V1 build of `LocalFinancialVault.html` to decrypt your existing file.

- If you still have your old copy of `LocalFinancialVault.html`, use that.
- If not, retrieve it from your browser's download history, a backup, or from the Git history of this repository:

```bash
git clone https://github.com/Scrince/Local_Financial_Vault.git
cd Local_Financial_Vault
git log --oneline        # find the last V1 commit
git checkout <commit-hash> -- Local_Financial_Vault.html
```

Open the checked-out file in your browser. **Do not update the file yet.**

#### Step 2 — Open Your V1 Vault

1. Open the old `Local_Financial_Vault.html` in your browser.
2. Use the **Open vault file** input to select your `.lfv` file.
3. Enter your password. Your vault data should load normally.

If this step fails, check:
- You are using the old (V1) HTML file, not the new one.
- Your password is correct.
- The `.lfv` file is not corrupted (check file size is reasonable; a near-empty vault is typically a few KB).

#### Step 3 — Export Your Data as JSON

1. Click the **⬇ Export** button in the top bar.
2. In the Export modal, select the **`{ } JSON`** format tab.
3. Click **Select All** to include all categories (or choose specific ones).
4. Click **Export**. A `.json` file will download — keep this file safe. **It contains all your financial data in plaintext.**

#### Step 4 — Open the New App

1. Download the latest `Local_Financial_Vault.html` from this repository.
2. Open it in your browser. **Do not open your old `.lfv` file with this version** — it will fail with a format error, which is expected.

#### Step 5 — Create a New Vault

1. Click **＋ Create Vault**.
2. Enter a vault name and a strong password (12+ characters). You may use the same password as your V1 vault — the V2 format's stronger KDF provides better protection regardless.
3. Click **Create**.

#### Step 6 — Re-Enter Your Data

The JSON export from Step 3 is a plaintext reference. Use it to manually re-enter your data into the new vault's categories.

> **Why not auto-import?**
> Automatic JSON import is not currently implemented in the app. Manual re-entry takes a few minutes for most vaults and avoids any ambiguity about which fields map where, particularly for custom notes fields. If you have a very large vault, re-entry by category (Bank Accounts first, then Cards, etc.) is the most efficient approach.

Note that the current schemas have changed since V1 — see [Schema Changes](#schema-changes-within-v2) below for details on what fields exist in each category.

#### Step 7 — Save the New Vault

1. Click **💾 Save Vault**.
2. If this is a new vault, you will be prompted to name the file.
3. A V2-format `.lfv` file will be saved. You can verify it is V2 by opening it in a text editor and checking for `"magic": "LOCALFINANCIALVAULT-V2"`.

#### Step 8 — Verify and Clean Up

1. Lock the vault (**🔒 Lock / Clear**) and re-open it to confirm everything decrypts correctly.
2. Check a few entries against your JSON export to confirm the data transferred correctly.
3. Once satisfied, **securely delete**:
   - The V1 `.lfv` file (it is no longer needed and uses a weaker scheme).
   - The plaintext JSON export from Step 3 (it contains all your financial data unencrypted).
   - The old `Local_Financial_Vault.html` (V1 build), unless you are keeping it for archival purposes.

---

### Troubleshooting (V1 → V2)

**"Unable to decrypt file. Check password or file integrity. Magic header mismatch."** You are trying to open a V1 file with the V2 app. Use the V1 build of the app to open it, then follow the migration steps above.

**"Unable to decrypt file. Check password or file integrity. MAC verification failed."** Either the password is wrong, or the file has been corrupted. If you are certain the password is correct, check that the file was not partially truncated during a previous failed save.

**I lost my old copy of `Local_Financial_Vault.html`.** Retrieve it from the Git history as described in Step 1. The last V1 commit can be identified by whether `Local_Financial_Vault.html` in that commit contains `AES-GCM` (V1 marker) or `LOCALFINANCIALVAULT-V2` (V2 marker).

**My V1 file is a raw Base64 string but the old app also won't open it.** If you previously used the File System Access API picker (Chromium browsers), try the standard file input instead, or vice versa.

---

## Part 2 — Within-V2 Schema Migrations (Automatic)

V2 vaults saved with older versions of the app may contain fields that have since been renamed, removed, or restructured. The `migrateVault()` function runs automatically every time a vault is decrypted, upgrading these fields before the data is rendered. **No user action is required.**

### What `migrateVault()` Does

**Credit Cards:**
- If `expMonth` and/or `expYear` are present (old sub-fields), they are merged into `expDate` in `MM/YY` format, then deleted.
- If `cardType` is missing, it is set to `""` (empty string, shown as "-- Select --" in the form).
- If `creditLimit` is present and `balance` is absent, `creditLimit` is renamed to `balance` and then deleted. (This was a field rename in an early V2 build.)

**Crypto:**
- `purchasePrice` is deleted if present (field removed from the schema).
- If `walletId` is missing, it is set to `""`.

**Stocks:**
- If `value` is missing, it is computed from `shares × currentPrice` (old fields) if both are present; otherwise it is set to `""`.
- `purchasePrice` and `currentPrice` are deleted after migration.
- `shares` is kept as a plain text field.

### Schema Changes Within V2

The table below shows the current (2.1.0) schema for each category:

| Category | Current Fields |
|---|---|
| **Bank Accounts** | institution, accountType, accountNumber, routingNumber, balance, notes |
| **Credit Cards** | issuer, cardType (select), cardNumber, expDate (MM/YY), cvv, pin, balance, notes, cardLimit |
| **Retirement Accounts** | institution, accountType, accountNumber, currentBalance, notes |
| **Stocks** | ticker, companyName, shares (text), value, account (brokerage), notes |
| **Crypto** | coin, symbol, quantity, currentPrice (value $), wallet, walletId, notes |
| **Other Assets** | category, description, estimatedValue, loanAmount, notes *(+ computed: Net Equity)* |
| **Debts** | debtType, lender, accountNumber, balance, interestRate, minPayment, notes |

---

## Security Note on the Plaintext JSON Export

The JSON export file produced during V1→V2 migration contains **all your financial data in plaintext** — account numbers, balances, PINs, CVVs, and notes. Treat it with the same care as your vault password:

- Do not leave it in your Downloads folder.
- Do not upload it to cloud storage, email it to yourself, or copy it to an unencrypted USB drive.
- Delete it immediately once migration is complete, using your OS's secure delete option if available (e.g. `srm` on macOS, `shred` on Linux, or empty the Recycle Bin on Windows with a follow-up drive wipe if you are particularly cautious).

---

## Format Version Summary

| Format | Magic header | App version | KDF | Can open with current app? |
|---|---|---|---|---|
| V1 | *(none)* | 1.0 | PBKDF2-SHA-256, 100k | ❌ No |
| V2 | `LOCALFINANCIALVAULT-V2` | 2.0+ | PBKDF2-SHA-512, 1M | ✅ Yes (auto-migrated) |
