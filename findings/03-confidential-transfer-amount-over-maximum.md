# 03 · Transfer accepts an amount above the confidential-transfer maximum (2^48−1)

|              |                                                            |
|--------------|------------------------------------------------------------|
| **Severity** | Low                                                        |
| **Target**   | `softseco/confidential-sdk` · `src/internal/confidentialTransferProof.ts` |
| **Category** | Input validation / robustness (SDK)                        |
| **Status**   | Fixed — `v1.0.1`                                           |

## Summary

A confidential transfer encodes the amount in a 48-bit range (lo 16 bits + hi 32 bits), so the
maximum transferable amount is `2^48 − 1`. The transfer planner did not reject larger amounts; it
would proceed and build an **invalid range proof**, surfacing as a confusing on-chain proof failure
rather than a clear client-side error.

## Impact

No fund loss and no security bypass — the on-chain program rejects the malformed proof. The effect is
a poor, hard-to-diagnose failure mode for integrators passing an out-of-range amount. Rated **Low**.

## Remediation

Add an explicit bound check in the transfer plan, throwing a descriptive error before any proof is
generated:

```typescript
// 2^48 - 1  =  lo (16 bits) + hi (32 bits)
const MAX_TRANSFER_AMOUNT =
  (1n << (TRANSFER_AMOUNT_LO_BIT_LENGTH + TRANSFER_AMOUNT_HI_BIT_LENGTH)) - 1n;

if (amount > MAX_TRANSFER_AMOUNT) {
  throw new Error(
    `amount ${amount} exceeds the confidential-transfer maximum ` +
    `of ${MAX_TRANSFER_AMOUNT} (2^48-1)`,
  );
}
```

## Reference

Fixed in `v1.0.1`; covered by a unit test asserting the planner rejects an over-maximum amount. See
[softseco/confidential-sdk](https://github.com/softseco/confidential-sdk).
