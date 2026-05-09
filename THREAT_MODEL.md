# Threat Model

This document describes the security goals, trust boundaries, attacker profiles, and explicit non-goals of Local Financial Vault. It is intended to help users make informed decisions about whether this tool is appropriate for their situation, and to guide contributors evaluating the security implications of proposed changes.

---

## What This Tool Is

Local Financial Vault is a **single-page offline web application** that stores encrypted financial data in a file on the user's own device. There is no server, no account, no sync service, and no network traffic of any kind. The entire application — UI, cryptography, and data — exists in one HTML file.

The fundamental security contract is:

> **A `.lfv` vault file, obtained by an attacker who does not know the password (and does not possess the keyfile, if one is configured), should reveal nothing about its contents.**

Everything in this threat model flows from that contract.

---

## Assets Being Protected

| Asset | Description | Sensitivity |
|---|---|---|
| **Account numbers** | Bank, credit card, brokerage, crypto wallet identifiers | High |
| **Authentication credentials** | PINs, CVVs, passwords stored in notes | Critical |
| **Financial balances** | Account balances, debt amounts, net worth | High |
| **Financial history** | Snapshots of net worth over time | Medium |
| **Vault password** | The master password used to derive encryption keys | Critical |
| **Keyfile** | The second-factor file mixed into key derivation (if used) | Critical |
| **Existence of the vault** | The fact that a vault file exists on the device | Low–Medium |

---

## Trust Boundaries

```
+-------------------------------------------------------------+
|                        USER'S DEVICE                        |
|                                                             |
|  +------------------------------------------------------+  |
|  |                   BROWSER SANDBOX                    |  |
|  |                                                      |  |
|  |   +-----------------------------------------------+ |  |
|  |   |         LocalFinancialVault.html              | |  |
|  |   |                                               | |  |
|  |   |   In-memory vault (decrypted JSON)            | |  |
|  |   |   In-memory password                          | |  |
|  |   |   In-memory keyfile bytes (if used)           | |  |
|  |   |   SubtleCrypto (browser-native)               | |  |
|  |   +-----------------------------------------------+ |  |
|  |                          |                           |  |
|  |              File System Access API /                |  |
|  |              <a download> blob URL                   |  |
|  |                          |                           |  |
|  +--------------------------|---------------------------+  |
|                             |                              |
|              +--------------v--------------+              |
|              |    .lfv file on disk        |              |
|              |    (encrypted at rest)      |              |
|              +-----------------------------+              |
|                                                           |
|              +--------------v--------------+              |
|              |  .key / .enc.key on disk    |              |
|              |  (keyfile, if used)         |              |
|              +-----------------------------+              |
|                                                           |
+-------------------------------------------------------------+

No network boundary exists. The application makes zero outbound connections.
```

The only data that crosses a meaningful boundary is the `.lfv` file (always encrypted) and the keyfile (raw or encrypted). Both are written by the browser to the local file system only.

---

## Attacker Profiles

### Attacker A — Offline File Thief (No Keyfile)

**Scenario:** An attacker obtains a copy of a `.lfv` file from a vault that was created without a keyfile. This might happen through physical access to the device, theft of a backup drive, access to cloud storage where the user manually uploaded the file, or access to a shared file system.

**What the attacker has:** The `.lfv` file. Nothing else.

**Goal:** Recover plaintext financial data or the vault password.

**Mitigations:**
- The file is encrypted end-to-end. The only plaintext fields are the cryptographic envelope (algorithm identifiers, salt, nonce, MAC, `has_keyfile` flag) — no financial data is exposed.
- Key derivation uses PBKDF2-HMAC-SHA-512 with 1,000,000 iterations. On modern consumer hardware, this limits brute-force attempts to roughly **50–200 password guesses per second** — making short or common passwords the primary remaining vulnerability.
- The 256-bit salt prevents precomputed (rainbow table) attacks. Every file has a unique salt even if the same password is reused across files.
- The HMAC-SHA-512 MAC prevents the attacker from determining whether a guess is "close" — a wrong password produces a MAC failure with no partial information leakage.
- The HMAC-SHA-512-CTR cipher has no padding and no oracle surface. There are no ciphertext-modification attacks analogous to padding oracle or bit-flipping attacks against CBC.

**Residual risk:** A weak or short password. The KDF is intentionally slow, but a password of fewer than ~6 random words or ~12 random characters can be brute-forced in a reasonable timeframe by a motivated attacker with GPU resources.

**User guidance:** Use a randomly generated passphrase of at least 6 words (e.g. from a password manager) or a random 20+ character string. Do not use biographical information, dictionary words, or any password reused elsewhere. Consider enabling keyfile protection for a second factor.

---

### Attacker A2 — Offline File Thief (With Keyfile, Vault File Only)

**Scenario:** An attacker obtains a copy of a `.lfv` file from a vault that was created with a keyfile, but does not obtain the keyfile itself (because it is stored separately).

