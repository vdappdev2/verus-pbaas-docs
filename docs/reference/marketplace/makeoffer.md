# makeoffer

> Create a new on-chain atomic swap offer.

**Category:** Marketplace
**Daemon help:** `verus help makeoffer`

---

## Summary

Create a fully decentralized offer to trade currency for currency, currency for an identity, or identity for identity. No intermediary, no escrow. The offer transaction locks the offered asset on-chain until the offer is taken, expires, or is cancelled with [`closeoffers`](closeoffers.md).

---

## Syntax

```
makeoffer "fromaddress" '{"currency":"name","amount":n}' '{"currency":"name","amount":n,"address":"dest"}' "changeaddress" (expiryheight) (returntx) (feeamount)
```

The second positional argument is the `offer` (what you give), the third is the `for` (what you want in return).

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `fromaddress` | string | Yes | — | Funding source for fees. R-address, VerusID (`name@`), i-address, or wildcard (`"*"`, `"R*"`, `"i*"`). Does not need to control the offered asset — purely the fee source. |
| 2 | `offer` | object | Yes | — | What to offer. See [Offer object](#offer-object) below. |
| 3 | `for` | object | Yes | — | What to accept in return. See [For object](#for-object) below. |
| 4 | `changeaddress` | string | Yes | — | **Required.** Destination for change. R-address, VerusID (`name@`), or i-address. |
| 5 | `expiryheight` | number | No | current + 20 | Absolute block height at which this offer expires. Default ~20 minutes — almost certainly too short for marketplace listings. Set higher for real listings (e.g., +1440 for ~1 day, +10080 for ~1 week). No upper bound. |
| 6 | `returntx` | boolean | No | `false` | Return signed transaction hex instead of broadcasting. Nothing is locked on-chain until the hex is broadcast. Enables [off-chain offer passing](../../how-to/marketplace/list-identity-for-sale.md#off-chain-offers). |
| 7 | `feeamount` | number | No | `0.0001` | Fee in native coin. No `feecurrency` option — always native. |

### Offer object

What you are giving. Can be a currency amount or an identity.

**Currency:**

```json
{"currency": "currencynameorid", "amount": 100}
```

**ID control token** (for identities with `flags: 5` in [`getidentity`](../marketplace/../identity/getidentity.md)):

```json
{"currency": "idname.parent", "amount": 0.00000001}
```

There is exactly 1 satoshi (0.00000001) of each control token. See [ID control tokens and the marketplace](../../concepts/atomic-swaps.md#id-control-tokens).

**Identity** (only for IDs *without* control tokens):

Shorthand:
```json
{"identity": "idname@"}
```

Full definition (same structure as `updateidentity`):
```json
{
  "name": "identityname",
  "parent": "parentnameorid",
  "primaryaddresses": ["R-address"],
  "minimumsignatures": 1,
  "revocationauthority": "nameorID",
  "recoveryauthority": "nameorID"
}
```

### For object

What you want in return. Structure depends on what you are requesting.

**Currency:** The `address` field is **required** — this is where you receive payment when someone takes the offer.

```json
{"currency": "currencynameorid", "amount": 100, "address": "myaddress@"}
```

**ID control token:**

```json
{"currency": "idname.parent", "amount": 0.00000001, "address": "myaddress@"}
```

**Identity:** Must use full identity definition (shorthand not available on the `for` side). The definition determines the new state of the identity after the swap — you are pre-defining what the identity looks like when you receive it. Always include `parent` for sub-identities.

```json
{
  "name": "identityname",
  "parent": "parentnameorid",
  "primaryaddresses": ["R-address-you-control"],
  "minimumsignatures": 1,
  "revocationauthority": "yourauthority@",
  "recoveryauthority": "yourauthority@"
}
```

---

## Return value

| Field | Type | Description |
|-------|------|-------------|
| `txid` | string | Transaction ID of the offer (when `returntx: false`) |
| `oprettxid` | string | Transaction ID of the OP_RETURN output (internal) |
| `hex` | string | Serialized transaction hex (when `returntx: true`) |

---

## Important behaviors

- The offered asset must be in this wallet.
- `fromaddress` is purely the fee source — it does not need to hold or control the offered asset.
- The offer locks the asset on-chain. Funds or identity are unavailable until the offer is taken, expires, or is closed.
- A dust minimum (~0.0001 native coin) is included in the offer output for UTXO validity.
- Marketplace is **same-chain only**. Assets must be on the target chain before they can be traded.

---

## Examples

### Offer currency for currency

Offer 1 YEN for 1 USD, receiving at `myid@`:

```
makeoffer "myid@" '{"currency":"yen","amount":1}' '{"currency":"usd","amount":1,"address":"myid@"}' "myid@" 990000
```

### Offer an identity for sale

List `myname@` for 100 VRSC:

```
makeoffer "myid@" '{"identity":"myname@"}' '{"currency":"vrsc","amount":100,"address":"myid@"}' "myid@" 995000
```

### Offer an ID control token

Sell control of `premium.brand@` (which has a control token) for 500 VRSC:

```
makeoffer "myid@" '{"currency":"premium.brand","amount":0.00000001}' '{"currency":"vrsc","amount":500,"address":"myid@"}' "myid@" 995000
```

---

## See also

- [`takeoffer`](takeoffer.md) — accept an existing offer
- [`getoffers`](getoffers.md) — discover offers on the marketplace
- [`closeoffers`](closeoffers.md) — cancel an active offer
- [`listopenoffers`](listopenoffers.md) — list your own open offers
- [Atomic Swaps on Verus](../../concepts/atomic-swaps.md) — how the offer/accept model works
- [How to List an Identity for Sale](../../how-to/marketplace/list-identity-for-sale.md) — step-by-step guide
