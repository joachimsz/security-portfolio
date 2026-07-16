# 02 · Confidential transfer assembles an oversized transaction that always fails

|              |                                                            |
|--------------|------------------------------------------------------------|
| **Severity** | Medium                                                     |
| **Target**   | `softseco/confidential-sdk` · `rust/src/operations.rs` (`transfer`) |
| **Category** | Correctness / availability (SDK)                           |
| **Status**   | Fixed — `v1.0.1` (crates.io)                               |

## Summary

The Rust SDK's `transfer` built its three required zero-knowledge proofs (equality,
ciphertext-validity, range) **inline** in the transfer instruction. The resulting transaction was
~3308 bytes — well over Solana's 1232-byte packet limit — so **every real confidential transfer
failed**. Deposit, apply, and decrypt worked; the core private-transfer path did not.

## Impact

The SDK's headline feature is non-functional on-chain: an integrator following the documented flow
can move funds into a confidential balance but cannot confidentially transfer them out using the
crate. No funds are lost and there is no malicious exploit path — this is a correctness / availability
defect in client-side tooling — but it would have shipped a broken headline feature. Rated
**Medium**.

## Technical detail

Confidential transfers need three sizeable ZK proofs. Shipping them inline pushes the transaction
past the hard 1232-byte limit. The proofs must instead be verified into **context-state accounts** up
front, so the transfer instruction only references them.

## Proof of concept

A gated end-to-end integration test (create CT mint → configure two accounts → mint → deposit → apply
→ transfer 400) reproduced it deterministically. The transfer step failed with:

```text
VersionedTransaction too large: 3308 bytes (max: encoded/raw 1644/1232)
```

## Remediation

Port `transfer` to context-state proof accounts, mirroring the TypeScript SDK and the official
`spl-token` CLI: verify each proof into its own account (the range proof split across two
transactions), send a small transfer that references them, then close them to reclaim rent.

```rust
// verify each proof into a context-state account (range split across 2 txs)
token.confidential_transfer_create_context_state_account(&range, .., true).await?;
token.confidential_transfer_create_context_state_account(&equality, .., false).await?;
token.confidential_transfer_create_context_state_account(&validity, .., false).await?;

// small transfer that only references the three proof accounts
token.confidential_transfer_transfer(
    &src, &dst, &owner,
    Some(&equality), Some(&validity_acct), Some(&range),
    amount, ..,
).await?;

// close the three context-state accounts (rent back to payer)
```

After the fix the integration test passes end-to-end (sender keeps 600, recipient receives 400), and
the transfer transaction fits the packet limit.

> **Note.** Found via a test-first approach: the failing integration test was written to *prove* the
> bug before any fix, then re-run to verify the fix.

## Reference

Released in `v1.0.1`. See the
[CHANGELOG](https://github.com/softseco/confidential-sdk/blob/main/CHANGELOG.md) and
[softseco/confidential-sdk](https://github.com/softseco/confidential-sdk).
