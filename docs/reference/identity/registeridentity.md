# registeridentity

> Step 2 of identity registration — register the identity on-chain.

**Category:** Identity
**Daemon help:** `verus help registeridentity`

---

## Summary

Uses a confirmed name commitment from [`registernamecommitment`](registernamecommitment.md) to register a VerusID on-chain. The commitment must have at least 1 confirmation. The identity definition includes primary addresses, authorities, and optionally content and a private address.

---

## Syntax

```
registeridentity "jsonidregistration" (returntx) (feeoffer) (sourceoffunds)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `jsonidregistration` | object | Yes | — | Registration object containing commitment data and identity definition. See [Registration object](#registration-object). |
| 2 | `returntx` | boolean | No | `false` | Return hex instead of broadcasting. |
| 3 | `feeoffer` | number | No | standard fee | Fee amount. Use [fee discovery](#fee-discovery) to find the correct amount. |
| 4 | `sourceoffunds` | string | No | `"*"` | Funding source. Supports wildcards (`"*"`, `"R*"`, `"i*"`, `"z*"`), specific addresses, and VerusID names. z-address works for root IDs; fails for sub-IDs under non-native namespaces. |

### Registration object

```json
{
  "txid": "commitment-txid",
  "namereservation": {
    "name": "thename",
    "salt": "salt-from-step1",
    "referral": "referral-id-or-empty",
    "parent": "parent-i-address",
    "nameid": "name-i-address"
  },
  "identity": {
    "name": "thename",
    "parent": "parent-i-address",
    "primaryaddresses": ["R-address"],
    "minimumsignatures": 1
  }
}
```

The `namereservation` is the exact object returned by [`registernamecommitment`](registernamecommitment.md).

The `identity` object fields:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Must match the committed name |
| `parent` | string | Yes | — | Parent namespace i-address. Always include. |
| `primaryaddresses` | string[] | Yes | — | R-addresses that control spending. Up to 25 for multi-sig. Only transparent R-addresses — not i-addresses or z-addresses. |
| `minimumsignatures` | number | Yes | 1 | Signatures required from `primaryaddresses`. Up to 13. |
| `revocationauthority` | string | No | self | Identity that can revoke. Omit to default to self. |
| `recoveryauthority` | string | No | self | Identity that can recover after revocation. Omit to default to self. |
| `privateaddress` | string | No | none | Sapling z-address for private transactions. |
| `contentmultimap` | object | No | empty | VDXF-structured content. Can be populated at registration time. |

> **Safety:** Never set `revocationauthority` to an external identity while leaving `recoveryauthority` as self. If the external authority revokes, the identity cannot recover itself — this bricks the identity permanently. See [VerusID Authority Model](../../concepts/verusid-authority-model.md).

> **Timelock:** Do not include `timelock` in the registration. Use [`setidentitytimelock`](setidentitytimelock.md) after registration for safe timelock management.

---

## Return value

- **Default:** Transaction ID string
- **With `returntx: true`:** Hex string of the signed transaction

---

## Fee discovery

Registration fees vary by chain and namespace.

**Method 1 — Check `getcurrency`:** Call `getcurrency` on the parent to find `idregistrationfees`. For basket currencies, `idimportfees` encodes which reserve currency the fee is denominated in (satoshi-scale index: `0.00000000` = first reserve, `0.00000001` = second, etc.).

**Method 2 — Fee shortcut:** Pass `feeoffer: 0.00000001`. The daemon rejects with the minimum required fee in the error message:

```
"Fee offer must be at least 80.00000000"
```

Retry with that amount. The fee check runs **before commitment validation** — no cost, no on-chain effect. Works even with an unconfirmed commitment.

### Observed fees (vrsctest)

| Namespace | Referral | Fee |
|-----------|----------|-----|
| Root (vrsctest) | Yes | 80 VRSCTEST |
| Root (vrsctest) | No | 100 VRSCTEST |
| kaiju (basket) | No | 84.86913942 VRSCTEST |
| mcp3 (simple token) | No | 50 mcp3 |
| mcp3 (simple token) | Yes (1 level) | 41.66666666 mcp3 |

---

## Important behaviors

- The commitment must have **at least 1 confirmation** before registration.
- **Content at registration** works — `contentmultimap` in the identity definition is stored immediately.
- **`sourceoffunds` does not need to match `controladdress`** from the commitment.
- **z-address `sourceoffunds`** works for root-namespace IDs (fee is native coin). Fails for basket sub-IDs ("Cannot source non-native currencies from a private address") and simple token sub-IDs (fails at commitment step).

---

## Example

```
registeridentity '{"txid":"abc123...","namereservation":{"name":"alice","salt":"def456...","referral":"","parent":"iJhCez...","nameid":"i1234..."},"identity":{"name":"alice","parent":"iJhCez...","primaryaddresses":["RMyAddr..."],"minimumsignatures":1}}' false 100 "fundingid@"
```

---

## See also

- [`registernamecommitment`](registernamecommitment.md) — step 1: create the name commitment
- [`getidentity`](getidentity.md) — verify registration
- [How to Register a VerusID](../../how-to/identity/register-identity.md) — end-to-end guide
- [VerusID Authority Model](../../concepts/verusid-authority-model.md) — safe authority configuration
