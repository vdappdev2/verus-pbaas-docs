# listtransactions

> List recent wallet transactions with pagination.

**Category:** Multichain
**Daemon help:** `verus help listtransactions`

---

## Summary

`listtransactions` returns an array of recent wallet transactions — sends, receives, and multi-currency operations — ordered most-recent-last. Each entry includes amounts, confirmations, block info, and for multi-currency transactions, token amounts via `reservetransfer` (sends) and `smartoutput` (receives).

Use it to browse wallet activity, find transaction IDs for [`gettransaction`](gettransaction.md) lookups, or monitor conversion results.

---

## Syntax

```
listtransactions ("*" count from includewatchonly)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `account` | string | No | `"*"` | Deprecated. Use `"*"` for all accounts. |
| 2 | `count` | number | No | `10` | Number of transactions to return. |
| 3 | `from` | number | No | `0` | Number of transactions to skip (for pagination). |
| 4 | `includewatchonly` | boolean | No | `false` | Include transactions involving watchonly addresses. |

---

## Return value

Returns an array of transaction entries, ordered oldest-first within the result. Each entry represents one output relevant to the wallet — a single transaction with multiple wallet-relevant outputs produces multiple entries.

| Field | Type | Description |
|-------|------|-------------|
| `account` | string | Deprecated. Always `""`. |
| `address` | string | Address involved in the transaction. Not present for `"move"` category. |
| `category` | string | `"send"`, `"receive"`, or `"move"`. |
| `amount` | number | Amount in native currency. Negative for sends, positive for receives. `0` for token-only or identity operations. |
| `vout` | number | Output index within the transaction. |
| `fee` | number | Fee in native currency (sends only, negative). |
| `confirmations` | number | Number of block confirmations. |
| `blockhash` | string | Block hash containing the transaction. |
| `blockindex` | number | Position within the block. |
| `blocktime` | number | Block timestamp (Unix epoch seconds). |
| `expiryheight` | number | Block height after which the transaction expires if unconfirmed. |
| `txid` | string | Transaction ID. |
| `walletconflicts` | array | Transaction IDs that conflict with this one. |
| `time` | number | Transaction creation time (Unix epoch seconds). |
| `timereceived` | number | Time the node received the transaction (Unix epoch seconds). |
| `vjoinsplit` | array | Sprout joinsplit data (legacy, typically empty). |
| `size` | number | Transaction size in bytes. |
| `reservetransfer` | object | Present on conversion/transfer sends. Same structure as in [`gettransaction`](gettransaction.md#reserve-transfer-object). |
| `smartoutput` | object | Present on multi-currency receives and identity operations. Same structure as in [`gettransaction`](gettransaction.md#smart-output-object). |
| `fromimport` | object | Present on receives resulting from imports (conversion outputs). Same structure as in [`gettransaction`](gettransaction.md#from-import-object). |

---

## Important behaviors

- **One entry per wallet-relevant output.** A single transaction can produce multiple entries if it has multiple outputs touching the wallet (e.g., a send and its change output, or a conversion send and the resulting receive).
- **Conversion sends and receives are separate entries.** A conversion creates a send entry (with `reservetransfer`) in one block and a receive entry (with `smartoutput.reserveoutput`) in the next block's import transaction. They have different txids.
- **`amount: 0` for token operations.** Token sends and identity registrations show `amount: 0` because the top-level amount only reflects native currency. Token values appear in `reservetransfer.currencyvalues` (sends) or `smartoutput.reserve_balance` (receives).
- **Pagination is skip-based.** `from: 100, count: 20` returns transactions 100-119. Results are ordered oldest-first, so the most recent transactions are at the end of the full list.
- **`walletconflicts` indicates competing transactions.** Non-empty when the node has seen transactions spending the same inputs — common during transaction replacement before confirmation.
- **`fee` only appears on sends.** Receive entries do not include a fee field.
- **`timereceived` may differ from `time`.** `time` is when the transaction was created; `timereceived` is when this node first saw it. For transactions created by this wallet, they are typically identical. For external transactions, `timereceived` may be later.

---

## Examples

### List the 10 most recent transactions

```
listtransactions
```

### Paginate through history

```
listtransactions "*" 20 100
```

Returns 20 transactions starting from the 100th, skipping the first 100.

### Conversion send entry

A conversion of 0.1 VRSCTEST to Kaiju appears as:

```json
{
  "address": "RTqQe58LSj2yr5CrwYFwcsAQ1edQwmrkUU",
  "category": "send",
  "amount": -0.1002001,
  "reservetransfer": {
    "version": 1,
    "currencyvalues": {
      "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq": 0.1
    },
    "flags": 3,
    "convert": true,
    "feecurrencyid": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
    "fees": 0.0002001,
    "destinationcurrencyid": "iHBwQo7LUmb7QKKqbsd8Kw9BxdQvgTdK9f"
  },
  "fee": -0.0001,
  "confirmations": 13,
  "txid": "c6cef6a28988caffc2e5cee83a0088f7892413df4a5189d523a8a299a22e3217"
}
```

- `reservetransfer.convert: true` — this is a conversion, not a simple send
- `reservetransfer.currencyvalues` — 0.1 VRSCTEST being converted
- `reservetransfer.destinationcurrencyid` — the target basket (Kaiju)
- `amount: -0.1002001` — input amount plus transfer fee

### Conversion receive entry

The conversion output arrives one block later as a separate transaction:

```json
{
  "address": "iQ18U7oU9c9NU2Weh87ER3a4D7ZRQq1PwE",
  "category": "receive",
  "smartoutput": {
    "type": "cryptocondition",
    "reserveoutput": {
      "version": 1,
      "currencyvalues": {
        "iHBwQo7LUmb7QKKqbsd8Kw9BxdQvgTdK9f": 3.33379684
      }
    },
    "reserve_balance": {
      "Kaiju": 3.33379684
    },
    "spendableoutput": true
  },
  "fromimport": {
    "sourcetransfer": {
      "currencyvalues": {
        "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq": 0.1
      },
      "convert": true,
      "destinationcurrencyid": "iHBwQo7LUmb7QKKqbsd8Kw9BxdQvgTdK9f"
    }
  },
  "amount": 0,
  "confirmations": 11,
  "txid": "46a90ebd921c8c7393d86510b0a06752876bd412bcb47d06ed5757c771488d92"
}
```

- `smartoutput.reserve_balance` — 3.33 Kaiju received
- `fromimport.sourcetransfer` — links back to the original 0.1 VRSCTEST conversion request
- Different `txid` from the send — the conversion output is a separate import transaction

### Identity registration entry

```json
{
  "address": "iLWMiUUPoNCgJtJ54KVFaJrqujjxy5fuPv",
  "category": "receive",
  "smartoutput": {
    "type": "cryptocondition",
    "identityprimary": {
      "version": 3,
      "flags": 1,
      "primaryaddresses": ["RGGfvnyjF1is1jGwPKVSNSDNov6V4aBDMF"],
      "minimumsignatures": 1,
      "name": "uzo",
      "identityaddress": "iLWMiUUPoNCgJtJ54KVFaJrqujjxy5fuPv",
      "parent": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
      "revocationauthority": "iLWMiUUPoNCgJtJ54KVFaJrqujjxy5fuPv",
      "recoveryauthority": "iLWMiUUPoNCgJtJ54KVFaJrqujjxy5fuPv",
      "timelock": 0
    },
    "spendableoutput": false
  },
  "amount": 0,
  "confirmations": 121,
  "txid": "756a56d174524027b191b2e02c18bb207ec9b6549ff0730b2f3da6da52918ffa",
  "walletconflicts": ["90f2b9ffd302059ac334a3c96d3352f3a8b3376ef641fb0d7f63bfd108e8bbca"]
}
```

- `smartoutput.identityprimary` — full identity definition (name, addresses, authorities)
- `spendableoutput: false` — identity outputs are protocol-controlled
- `walletconflicts` — an earlier version of this identity transaction was replaced

---

## See also

- [`gettransaction`](gettransaction.md) — full details for a single transaction
- [`z_getoperationstatus`](z_getoperationstatus.md) — track async operations to get txids
- [`sendcurrency`](sendcurrency.md) — create sends, conversions, and transfers
- [`getcurrencybalance`](getcurrencybalance.md) — check balances across all currencies
