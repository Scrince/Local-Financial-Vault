# Security Policy

## Supported Versions

Only the latest version of `LocalFinancialVault.html` in the `main` branch receives security updates. There are no versioned release branches to backport fixes to — the project is a single self-contained file.

| Version | Supported |
|---|:---:|
| V2 (current, `main`) | ✅ |
| V1 (PBKDF2-SHA256 + AES-GCM) | ❌ |

If you are using a V1 vault file (no `magic` field in the JSON, or a raw Base64 blob), please re-save it with the current version to upgrade to the V2 format.

---

## Encryption Scheme — V2

The current format (`LOCALFINANCIALVAULT-V2`) uses the following construction:

### Key Derivation

```
PBKDF2-HMAC-SHA512(password, salt, iterations=1_000_000, dkLen=64)
  → 512-bit IKM (input key material)

encKey  = HMAC-SHA512(IKM, "encryption")       // 512-bit encryption key
macKey  = HMAC-SHA512(IKM, "authentication")   // 512-bit authentication key
```

- Salt is 256-bit (32 bytes), randomly generated per save using `crypto.getRandomValues`.
- Keys are domain-separated to prevent cross-context key reuse.
- All primitives are implemented via the browser's native **Web Crypto API** (`SubtleCrypto`). No third-party cryptographic libraries are used.

### Encryption

```
HMAC-SHA512-CTR stream cipher:
  For each 64-byte block i:
    pad[i] = HMAC-SHA512(encKey, nonce ‖ counter_i)
    ciphertext[i] = plaintext[i] XOR pad[i]
```

- Nonce is 256-bit (32 bytes), randomly generated per save.
- The stream cipher produces a unique keystream per (key, nonce) pair. Nonce reuse with the same key would be catastrophic — the random 256-bit nonce makes collision probability negligible.

### Authentication

```
MAC = HMAC-SHA512(macKey, magic ‖ salt ‖ nonce ‖ ciphertext)
```

- The MAC covers the magic header, all key derivation parameters, and the full ciphertext.
- MAC verification occurs **before** any decryption (encrypt-then-MAC). A wrong password or any bit flip in the file is detected immediately.
- Comparison uses a simple XOR accumulator to mitigate timing side-channels.

### File Structure

```json
{
  "magic":          "LOCALFINANCIALVAULT-V2",
  "version":        2,
  "kdf":            "PBKDF2-HMAC-SHA512",
  "kdf_iterations": 1000000,
  "cipher":         "HMAC-SHA512-CTR",
  "mac_algo":       "HMAC-SHA512",
  "salt_b64":       "...",
  "nonce_b64":      "...",
  "ciphertext_b64": "...",
  "mac_b64":        "..."
}
```

---

## Threat Model

### What this tool protects against

- **Offline file theft** — an attacker who obtains a `.lfv` file cannot read it without the password. The 1M-iteration KDF makes bulk brute-force attacks computationally expensive.
- **Ciphertext tampering** — the HMAC-SHA512 MAC detects any modification to the file.
- **Metadata leakage from the file itself** — all financial data, category names, entry counts, and field values are inside the encrypted payload. The only plaintext in the file is the cryptographic envelope (algorithm identifiers, salt, nonce, MAC).
- **IV/nonce reuse** — fresh 256-bit nonces are generated cryptographically at random on every save.

### What this tool does NOT protect against

- **A compromised device** — if the OS, browser, or file system is under attacker control, client-side encryption cannot fully protect your data.
- **Weak passwords** — the KDF is intentionally slow, but a short or guessable password can still be brute-forced offline. Use a long, random passphrase.
- **Browser extensions** — a malicious browser extension with access to the page can read in-memory JavaScript variables, including the decrypted vault and the password. Run the vault in a trusted browser profile without untrusted extensions.
- **Physical access attacks** — screen recording, over-the-shoulder observation, or memory forensics after the session.
- **Auto-lock limitations** — the 10-minute auto-lock timer is based on DOM events. A suspended OS (e.g. laptop lid close) will not trigger the lock until the timer fires after resume.

---

## Reporting a Vulnerability

If you discover a security vulnerability — including cryptographic weaknesses, logic flaws, or any issue that could expose encrypted vault data — please report it **privately** before disclosing publicly.

**How to report:**

1. Open a [GitHub Security Advisory](https://github.com/Scrince/Local-Financial-Vault/security/advisories/new) on this repository (preferred — keeps the report private until patched).
2. Alternatively, open a GitHub Issue marked `[SECURITY]` if the issue is low-severity and does not risk exposing user data.

**Please include:**

- A clear description of the vulnerability.
- Steps to reproduce or a proof-of-concept (if applicable).
- The potential impact and affected versions.
- Any suggested mitigations or fixes, if you have them.

**Response commitment:**

- Acknowledgement within **72 hours**.
- A status update within **7 days**.
- A patch and coordinated disclosure within **30 days** of a confirmed critical vulnerability.

We appreciate responsible disclosure and will credit reporters in the changelog unless they prefer to remain anonymous.

---

## Security Design Principles

The following principles guide all cryptographic decisions in this project:

1. **No external dependencies.** All cryptography uses the browser's built-in `SubtleCrypto` API. No third-party libraries means no supply-chain risk.
2. **Encrypt-then-MAC.** Authentication always covers the ciphertext, never the plaintext.
3. **Domain separation.** Encryption and authentication keys are derived separately, preventing cross-context key confusion.
4. **Fail closed.** A MAC failure or format mismatch results in an immediate error — no partial decryption, no plaintext output.
5. **Minimal metadata.** The only unencrypted data in a `.lfv` file is the cryptographic envelope. No timestamps, entry counts, or category names are stored in plaintext.
6. **Fresh randomness per operation.** Salt and nonce are regenerated on every save, regardless of whether the password changed.