**What the attacker has:** The `.lfv` file. The `has_keyfile: true` flag tells them a keyfile is required.

**Goal:** Recover plaintext financial data without the keyfile.

**Mitigations:**
- The keyfile contributes 512 bits of entropy (64 random bytes via `crypto.getRandomValues`) to the derived key via `HMAC-SHA512(keyfileBytes, UTF8(password))`. Without the keyfile, no amount of password guessing can reproduce the effective password — the search space is 2^512 regardless of actual password strength.
- Even an attacker who somehow knows the exact vault password cannot decrypt the vault without the keyfile.

**Residual risk:** None practical. This scenario is computationally unbreakable given correct keyfile separation.

---

### Attacker A3 — Offline File Thief (Vault File AND Keyfile)

**Scenario:** An attacker obtains both the `.lfv` file and the keyfile — for example, because both were stored in the same location (same folder, same cloud sync, same backup).

**What the attacker has:** The vault file and the keyfile.

**Goal:** Recover plaintext financial data.

**Mitigations:**
- With the keyfile in hand, decryption attempts are still gated by the vault password and the 1,000,000-iteration KDF. The attacker must still brute-force the password at 50–200 guesses/second.
- If an encrypted keyfile (`.enc.key`) is used, the attacker must also brute-force the separate keyfile password (200,000-iteration KDF) to extract the raw key bytes before they can attempt vault decryption.

**Residual risk:** Reduces to password strength, same as the no-keyfile scenario for a raw keyfile. Slightly better for an encrypted keyfile (two passwords to crack). The security benefit of the keyfile is fully realized only when it is stored separately from the vault.

**User guidance:** Store the keyfile on a different physical medium than the vault file — a USB drive, a printed QR code, a separate cloud provider. Never keep both in the same folder or backup.

---

### Attacker B — Network Passive Observer

**Scenario:** An attacker monitors the user's network traffic — e.g. on a public Wi-Fi network, through an ISP, or via a compromised router.

**What the attacker has:** The ability to observe all HTTP/HTTPS traffic from the user's device.

**Goal:** Intercept vault data or credentials.

**Mitigations:**
- The application makes **zero network requests** after initial load. There is no sync, no telemetry, no CDN dependency, no external script, no web font request — nothing.
- Once the HTML file is on the user's device, an air-gapped device could use it indefinitely with no network access.
- Vault save/open operations use local browser APIs only (File System Access API or `<a download>`), not network calls.

**Residual risk:** None in normal operation. The only network exposure is the initial download of `LocalFinancialVault.html` from GitHub, which is served over HTTPS. An attacker who can MITM that connection could substitute a malicious HTML file — but this is a supply-chain attack, not a network interception attack against vault data. See Attacker E.

---

### Attacker C — Malicious Browser Extension

**Scenario:** The user has a browser extension installed that has broad page permissions (`<all_urls>` or permission to the page origin). The extension is malicious or has been compromised.

**What the attacker has:** JavaScript execution context within the page, access to the DOM, and the ability to read JavaScript variables.

**Goal:** Steal the decrypted vault contents, the in-memory password, or the in-memory keyfile bytes.

**Mitigations:**
- None. A browser extension with page-level access can read any JavaScript variable, intercept any function call, and observe any DOM value. The vault password, keyfile bytes, and decrypted vault object are all held in memory during an active session.

**Residual risk:** Complete vault and password compromise if a malicious extension is present. The keyfile bytes are also exposed if a keyfile is in use.

**User guidance:** Run Local Financial Vault in a dedicated browser profile with no extensions installed, or in a browser where extensions are disabled for that session. Do not install extensions you do not trust from maintainers you cannot verify.

---

### Attacker D — Compromised Operating System or Hardware

**Scenario:** The user's device has malware, a rootkit, a keylogger, or compromised firmware installed. Alternatively, the device is subject to physical memory forensics (e.g. cold boot attack).

**What the attacker has:** Arbitrary code execution on the host OS, or access to RAM contents.

**Goal:** Steal the vault password (via keylogger), the decrypted vault (via memory dump), the keyfile bytes (via memory dump or file system access), or the encrypted file.

**Mitigations:**
- None. Client-side encryption cannot protect against a compromised host. The password is typed via the keyboard (visible to keyloggers), the decrypted vault and keyfile bytes exist in browser memory (visible to memory forensics tools), and the encrypted file exists on disk (accessible to any process with file system access).

**Residual risk:** Complete compromise. This tool is not designed to defend against an attacker with OS-level access.

**User guidance:** Maintain good device hygiene — keep the OS patched, run reputable antivirus/EDR tools, avoid running the vault on a device you do not fully control (shared computers, devices with unknown software, borrowed laptops). Lock the vault immediately after use (`🔒 Lock / Clear`) to minimize the window during which decrypted data and keyfile bytes are in memory.

---

### Attacker E — Supply Chain / Tampered File

