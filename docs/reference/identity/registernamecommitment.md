# registernamecommitment

> Step 1 of identity registration — reserve a name without revealing it.

**Category:** Identity
**Daemon help:** `verus help registernamecommitment`

---

## Summary

Creates a name commitment transaction that reserves a name on-chain without revealing it, preventing miners from front-running desirable name registrations. The commitment uses a salt-based hash so the name itself is hidden until step 2 ([`registeridentity`](registeridentity.md)).

Wait at least 1 block after the commitment confirms before calling `registeridentity`.

---

## Syntax

```
registernamecommitment "name" "controladdress" ("referralidentity") ("parentnameorid") ("sourceoffunds")
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `name` | string | Yes | — | Name to commit to. Must not already exist. No leading/trailing/consecutive spaces. Cannot include: `\ / : * ? " < > \| @`. |
| 2 | `controladdress` | string | Yes | — | Address that controls this commitment — must be in the current wallet. Not necessarily the address that will control the identity. Accepts R-addresses, VerusID friendly names (`"myid@"`), and i-addresses. Does **not** accept z-addresses. |
| 3 | `referralidentity` | string | No | — | Referral identity for discounted registration. Discount amount varies by currency definition (`idreferrallevels` in `definecurrency`). |
| 4 | `parentnameorid` | string | No | — | Parent currency name or i-address. For root IDs on VRSC/vrsctest, omit. For sub-IDs under a currency namespace, pass the currency name. |
| 5 | `sourceoffunds` | string | No | `"*"` | Funding source. Supports wildcards (`"*"`, `"R*"`, `"i*"`), specific addresses, and VerusID names. Specific z-addresses work for root-namespace commitments but fail for sub-IDs under non-native namespaces. |

---

## Return value

```json
{
  "txid": "hexstring",
  "namereservation": {
    "version": 1,
    "name": "thename",
    "parent": "i-address",
    "salt": "hexstring",
    "referral": "i-address-or-empty",
    "nameid": "i-address"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `txid` | string | Transaction ID of the commitment |
| `namereservation` | object | Commitment data — **preserve this entire object** for step 2 |
| `namereservation.salt` | string | Random salt that hides the name. Required for registration. |
| `namereservation.nameid` | string | The i-address that this name will resolve to |

---

## Important behaviors

- **Commitment stays valid** for a long time (possibly unlimited), unless someone else registers the name first.
- **If someone registers the name** between your commitment and `registeridentity`, registration simply fails. No funds are lost beyond the commitment transaction fee.
- **The `namereservation` output must be preserved** — the salt is needed for step 2 and cannot be recovered.
- **`controladdress` does not need to fund** the transaction if `sourceoffunds` is specified separately.
- **z-address as `controladdress`** is rejected ("Invalid control address for commitment").
- **z-address as `sourceoffunds`** works for root IDs (fee is native coin) but fails for sub-IDs under non-native namespaces ("Invalid parameters").

---

## Examples

### Root ID commitment

```
registernamecommitment "alice" "myid@"
```

### Sub-ID commitment with referral

```
registernamecommitment "newsub" "myid@" "existingsub.mcp3@" "mcp3"
```

- `"newsub"` — the sub-ID name to register
- `"myid@"` — controladdress (must be in wallet)
- `"existingsub.mcp3@"` — referral identity for fee discount
- `"mcp3"` — parent currency namespace

---

## See also

- [`registeridentity`](registeridentity.md) — step 2: register the identity using this commitment
- [How to Register a VerusID](../../how-to/identity/register-identity.md) — end-to-end guide
- [VerusID Lifecycle](../../concepts/verusid-lifecycle.md) — why registration is two steps
