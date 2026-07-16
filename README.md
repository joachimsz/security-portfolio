# Security Portfolio — Joachim Szamborski

**Solana & Token-2022 security researcher.** Founder of [Softseco](https://github.com/softseco).

I build and break Solana programs. My focus is **Token-2022** — Transfer Hooks, Confidential
Transfers, and the extension surface — plus **Anchor** account validation and privilege
boundaries, and Solana's transaction-level constraints. I write the tooling I audit, so every
finding below ships with a reproducible proof of concept and a public fix.

## Selected findings

| # | Finding | Severity | Target | Status |
|---|---------|:--------:|--------|:------:|
| [01](findings/01-sentinel-missing-authority-check.md) | Missing authority check lets any signer rewrite a mint's compliance policy | **High** | Sentinel (Anchor) | Fixed |
| [02](findings/02-confidential-transfer-oversized-transaction.md) | Confidential transfer builds an oversized, always-failing transaction | **Medium** | confidential-sdk (Rust) | Fixed |
| [03](findings/03-confidential-transfer-amount-over-maximum.md) | Transfer accepts an amount above the 2^48−1 maximum | **Low** | confidential-sdk (TS) | Fixed |

These come from internal security reviews (self-audits) of my own open-source projects; each links
to the public fix. The list grows as I publish audit-competition and client findings.

## Open-source

- **[confidential-sdk](https://github.com/softseco/confidential-sdk)** — SPL Token-2022 Confidential
  Transfers SDK (TypeScript + Rust), including auditor selective disclosure. Encrypted balances and
  private transfers via ElGamal/AES and three zero-knowledge proofs verified through context-state
  accounts.
- **[Sentinel](https://github.com/softseco/sentinel)** — programmable compliance engine for
  Token-2022 tokenized assets, built on the Transfer Hook extension: per-mint allowlist, blocklist,
  and per-transfer limit enforced on-chain for every transfer.

## What I audit

- **Token-2022 extensions** — Transfer Hooks, Confidential Transfers, and their client-side flows.
- **Anchor programs** — account validation, `has_one` / `Signer` / PDA-seed constraints, and
  privilege boundaries.
- **Solana runtime constraints** — transaction size, compute budget, CPI and account-resolution
  correctness.

## Contact

- **GitHub** — [github.com/joachimsz](https://github.com/joachimsz)
- **Email** — hello@softseco.com
- **X** — [@joachimsz_](https://x.com/joachimsz_)
- **Cantina / Sherlock** — same handle (`joachimsz_`), profiles added as I enter audit contests

---

<sub>Severity model — **High:** direct on-chain impact, or full subversion of the program's security
model. **Medium:** core functionality broken or materially degraded, no direct fund loss or malicious
path. **Low:** robustness / correctness weakness with limited impact.</sub>
