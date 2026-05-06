# Local Financial Vault
A secure, fully offline, encrypted web‑based vault for managing all your financial accounts, assets, debts, and net‑worth visualizations in one place.
Local Financial Vault
Local Financial Vault is a secure, fully offline, single‑page web application for managing all your financial accounts, assets, debts, and net‑worth history in one encrypted vault file. All data is stored locally, encrypted with modern cryptography, and never leaves your device.
Features
🔐 Secure, Encrypted Vault
Create a new vault protected by a master password (minimum 12 characters).
Vaults are encrypted using PBKDF2‑SHA256 (100k iterations) and AES‑GCM.
No plaintext financial data is ever written to disk.
Fully offline — no servers, no cloud, no external APIs.
📁 Vault Lifecycle
Create, open, and save encrypted vault files.
Import/export vaults as Base64‑encoded encrypted files.
Opening a vault replaces all in‑memory data.
Automatic lock after 10 minutes of inactivity.
🧾 Structured Financial Data
Manage entries across all major financial categories:
Bank Accounts
Credit Cards
Retirement Accounts
Stocks (manual price entry)
Cryptocurrency (manual price entry)
Other Assets
Debts
Each category includes standardized fields (institution, balances, account numbers, notes, etc.) with numeric validation where appropriate.
Operations supported for every category:
Add
Edit
Delete
Duplicate
Table‑based viewing
🔍 Global Search & Filtering
Case‑insensitive, substring search across all fields.
Filter by category for focused viewing.
📊 Net Worth & Financial Calculations
The app automatically computes:
Total Assets
Total Debts
Net Worth
Assets include bank balances, retirement accounts, stocks, crypto, and other assets. Debts include all loan/mortgage balances.
A summary panel displays all totals in real time.
📈 Snapshots & Visualizations
Record financial snapshots over time and view:
Net worth over time (line chart)
Assets vs. debts (bar or pie chart)
Assets by category (breakdown chart)
Charts are rendered using HTML Canvas + custom JavaScript (no external libraries).
How It Works
🔒 Encryption Model
Password → PBKDF2‑SHA256 → 256‑bit key
Random salt and IV per vault
AES‑GCM encryption
Salt + IV + ciphertext stored together (Base64)
📦 Data Model
Vault JSON structure (before encryption):
Code
{
  "metadata": {
    "version": "1.0",
    "createdAt": "...",
    "lastModifiedAt": "..."
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
      "timestamp": "...",
      "totalAssets": 0,
      "totalDebts": 0,
      "netWorth": 0
    }
  ]
}
Installation & Usage
1. Clone the repository
Code
git clone https://github.com/yourusername/local-financial-vault.git
2. Open the app
Simply open index.html in any modern browser:
Chrome
Firefox
Edge
Safari
No build step, no server, no dependencies.
3. Create or open a vault
Choose New Vault to start fresh.
Choose Open Vault to load an existing encrypted file.
All data stays local to your machine.
Non‑Functional Requirements
Fully offline; no network calls.
Works with thousands of records while staying responsive.
Dark‑mode, tabbed UI with simple tables.
Compatible with modern desktop browsers.
