# takeoffer

> Accept an existing on-chain offer, atomically executing the exchange.

**Category:** Marketplace
**Daemon help:** `verus help takeoffer`

---

## Summary

Accept an offer found via [`getoffers`](getoffers.md). Both sides of the trade execute in a single transaction, or neither does — a true atomic swap. The taker specifies what they deliver, what they accept, and where change goes.

---

## Syntax

```
takeoffer "fromaddress" '{"txid":"txid" | "tx":"hextx","changeaddress":"addr","deliver":{...},"accept":{...}}' (returntx) (feeamount)
```

The second argument is a single JSON object containing the offer reference, change address, deliver, and accept details.

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `fromaddress` | string | Yes | — | Funding source for fees. R-address, VerusID (`name@`), i-address, or wildcard (`"*"`, `"R*"`, `"i*"`). Does not need to hold the delivered asset — purely fee source. |
| 2 | (JSON object) | object | Yes | — | Single object containing all takeoffer parameters (see fields below). |
| 3 | `returntx` | boolean | No | `false` | Return signed transaction hex instead of broadcasting. For multi-sig or distributed authority workflows. |
| 4 | `feeamount` | number | No | `0.0001` | Fee in native coin. No `feecurrency` option — always native. |

### JSON object fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `txid` | string | Yes* | Transaction ID of the offer to accept (from [`getoffers`](getoffers.md)). Must have **at least 1 confirmation**. *Either `txid` or `tx` is required, not both. |
| `tx` | string | Yes* | Raw hex transaction (from [`makeoffer`](makeoffer.md) with `returntx: true`). For off-chain offer passing — no confirmation wait needed. *Either `txid` or `tx` is required, not both. |
| `changeaddress` | string | **Yes** | **Required** — daemon rejects without it. R-address, VerusID (`name@`), or i-address. |
| `deliver` | object/string | Yes | What the taker delivers. Must **exactly match** what the maker's `accept` specified. See [Deliver object](#deliver-object). |
| `accept` | object | Yes | What the taker receives. Must **exactly match** what the maker offered. See [Accept object](#accept-object). |

### Deliver object

What the taker gives. Amount must exactly match what the maker's `accept` specified.

**Currency:**
```json
{"currency": "currencynameorid", "amount": 100}
```

**Identity** (shorthand — only works on the deliver side):
```json
"fullidname@"
```

**ID control token:**
```json
{"currency": "idname.parent", "amount": 0.00000001}
```

### Accept object

What the taker receives. Must exactly match what the maker offered. The `address` field specifies where the taker receives the asset.

**Currency:**
```json
{"address": "destinationaddress", "currency": "currencynameorid", "amount": 100}
```

**Identity:** Must use full identity definition. The taker sets `primaryaddresses`, `minimumsignatures`, and authorities to define the identity's new state upon receipt.

```json
{
  "name": "identityname",
  "parent": "parentnameorid",
  "primaryaddresses": ["R-address-taker-controls"],
  "minimumsignatures": 1,
  "revocationauthority": "takerauthority@",
  "recoveryauthority": "takerauthority@"
}
```

Minimum required fields for identity accept: `name`, `primaryaddresses`, `minimumsignatures`. Include `parent` for sub-identities.

---

## Return value

| Field | Type | Description |
|-------|------|-------------|
| `txid` | string | Transaction ID of the completed swap (when `returntx: false`) |
| `hextx` | string | Serialized transaction hex (when `returntx: true`) |

---

## Important behaviors

- **1-confirmation rule** applies to the `txid` flow (on-chain offers). The `tx` flow (off-chain hex) skips this — nothing was broadcast yet.
- Both `deliver` and `accept` amounts must **exactly match** the maker's terms. No over- or under-delivery.
- Before taking, verify the offer has not expired: compare `blockexpiry` from [`getoffers`](getoffers.md) against `getblockcount`.
- When delivering an identity, it must be in this wallet.
- `deliver` is what you give. `accept` is what you get. Do not mix them up.
- A wallet can take its own offers.

### `txid` vs `tx` flow

| Flow | Source | Confirmation | Use case |
|------|--------|--------------|----------|
| `txid` | On-chain offer via [`getoffers`](getoffers.md) | 1+ confirmations required | Standard marketplace flow |
| `tx` | Raw hex from [`makeoffer`](makeoffer.md) with `returntx: true` | No confirmation needed | Off-chain offer passing, private deals |

---

## Examples

### Take a currency-for-currency offer

Accept an offer of 1 YEN for 1 USD:

```
takeoffer "myid@" '{"txid":"ca9756f7962648e0d0edde60cb0266d8f16257421bd49a5fee24672d79101ac2","changeaddress":"myid@","deliver":{"currency":"usd","amount":1},"accept":{"address":"myid@","currency":"yen","amount":1}}'
```

### Buy an identity

Accept an offer of `coolname@` for 100 VRSC:

```
takeoffer "myid@" '{"txid":"e8816674263af0ef27ef4c0c20ac3a9a7e6e68992b69a019e6524eb3b65b9379","changeaddress":"myid@","deliver":{"currency":"vrsc","amount":100},"accept":{"name":"coolname","parent":"iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq","primaryaddresses":["RMyNewAddress..."],"minimumsignatures":1,"revocationauthority":"mybackup@","recoveryauthority":"mybackup@"}}'
```

---

## See also

- [`getoffers`](getoffers.md) — find offers to take
- [`makeoffer`](makeoffer.md) — create an offer
- [`closeoffers`](closeoffers.md) — cancel offers
- [Atomic Swaps on Verus](../../concepts/atomic-swaps.md) — how the offer/accept model works
