# revokeidentity

> Revoke an identity, preventing it from spending or signing.

**Category:** Identity
**Daemon help:** `verus help revokeidentity`

---

## Summary

Revoke a VerusID as a safety mechanism for compromised keys. A revoked identity cannot spend funds, sign transactions, or authorize any operations. Only the revocation authority (or token revocation) can invoke this. Recovery is done via [`recoveridentity`](recoveridentity.md).

---

## Syntax

```
revokeidentity "identity" (returntx) (tokenrevoke) (feeoffer) (sourceoffunds)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `identity` | string | Yes | — | VerusID name or i-address to revoke |
| 2 | `returntx` | boolean | No | `false` | Return hex instead of broadcasting |
| 3 | `tokenrevoke` | boolean | No | `false` | Use ID control token to revoke. Token must be in wallet. |
| 4 | `feeoffer` | number | No | standard fee | Non-standard fee |
| 5 | `sourceoffunds` | string | No | `"*"` | Funding source. Supports wildcards (`"*"`, `"R*"`, `"i*"`), specific addresses (including z-addresses), and VerusID names. |

---

## Return value

- **Default:** Transaction ID string
- **With `returntx: true`:** Hex string

---

## Important behaviors

- **Only changes the flag.** `revokeidentity` sets the revoked flag (`flags: 32768`) and preserves all other fields — authorities, primary addresses, content, and timelock are unchanged. Effectively equivalent to an `updateidentity` with only the flag modification.
- **Not idempotent.** Cannot revoke an already-revoked identity.
- **Self-revocation is impossible.** An identity whose `revocationauthority` is itself cannot be revoked — returns "mandatory-script-verify-flag-failed". This applies to both root IDs and sub-IDs.
- **Revocation authority only needs to be in wallet for signing.** `sourceoffunds` can be any other address — the authority signs but does not need to pay fees.
- **Revoked identity can receive funds** — sends to it succeed, but funds are invisible to `getcurrencybalance` and unspendable until recovery. After recovery, funds reappear.
- **Revoked identity cannot spend.** Error: "Could not find enough UTXOs without shielding requirements to spend."
- **Destroys any active timelock countdown.** Timelock resets to 0 upon revocation.

---

## Examples

```
revokeidentity "compromised@"
```

With explicit funding source (revocation authority signs, different address pays):

```
revokeidentity "compromised@" false false 0.0001 "payer@"
```

---

## See also

- [`recoveridentity`](recoveridentity.md) — restore a revoked identity
- [VerusID Authority Model](../../concepts/verusid-authority-model.md) — how revocation authority works
- [How to Set Up Identity Security](../../how-to/identity/set-up-security.md) — configuring authorities safely