**Scenario:** An attacker substitutes a malicious version of `LocalFinancialVault.html` — either by compromising the GitHub repository, performing a man-in-the-middle attack on the download, or directly replacing the file on the user's device.

**What the attacker has:** The ability to serve or place a modified version of the application.

**Goal:** Exfiltrate vault data, credentials, or keyfile bytes; establish persistence.

**Mitigations:**
- The project is open source. The entire application is a single readable HTML file — any technically capable user can audit it in a text editor before running it.
- GitHub serves downloads over HTTPS, protecting against transit MITM in ordinary circumstances.
- There are no minified or obfuscated code paths. All cryptographic logic is in clearly commented, human-readable JavaScript.

**Residual risk:** A user who downloads and runs the file without inspection cannot verify it hasn't been tampered with at the source. There is no code-signing, no checksum verification workflow, and no reproducible build process (not applicable to a single HTML file with no build step).

**Planned mitigations:** Publishing SHA-256 checksums of released builds alongside each GitHub Release tag would allow users to verify file integrity independently. This is a planned future improvement.

**User guidance:** Download `LocalFinancialVault.html` only from the official repository (`https://github.com/Scrince/Local-Financial-Vault`). Do not run copies received via email, messaging apps, or file sharing services unless you can verify the SHA-256 hash against a published checksum.

---

### Attacker F — Physical Access (Unlocked Session)

**Scenario:** An attacker gains physical access to the user's device while the vault is open and unlocked in the browser — e.g. the user steps away from their desk.

**What the attacker has:** Physical access to an unlocked device with an active vault session.

**Goal:** Read, copy, or exfiltrate vault contents; extract the keyfile bytes from memory.

**Mitigations:**
- Auto-lock triggers after **10 minutes of inactivity**, clearing all in-memory vault data, the password, and the keyfile bytes.
- The user can manually lock at any time via **🔒 Lock / Clear**.

**Residual risk:** Within the 10-minute inactivity window, all vault data and keyfile bytes are visible. Locking the OS screen does not lock the vault — the auto-lock is DOM-event-driven and will not fire until the browser receives input events after the screen is unlocked, restarting the 10-minute countdown.

**User guidance:** Lock the vault manually (`🔒 Lock / Clear`) before stepping away. Do not rely on the auto-lock for physical security — it is designed as a convenience backstop, not a physical access control.

---

## Non-Goals

The following are explicitly outside the security model of this tool:

- **Protection against a compromised OS, kernel, or hardware** — No client-side application can provide this.
- **Protection against malicious browser extensions with page access** — The browser extension trust model is controlled by the browser and the user, not by this application.
- **Keyfile recovery** — If the keyfile is lost, the vault is permanently unrecoverable. There is no escrow, recovery code, or backup mechanism.
- **Multi-user access control** — There is one vault, one password (plus optional keyfile), one access level. There is no concept of read-only access, shared access, or role separation.
- **Remote wipe or vault revocation** — If a device is stolen, there is no mechanism to remotely invalidate or destroy the vault file. Password strength and keyfile separation are the only post-theft protections.
- **Anonymity or metadata resistance beyond the file itself** — The operating system records file access times, browser history may record that the HTML file was opened, and the file name itself may reveal its purpose. This tool encrypts the file's contents, not its existence.
- **Protection against the user themselves** — The vault is fully accessible to anyone who knows the password and possesses the keyfile. There is no duress password, no hidden volume, and no plausible deniability mechanism.
- **Audit logging or tamper evidence of in-session changes** — Actions taken within an open vault session (add, edit, delete, export) are not logged anywhere.

---

## Cryptographic Assumptions

The security of this tool rests on the following cryptographic assumptions:

| Assumption | Justification |
|---|---|
| HMAC-SHA-512 is a secure PRF | Widely accepted; no known breaks |
| PBKDF2-HMAC-SHA-512 is a memory-hard-enough KDF for interactive logins | Acceptable for this use case; Argon2 would be stronger but is not available in SubtleCrypto |
| The browser's `SubtleCrypto` implementation is correct and uncompromised | Required for any browser-based cryptography |
| `crypto.getRandomValues` produces cryptographically secure random output | Required for salt, nonce, and keyfile generation |
| A 256-bit random nonce has negligible collision probability over any realistic number of saves | P(collision) ≈ saves² / 2²⁵⁶ ≈ 0 for any human-scale usage |
| 64 bytes of `getRandomValues` output provides 512 bits of entropy for keyfile material | Standard assumption for a CSPRNG |

If any of these assumptions are violated — for example, by a browser cryptography bug or a discovered HMAC-SHA-512 weakness — the security guarantees of the V2 format would need to be reassessed.

---

## Revision History

| Date | Change |
|---|---|
| 2025-05-06 | Initial threat model, written for V2 format |
| 2026-05-08 | Added keyfile attacker profiles (A2, A3); updated A, C, D, F to reflect in-memory keyfile bytes; updated trust boundary diagram; updated non-goals; updated cryptographic assumptions table; updated security contract |
