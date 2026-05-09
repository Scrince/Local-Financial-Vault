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
[Optional] effectivePassword = HMAC-SHA512(keyfileBytes, UTF8(password))
           (if no keyfile, effectivePassword = password)

PBKDF2-HMAC-SHA512(effectivePassword, salt, iterations=1_000_000, dkLen=64)
  → 512-bit IKM (input key material)

encKey  = HMAC-SHA512(IKM, "encryption")       // 512-bit encryption key
macKey  = HMAC-SHA512(IKM, "authentication")   // 512-bit authentication key
```

- Salt is 256-bit (32 bytes), randomly generated per save using `crypto.getRandomValues`.
- Keys are domain-separated to prevent cross-context key reuse.
- When a keyfile is present, the password and keyfile are cryptographically bound via HMAC before KDF. Neither factor alone provides any information about the derived key material. The effective entropy floor is 512 bits (the HMAC-SHA512 output) regardless of password strength.
- All primitives are implemented via the browser's native **Web Crypto API** (`SubtleCrypto`). No third-party cryptographic libraries are used.

### Encryption

```
HMAC-SHA512-CTR stream cipher:
  For each 64-byte block i:
    pad[i] = HMAC-SHA512(encKey, nonce || counter_i)
    ciphertext[i] = plaintext[i] XOR pad[i]
```

- Nonce is 256-bit (32 bytes), randomly generated per save.
- The stream cipher produces a unique keystream per (key, nonce) pair. Nonce reuse with the same key would be catastrophic — the random 256-bit nonce makes collision probability negligible.

### Authentication

```
MAC = HMAC-SHA512(macKey, magic || salt || nonce || ciphertext)
```

- The MAC covers the magic header, all key derivation parameters, and the full ciphertext.
- MAC verification occurs **before** any decryption (encrypt-then-MAC). A wrong password, wrong keyfile, or any bit flip in the file is detected immediately.
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
  "has_keyfile":    false,
  "salt_b64":       "...",
  "nonce_b64":      "...",
  "ciphertext_b64": "...",
  "mac_b64":        "..."
}
```

The `has_keyfile` flag is stored in plaintext in the envelope so the application can prompt for a keyfile before attempting decryption. It does not weaken security — an attacker who knows a vault requires a keyfile still cannot decrypt it without both the keyfile and the password.

---

## Keyfile Scheme

Keyfiles are optional second-factor files that are cryptographically mixed into the vault password before key derivation. Two formats are supported:

### Raw Keyfile (`.key`)

```json
{ "magic": "KEYFILE", "v": 1, "len": 64, "key": "<64 random bytes, base64>" }
```

64 bytes (512 bits) of output from `crypto.getRandomValues`. The raw bytes are mixed directly into the password. Protecting this file from unauthorized access is the user's responsibility — it has no encryption of its own.

### Encrypted Keyfile (`.enc.key`)

```json
{
  "magic": "KEYFILE-ENC", "v": 1,
  "kdf":   { "algo": "PBKDF2-SHA512", "i": 200000, "s": "<16-byte salt, base64>" },
  "n":     "<16-byte nonce, base64>",
  "ct":    "<encrypted key bytes, base64>",
  "mac":   "<HMAC-SHA512 integrity tag, base64>"
}
```

Uses the same construction as the vault itself — PBKDF2-SHA512 (200,000 iterations) for key derivation, HMAC-SHA512-CTR for encryption, and HMAC-SHA512 MAC for authentication — with a 4-byte counter. The MAC is verified before decryption. The keyfile password is separate from the vault password; both are required to open the vault.

### Keyfile Security Properties

- The keyfile and vault file **must be stored separately**. If both are present on the same device and an attacker obtains both, security reduces to password strength alone. The keyfile provides the strongest protection when stored on a separate physical medium (USB drive, printed QR code, separate cloud service).
- Loss of the keyfile makes the vault **permanently unrecoverable**. There is no backup mechanism built into the application. Users are warned of this at creation time.
- The encrypted keyfile's 200,000-iteration KDF is lower than the vault's 1,000,000-iteration KDF. This is appropriate because the keyfile password protects 64 bytes of random material (not a vault full of data), and the keyfile itself is stored rather than typed repeatedly. The encrypted keyfile remains resistant to brute-force when protected by a strong keyfile password.

---

## Threat Model

### What this tool protects against

- **Offline file theft** — an attacker who obtains a `.lfv` file cannot read it without the password (and keyfile, if used). The 1M-iteration KDF makes bulk brute-force attacks computationally expensive.
- **Offline file theft with keyfile** — an attacker who obtains the vault file but not the keyfile cannot make any useful decryption attempts, regardless of computational resources, because the keyfile contributes 512 bits of entropy to the derived key.
- **Ciphertext tampering** — the HMAC-SHA512 MAC detects any modification to the file.
- **Metadata leakage from the file itself** — all financial data, category names, entry counts, and field values are inside the encrypted payload. The only plaintext in the file is the cryptographic envelope (algorithm identifiers, salt, nonce, MAC, and the `has_keyfile` flag).
- **IV/nonce reuse** — fresh 256-bit nonces are generated cryptographically at random on every save.

### What this tool does NOT protect against

- **A compromised device** — if the OS, browser, or file system is under attacker control, client-side encryption cannot fully protect your data.
- **Weak passwords** — the KDF is intentionally slow, but a short or guessable password can still be brute-forced offline. Use a long, random passphrase.
- **Browser extensions** — a malicious browser extension with access to the page can read in-memory JavaScript variables, including the decrypted vault and the password. Run the vault in a trusted browser profile without untrusted extensions.
- **Physical access attacks** — screen recording, over-the-shoulder observation, or memory forensics after the session.
- **Auto-lock limitations** — the 10-minute auto-lock timer is based on DOM events. A suspended OS (e.g. laptop lid close) will not trigger the lock until the timer fires after resume.
- **Keyfile loss** — there is no recovery mechanism if the keyfile is lost. The warning displayed at creation time is the only safeguard.

---

## Export Redaction

The Export modal includes optional redaction of sensitive fields:

- **Bank account numbers** — truncated to last 4 digits in CSV, JSON, and PDF output.
- **Card numbers** — truncated to last 4 digits.
- **Card CVV & PIN** — replaced with `•••`.

Redaction is applied at export time only. The vault file itself always stores the full unredacted values. Redacted exports are suitable for sharing or printing without exposing full account credentials.

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
7. **Two-factor binding.** When a keyfile is used, password and keyfile are cryptographically bound via HMAC before KDF — not simply concatenated. Either factor alone is cryptographically useless without the other.
