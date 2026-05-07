# Contributing to Local Financial Vault

Thank you for your interest in contributing. This project is a single self-contained HTML file — there is no build system, no package manager, and no external dependencies. Keeping it that way is a core design goal.

---

## Project Philosophy

Before contributing, please understand the guiding constraints:

- **Single file.** The entire application — HTML, CSS, and JavaScript — lives in `LocalFinancialVault.html`. Pull requests that split this into multiple files will not be accepted.
- **Zero external dependencies.** No CDN links, no npm packages, no third-party scripts. All cryptography uses the native Web Crypto API (`SubtleCrypto`). All charts use plain HTML Canvas.
- **Offline-first.** The app must work with no internet connection after the initial download. Any change that introduces a network request will be rejected.
- **Security over convenience.** Cryptographic decisions are made conservatively. Do not propose weakening the KDF iteration count, downgrading hash functions, or simplifying the key derivation scheme for performance reasons.

---

## Ways to Contribute

### 🐛 Bug Reports

Open a [GitHub Issue](https://github.com/Scrince/Local-Financial-Vault/issues) with:

- A clear title and description of the problem.
- Steps to reproduce the bug.
- Expected vs. actual behavior.
- Browser name and version.
- Whether the issue affects a specific vault operation (create / open / save / export / etc.).

Do **not** include real financial data in bug reports.

### 💡 Feature Requests

Open a GitHub Issue with the `enhancement` label. Describe:

- The use case or problem you're trying to solve.
- Your proposed solution or behavior.
- Any tradeoffs or security implications you've considered.

Feature requests that require external dependencies or network access will generally not be accepted.

### 🔐 Security Issues

**Do not open a public issue for security vulnerabilities.** Follow the process in [SECURITY.md](SECURITY.md) to report privately via a GitHub Security Advisory.

### 🔧 Pull Requests

All PRs are welcome. To maximize the chance of acceptance:

1. Open an issue first for any non-trivial change so the approach can be discussed before you invest time coding.
2. Keep changes focused — one concern per PR.
3. Follow the existing code style (vanilla JS, no modules, no classes, consistent indentation).
4. Test manually in at least two browsers (one Chromium-based, one non-Chromium).
5. Update `README.md` if your change affects user-facing behavior or the encryption model.

---

## Development Setup

No build step required. Just open the file:

```bash
# Clone the repo
git clone https://github.com/Scrince/Local-Financial-Vault.git
cd Local-Financial-Vault

# Open directly in your browser
open LocalFinancialVault.html          # macOS
start LocalFinancialVault.html         # Windows
xdg-open LocalFinancialVault.html     # Linux
```

Or drag and drop `LocalFinancialVault.html` into your browser.

> **Note:** Some browser security policies restrict File System Access API features when opening an HTML file directly via `file://`. For full FSAA in-place save testing, serve the file over a local HTTP server:
>
> ```bash
> python3 -m http.server 8080
> # then open http://localhost:8080/LocalFinancialVault.html
> ```

---

## Code Style

- **Vanilla JavaScript only** — no TypeScript, no frameworks, no transpilation.
- Use `async/await` for all asynchronous operations.
- Keep functions small and named clearly.
- All cryptographic code lives in the `CRYPTO` section (clearly delimited by block comments). Crypto changes must include an explanation of the security rationale in the PR description.
- Avoid adding new global variables unless strictly necessary.
- HTML and CSS modifications should preserve the existing dark/light mode variable system (`var(--accent)`, etc.).

---

## Cryptographic Contribution Guidelines

Changes to the encryption scheme are the highest-risk category of contribution. If you are proposing a cryptographic change:

1. **Explain the motivation clearly.** Why is the current scheme insufficient for the proposed change?
2. **Link to relevant references.** RFCs, academic papers, or authoritative documentation.
3. **Do not break backward compatibility silently.** Any new format must be detectable (e.g. via the `magic` and `version` fields) so existing vault files can still be opened.
4. **Maintain encrypt-then-MAC.** The MAC must always cover the ciphertext, not the plaintext.
5. **Do not reduce the KDF iteration count.** 1,000,000 iterations is a minimum. Proposals to increase it are welcome.
6. **Preserve domain separation.** Encryption and authentication keys must remain independently derived.

Proposals that weaken the current scheme in exchange for performance will not be accepted.

---

## Commit Messages

Use concise, imperative-mood commit messages:

```
Fix: correct nonce counter encoding for large vaults
Add: duplicate entry button for all categories
Docs: update README encryption model diagram
Security: increase PBKDF2 iterations to 2M
```

Prefix with `Fix:`, `Add:`, `Remove:`, `Docs:`, `Security:`, or `Refactor:`.

---

## Licensing

By submitting a pull request, you agree that your contribution will be licensed under the [MIT License](LICENSE.md) that covers this project.
