# recoveridentity

> Restore a revoked identity with new state.

**Category:** Identity
**Daemon help:** `verus help recoveridentity`

---

## Summary

Restore a revoked identity, typically with new primary addresses (since the original keys may be compromised). Only the recovery authority (or token recovery) can invoke this. The identity must be revoked first — you cannot recover an active identity.

---

## Syntax

```
recoveridentity "jsonidentity" (returntx) (tokenrecover) (feeoffer) (sourceoffunds)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `jsonidentity` | object | Yes | — | New identity definition. Same structure as [`updateidentity`](updateidentity.md). |
| 2 | `returntx` | boolean | No | `false` | Return hex instead of broadcasting |
| 3 | `tokenrecover` | boolean | No | `false` | Use ID control token to recover. Token must be in wallet. |
| 4 | `feeoffer` | number | No | standard fee | Non-standard fee |
| 5 | `sourceoffunds` | string | No | `"*"` | Funding source. Supports wildcards (`"*"`, `"R*"`, `"i*"`, `"z*"`), specific addresses, and VerusID names. z-address works. |

---

## Return value

- **Default:** Transaction ID string
- **With `returntx: true`:** Hex string

---

## Field behavior in recovery

Recovery uses the same identity JSON as [`updateidentity`](updateidentity.md), but with different carry-over rules for some fields:

| Field | Behavior |
|-------|----------|
| `primaryaddresses` | Set to new addresses (the whole point of recovery) |
| `revocationauthority` | Carried over if omitted |
| `recoveryauthority` | Carried over if omitted |
| `privateaddress` | Carried over if omitted. `null` clears. `""` preserves. |
| `contentmultimap` | Behavior TBD |
| `timelock` | **Does NOT carry over** — omitting resets to 0. Including `timelock: N` sets an absolute lock (flags=0), not a delay lock. |

---

## Important behaviors

- **Identity must be revoked** before recovery. Cannot recover an active identity.
- **Recovery authority only needs to be in wallet for signing.** `sourceoffunds` can be any other address.
- **Funds sent while revoked reappear after recovery.** Balance is restored including any value received during the revoked period.
- **Timelock resets to 0** unless explicitly set. If you set `timelock: N` in recovery, it creates an absolute block height lock — not a delay lock. Use [`setidentitytimelock`](setidentitytimelock.md) after recovery for safe timelock configuration.
- **`privateaddress`** can be changed, cleared (`null`), or preserved (omit) during recovery.

---

## Example

Recover with new primary addresses and safe external authorities:

```
recoveridentity '{"name":"compromised","parent":"iJhCez...","primaryaddresses":["RNewSafeAddr..."],"minimumsignatures":1,"revocationauthority":"guardian@","recoveryauthority":"backup@"}'
```

---

## See also

- [`revokeidentity`](revokeidentity.md) — revoke an identity (prerequisite)
- [`getidentity`](getidentity.md) — verify recovery
- [VerusID Authority Model](../../concepts/verusid-authority-model.md) — authority configuration
- [How to Set Up Identity Security](../../how-to/identity/set-up-security.md) — recovery workflow
