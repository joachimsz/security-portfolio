# 01 · Missing authority check lets any signer rewrite a mint's compliance policy

|              |                                                            |
|--------------|------------------------------------------------------------|
| **Severity** | High                                                       |
| **Target**   | `softseco/sentinel` · `programs/sentinel/src/lib.rs`       |
| **Category** | Access control / privilege escalation                      |
| **Status**   | Fixed — before the `v1.0.0` release                        |

## Summary

Sentinel enforces per-mint compliance (allowlist, blocklist, transfer limit) for Token-2022 assets.
The privileged instructions that manage a mint's policy — `initialize_policy`, `update_policy`, and
the allow/block-list add & remove instructions — loaded the mint's `PolicyConfig` PDA by its seeds
only, **without binding the transaction signer to the policy's stored `authority`**. Any account that
signed the transaction was accepted as the authority.

## Impact

An arbitrary, unauthorized wallet could:

1. add itself — or a sanctioned address — to the **allowlist**, receiving a token it must not hold;
2. add any holder to the **blocklist**, censoring or freezing them out of transfers;
3. rewrite the policy via `update_policy` (disable the allowlist, or zero the transfer limit).

Because Sentinel is the compliance gate for regulated / real-world-asset tokens, this defeats the
security model of **every mint** governed by the program. It is exploitable on-chain by any signer at
negligible cost. Rated **High**.

## Technical detail

The management account structs declared `authority: Signer` and loaded `policy_config` from its PDA
seeds, but omitted the constraint tying the two together — so Anchor validated any signer:

```rust
// Vulnerable shape (pre-fix): any signer is accepted as `authority`.
#[derive(Accounts)]
pub struct AddToAllowlist<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,          // <- any signer, unchecked
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(
        seeds = [POLICY_SEED, mint.key().as_ref()],
        bump  = policy_config.bump,
        // no `has_one = authority`         <- the bug
    )]
    pub policy_config: Account<'info, PolicyConfig>,
    // ...
}
```

`PolicyConfig` stores the rightful `authority` (set at `initialize_policy`), but nothing compared
`ctx.accounts.authority` against it.

## Proof of concept

An integration test signs `add_to_allowlist` with a second, unrelated keypair. Before the fix it
succeeded — an outsider wrote to the allowlist. After the fix it is rejected with `Unauthorized`:

```typescript
// tests/sentinel.ts
it("rejects allowlist writes from a non-authority", async () => {
  const outsider = Keypair.generate();
  await expect(
    program.methods.addToAllowlist()
      .accounts({ authority: outsider.publicKey, mint, /* ... */ })
      .signers([outsider]).rpc()
  ).to.be.rejectedWith(/Unauthorized/);
});
```

## Remediation

Bind the signer to the stored authority on every management instruction with Anchor's `has_one`, and
anchor policy creation to the mint's own `mint_authority` so only the token issuer can create a
policy:

```rust
// Management instructions: caller must be the policy authority.
#[account(
    seeds = [POLICY_SEED, mint.key().as_ref()],
    bump  = policy_config.bump,
    has_one = authority @ SentinelError::Unauthorized,   // <- fix
)]
pub policy_config: Account<'info, PolicyConfig>,

// initialize_policy / initialize_extra_account_meta_list: only the mint authority.
#[account(
    constraint = mint.mint_authority == COption::Some(authority.key())
        @ SentinelError::Unauthorized,                   // <- fix
)]
pub mint: InterfaceAccount<'info, Mint>,
```

## Reference

Fixed before the `v1.0.0` release; covered by two negative tests (non-authority allow/block writes,
and policy initialization by a non-mint-authority). See
[softseco/sentinel](https://github.com/softseco/sentinel).
