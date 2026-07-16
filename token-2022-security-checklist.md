# Token-2022 (Token Extensions) — Security Checklist

A practical, auditor's checklist for **SPL Token-2022** programs and integrations on Solana. It
covers the cross-cutting traps of the extension model, a per-extension review list, and separate
lists for **hook authors** and for **protocols that accept** Token-2022 assets.

Maintained by **[Joachim Szamborski](./README.md)** — Solana / Token-2022 security, founder of
Softseco. Companion to my [security portfolio](./README.md); several items map to real findings I've
published.

> Use it as a review aid, not a guarantee. Extensions **interact** — always reason about the specific
> combination a given mint enables, not each extension in isolation.

---

## 0. First: enumerate the mint's extensions

- [ ] Unpack the mint with `StateWithExtensions` and **list every enabled extension** before
  reasoning about behavior.
- [ ] Never infer an account's **type or contents from its data length** — extensions make length
  variable, and the classic "identify by length" check is exploitable under Token-2022.
- [ ] Resolve and **validate the owning program** (classic Token vs Token-2022). Don't hardcode a
  single program id.
- [ ] Treat any extension you haven't reviewed as **potentially adversarial** — the mint authority
  can enable behavior that breaks your assumptions.

## 1. Cross-cutting traps (any protocol touching the token)

- [ ] **Fee-on-transfer (Transfer Fee):** *amount received ≠ amount sent*. Compute the actual credited
  amount from the post-balance delta (or `transfer_checked_with_fee`); never assume the vault got the
  full amount. Account for withheld fees and the `withdraw_withheld_authority`.
- [ ] **Permanent Delegate:** the mint may carry an authority that can transfer/burn **any** account of
  that mint. A single-vault protocol can be silently drained to zero. Decide explicitly whether to
  reject such mints.
- [ ] **Default Account State (frozen) + freeze authority:** new accounts may be frozen by default, and
  the freeze authority can freeze any holder at any time → transfers fail (censorship / DoS). Handle
  both cases.
- [ ] **Non-Transferable ("soulbound"):** transfers revert. Verify your flow doesn't assume
  transferability, and check extension **interactions** (historically, non-transferable tokens could be
  moved via confidential transfer due to a missing check).
- [ ] **Confidential balances are invisible:** the token-account `amount` field does **not** include the
  confidential balance. Balance reads, pricing, and accounting must not treat `amount` as the full
  balance.
- [ ] **Interest-Bearing:** interest mints **no** new tokens — only the *UI amount* grows, the raw
  `amount` is unchanged. Don't treat UI amount as spendable; convert via `amount_to_ui_amount`.
- [ ] **Mint Close Authority:** a mint with zero supply can be **closed** and its address reused. Don't
  assume a mint address is permanent.
- [ ] **Required Memo (MemoTransfer):** transfers into such accounts fail without a preceding memo
  instruction — integrators must attach one.
- [ ] **Metadata / `*-Pointer` extensions:** a pointer (metadata / group / member) can point to an
  **arbitrary** account. Verify it points to the mint itself (self-referential) or a trusted account
  before trusting the data — otherwise it is spoofable.

## 2. Transfer Hook — a program that *uses* a hooked mint

- [ ] The mint's `transfer_hook_program_id` is the **expected** program — don't trust an arbitrary hook.
- [ ] Recognize that **every transfer can run arbitrary code**: a malicious or buggy hook can revert
  (DoS / griefing) or exhaust CPI depth. Assess the composability impact on your protocol.
- [ ] The hook cannot be used to **re-enter** your instruction (a hook that CPIs back into a transfer of
  the same mint → recursion). Token-2022 marks accounts as `transferring` during the hook; verify, don't
  assume.

## 3. Transfer Hook — writing your *own* hook (compliance / allowlist / limit)

- [ ] **`ExtraAccountMetaList` PDA seeds include the mint.** If a resolved account (e.g. an
  allow/deny-list PDA) is derived *without* the mint, an attacker creates a different mint whose list
  contains their address and passes it instead. Mint-scoped seeds are mandatory.
  *(Sentinel derives every policy/allow/block PDA as `[seed, mint, …]` for exactly this reason.)*
- [ ] Accounts resolved from **token-account owner fields** use the correct offset (owner at bytes
  `32..64`) and the correct source/destination account index.
- [ ] The hook is only reachable via Token-2022's `Execute` CPI; discriminator/`fallback` routing rejects
  anything else, and resolved accounts **cannot be substituted** by the caller.
- [ ] **Fail closed:** a missing/uninitialized policy or list entry denies (or your explicit intended
  default) — never *silently allows*. Distinguish "account absent" from "rule disabled".
- [ ] **Least privilege:** the hook reads only what it needs, holds no spendable authority, and can't be
  abused as a signing oracle.
- [ ] **No self-transfer / recursion** of the same mint from inside the hook.
- [ ] Privileged management instructions (edit policy / lists) enforce the **stored authority**
  (`has_one`), and creation is bound to the mint authority.
  *(Portfolio [finding 01](findings/01-sentinel-missing-authority-check.md): a missing `has_one` let any
  signer rewrite the policy.)*
- [ ] Compute budget & tx size: heavy hooks fit the CU and 1232-byte limits.
  *(Portfolio [finding 02](findings/02-confidential-transfer-oversized-transaction.md): oversized
  transactions silently fail.)*

## 4. Confidential Transfer — specific checks

- [ ] Approve policy is intentional: `auto_approve_new_accounts` vs authority approval matches the trust
  model.
- [ ] If an **auditor** ElGamal key is configured, disclosure works as intended and the auditor cannot
  spend.
- [ ] The three proofs (equality, ciphertext-validity, range) are verified via **context-state
  accounts**, not inline — inline proofs overflow the 1232-byte transaction limit.
  *([finding 02](findings/02-confidential-transfer-oversized-transaction.md).)*
- [ ] Amounts above the encodable maximum (`2^48 − 1`) are rejected **before** proof generation, with a
  clear error. *([finding 03](findings/03-confidential-transfer-amount-over-maximum.md).)*
- [ ] Interactions reviewed (non-transferable + confidential, transfer-fee + confidential).

## 5. General Solana checks (still apply)

- [ ] Signer & owner checks on every privileged account; no account substitution.
- [ ] PDA seeds are canonical and collision-free; `bump` validated.
- [ ] Arithmetic is checked (no silent overflow / underflow) around amounts and fees.
- [ ] CPIs target the intended program id; account infos aren't attacker-swappable.
- [ ] Account-close / rent paths leave no reinitialization or dangling-authority holes.

---

## References

- Neodyme — [*SPL Token-2022: Don't shoot yourself in the foot with extensions*](https://neodyme.io/en/blog/token-2022/)
- Offside Labs — [*Token-2022 Security Best Practices (Extensions)*](https://blog.offside.io/p/token-2022-security-best-practices-part-2)
- Solana — [*Token Extensions* docs](https://solana.com/docs/tokens/extensions) · [*Transfer Hook* guide](https://solana.com/developers/guides/token-extensions/transfer-hook)
- [`solana-program/token-2022`](https://github.com/solana-program/token-2022) — official program and its published audits (Halborn, Zellic, Trail of Bits, NCC, OtterSec, Certora)

---

*By [Joachim Szamborski](./README.md) — Solana / Token-2022 security · founder, Softseco · `hello@softseco.com`*
