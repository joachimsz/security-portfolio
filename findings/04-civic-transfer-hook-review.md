# 04 · Security review — Civic Pass Token-2022 transfer hook

|              |                                                                              |
|--------------|------------------------------------------------------------------------------|
| **Type**     | External open-source security review                                         |
| **Target**   | [`civicteam/token-extensions-transfer-hook`](https://github.com/civicteam/token-extensions-transfer-hook) @ `b6cc715` (2024-03-18) |
| **Scope**    | `programs/civic-transfer-hook` — native Solana program (`Execute` + `InitializeExtraAccountMetas`) |
| **Result**   | **No critical or high-severity issues.** 5 Low / Informational observations. |
| **Method**   | Manual review against my [Token-2022 Security Checklist](../token-2022-security-checklist.md) |

## What the program does

A Token-2022 **Transfer Hook** that gates transfers on the **recipient** holding a valid Civic Pass
("gateway token") from a configured gatekeeper network. On every transfer, Token-2022 CPIs into
`Execute`, which resolves and verifies the destination's gateway token; a missing or invalid pass
makes the transfer fail.

## What the program gets right

The classic transfer-hook pitfalls are handled correctly — worth stating explicitly, because these
are exactly where hooks usually break:

- **Mint-scoped validation PDA.** The `ExtraAccountMetaList` address is derived from
  `(mint, program_id)` and checked, so a foreign mint's validation data cannot be substituted.
- **`Execute`-only invariant.** `check_token_account_is_transferring` on both source and destination
  rejects calls outside a real Token-2022 transfer (the `transferring` flag is only set by the token
  program mid-transfer).
- **No account substitution.** `ExtraAccountMetaList::check_account_infos` validates the resolved
  extra accounts; the gateway **program** is pinned to a hardcoded `GATEWAY_PROGRAM_ID`, and the
  gateway-token account must be the exact PDA derived from the destination account + network.
- **Privileged init gated.** `InitializeExtraAccountMetas` requires the mint authority to sign and to
  match the mint's stored `mint_authority`.
- **Fail-closed.** No valid gateway token ⇒ `verify_gateway_token_account_info` returns an error and
  the transfer reverts.

## Observations

### O-1 · Low — Unpinned / stale dependencies
**Location:** `programs/civic-transfer-hook/Cargo.toml`
The SPL crates are `git` dependencies on `solana-labs/solana-program-library` **without a `rev`/`tag`**,
and that repository is **archived** (SPL moved to the `solana-program` org). `solana-program` is pinned
to `1.17.16` (old). Unpinned git deps make builds non-reproducible and can silently pull an
incompatible revision; an old `solana-program` may miss later fixes.
**Recommendation:** pin an exact `rev`/`tag`, migrate to the maintained `solana-program` org crates,
and bump `solana-program`.

### O-2 · Informational — Hardcoded gateway-token seed index
**Location:** `programs/civic-transfer-hook/src/processor.rs` (the `[0; 8]` literal seed)
The gateway-token PDA is derived with a hardcoded index of `0`, so only the holder's **index-0** pass is
ever checked. A holder whose active pass sits at a non-zero index would be denied (fail-closed, so not a
bypass — a functionality/flexibility limit).
**Recommendation:** make the index configurable via the validation data or the instruction.

### O-3 · Informational — No explicit re-initialization guard
**Location:** `process_initialize_extra_account_metas`
Re-init is prevented only implicitly (the system `allocate` fails on an already-allocated account). An
explicit assertion that the account is uninitialized would be clearer and more defensive.
**Recommendation:** assert the validation account is empty before allocating.

### O-4 · Informational — Trust assumption: pass bound to the token account, not the wallet
**Location:** `process_execute` (verification uses the destination **token account** as the owner key)
The gateway token is associated with the destination **token account**, relying on an external
(on-/off-chain) service to maintain that association. The on-chain program cannot enforce its
correctness — a trust assumption integrators must understand.
**Recommendation:** document this prominently in the integration guide.

### O-5 · Informational — Only the recipient is gated
The hook verifies the **destination**; the sender is not gated. This matches the Civic model (control
who may *receive/hold*), but is a scope property worth stating so integrators don't assume sender
gating.
**Recommendation:** state the threat model explicitly in the README.

## Scope & assumptions

- Reviewed **only** the on-chain program; the TypeScript SDK, CLI, and demo app were out of scope.
- Correctness of account resolution (`spl-tlv-account-resolution`, `spl-transfer-hook-interface`) and
  of gateway verification (`solana-gateway`) is assumed — those are upstream dependencies, not this
  program.
- No exploitable vulnerability was found, so there is nothing to embargo; the hardening items (O-1, O-3)
  are suitable for a friendly issue/PR to the maintainers.

## References

- Target: `civicteam/token-extensions-transfer-hook` @ `b6cc715`
- Checklist: [Token-2022 Security Checklist](../token-2022-security-checklist.md)

---

*By [Joachim Szamborski](../README.md) — Solana / Token-2022 security · founder, Softseco · `hello@softseco.com`*
