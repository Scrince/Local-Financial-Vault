# Contributing to Local Financial Vault

Thank you for your interest in contributing to **Local Financial Vault**.  
This project is designed to be fully offline, secure, and dependency‑free, so contributions must follow strict guidelines to maintain those principles.

---

## 📌 Core Principles

All contributions **must** preserve the following:

- **100% offline operation**  
  No network requests, no external APIs, no CDNs.

- **No external libraries**  
  Only vanilla HTML, CSS, and JavaScript.

- **Strong cryptography model**  
  PBKDF2‑SHA256 (100k iterations) and AES‑GCM must remain the encryption standard.

- **Local‑only data storage**  
  No cloud sync, no remote storage, no telemetry.

- **Security first**  
  No plaintext persistence, no leaking sensitive data to logs, and no weakening of the crypto model.

---

## 🛠 Development Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/local-financial-vault.git
