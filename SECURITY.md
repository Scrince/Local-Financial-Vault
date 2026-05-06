# Security Policy

## 🔐 Overview

Local Financial Vault is designed with a strict security model:  
all data is stored **locally**, **encrypted**, and **never** transmitted over a network.  
Maintaining this security posture is the highest priority for the project.

This document outlines how to report vulnerabilities, what types of issues are considered security‑relevant, and the expectations for contributors.

---

## 📣 Reporting a Vulnerability

If you discover a security issue:

1. **Do not** open a public GitHub issue.
2. Contact the maintainer privately using GitHub’s **Security Advisory** feature or direct email (if listed).
3. Provide:
   - A clear description of the issue  
   - Steps to reproduce  
   - Potential impact  
   - Any suggested remediation  

You will receive a response as quickly as possible.

---

## 🛡 Supported Security Principles

Local Financial Vault adheres to the following core security principles:

### 1. **Fully Offline Operation**
- No network requests  
- No external APIs  
- No CDNs or remote scripts  
- No telemetry or analytics  

### 2. **Strong Cryptography**
- PBKDF2‑SHA256 (100,000 iterations) for key derivation  
- AES‑GCM for authenticated encryption  
- Random salt and IV per vault  
- No plaintext vault data is ever written to disk  

### 3. **Local‑Only Data Storage**
- All data remains in memory until encrypted and exported  
- No browser storage (localStorage, sessionStorage, IndexedDB) is used for sensitive data  
- In‑memory data is cleared on lock, reload, or tab close  

### 4. **Minimal Attack Surface**
- Single‑page application  
- No dependencies  
- No build system  
- No external libraries (crypto, UI, charting, etc.)  

---

## 🚫 Out‑of‑Scope / Will Not Be Accepted

To preserve the security model, the following changes will **not** be accepted:

- Adding any network communication  
- Integrating external libraries or frameworks  
- Weakening or replacing the cryptographic model  
- Storing sensitive data in browser storage  
- Adding cloud sync, remote backups, or multi‑user features  
- Introducing server‑side components  

---

## 🧪 Security Testing Expectations

Before submitting changes, contributors should verify:

- Vault creation, encryption, and decryption work correctly  
- No sensitive data appears in logs, console output, or browser storage  
- No network requests are made  
- Autolock clears all in‑memory data  
- Charts and UI features do not leak sensitive information  
- Error messages do not reveal cryptographic details  

---

## 🔄 Disclosure Policy

- Vulnerabilities will be acknowledged privately.  
- Fixes will be developed and tested before public disclosure.  
- Security advisories will be published when appropriate.  

---

## 🙏 Thank You

Security is foundational to Local Financial Vault.  
Thank you for helping keep the project safe, private, and trustworthy.
