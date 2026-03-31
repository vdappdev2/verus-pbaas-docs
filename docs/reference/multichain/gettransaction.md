# gettransaction

> Get detailed information about a wallet transaction by transaction ID.

**Category:** Multichain
**Daemon help:** `verus help gettransaction`

---

## Summary

`gettransaction` returns the full details of an in-wallet transaction: amounts, fees, confirmations, block placement, and the raw hex. For Verus multi-currency transactions, the response includes `reservetransfer` objects (on sends) and `smartoutput` objects (on receives) that describe conversions, identity operations, and reserve token transfers.

The transaction must exist in the node's wallet. For arbitrary transactions not in the wallet, use `getrawtransaction` with verbose mode instead.

---

## Syntax

```
gettransaction "txid" (includewatchonly)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `txid` | string | Yes | — | The transaction ID to look up. |
| 2 | `includewatchonly` | boolean | No | `false` | Include watchonly addresses in balance calculation and details. |

---

## Return value

| Field | Type | Description |
|-------|------|-------------|
| `amount` | number | Net value change in native currency. Negative for sends, positive for receives. `0` for token-only or identity transactions. |
| `fee` | number | Transaction fee in native currency (negative). Only present for sends. |
| `confirmations` | number | Number of block confirmations. `0` if unconfirmed. |
| `blockhash` | string | Hash of the block containing this transaction. |
| `blockindex` | number | Position of this transaction within the block. |
| `blocktime` | number | Block timestamp (Unix epoch seconds). |
| `expiryheight` | number | Block height after which the transaction expires if unconfirmed. |
| `txid` | string | The transaction ID. |
| `walletconflicts` | array | Transaction IDs that conflict with this one (double-spend candidates). |
| `time` | number | Transaction creation time (Unix epoch seconds). |
| `timereceived` | number | Time the node received the transaction (Unix epoch seconds). |
| `vjoinsplit` | array | Sprout joinsplit data (legacy, typically empty on Verus). |
| `details` | array | Per-output breakdown. See [Details array](#details-array). |
| `hex` | string | Raw serialized transaction. |

### Details array

Each entry in `details` describes one output relevant to the wallet:

| Field | Type | Description |
|-------|------|-------------|
| `account` | string | Deprecated. Always `""`. |
| `address` | string | Destination address. |
| `category` | string | `"send"` or `"receive"`. |
| `amount` | number | Amount in native currency. Negative for sends. |
| `vout` | number | Output index in the transaction. |
| `fee` | number | Fee in native currency (sends only, negative). |
| `size` | number | Transaction size in bytes. |
| `reservetransfer` | object | Present on conversion/transfer sends. See [Reserve transfer object](#reserve-transfer-object). |
| `smartoutput` | object | Present on receives involving smart outputs. See [Smart output object](#smart-output-object). |
| `fromimport` | object | Present on receives that result from an import (e.g., conversion output). See [From import object](#from-import-object). |

### Reserve transfer object

Present on send-side entries for currency conversions, cross-chain transfers, and preconversions. Describes what was sent into the protocol for processing.

| Field | Type | Description |
|-------|------|-------------|
| `version` | number | Transfer format version. |
| `currencyvalues` | object | Map of currency i-address to amount being transferred. |
| `flags` | number | Bitfield describing the transfer type. |
| `convert` | boolean | `true` if this is a conversion operation. |
| `preconvert` | boolean | `true` if this is a prelaunch conversion. |
| `burnchangeprice` | boolean | `true` if this is a burn/mint that changes the price. |
| `feecurrencyid` | string | i-address of the currency used to pay the transfer fee. |
| `fees` | number | Transfer fee amount (separate from the mining fee). |
| `destinationcurrencyid` | string | i-address of the target currency or basket. |
| `destination` | object | Contains `address` (destination i-address or R-address), `type`, and optionally `auxdests`. |

### Smart output object

Present on receive-side entries for multi-currency receives, identity operations, and other protocol outputs.

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Output type. `"cryptocondition"` for protocol outputs. |
| `spendableoutput` | boolean | Whether this output can be spent by the wallet. `false` for identity definition outputs. |
| `reqSigs` | number | Required signatures to spend. |
| `addresses` | array | Addresses involved in spending conditions. |
| `reserveoutput` | object | Present for multi-currency receives. Contains `version` and `currencyvalues` mapping currency i-addresses to amounts. |
| `reserve_balance` | object | Friendly-name version of `reserveoutput.currencyvalues` (e.g., `{"Kaiju": 3.33379684}`). |
| `identityprimary` | object | Present for identity registration/update transactions. Contains the full identity definition (name, addresses, authorities, timelock, content). |
| `evalnone` | string | `"valid"` for simple spendable protocol outputs. |

### From import object

Present on receive-side entries that result from a processed import — typically the output side of a conversion.

| Field | Type | Description |
|-------|------|-------------|
| `importtxout` | object | Contains `txid` and `voutnum` of the import transaction. |
| `sourcetransfer` | object | The original `reservetransfer` that initiated this conversion. Same structure as the [Reserve transfer object](#reserve-transfer-object). |

This links the receive (conversion output) back to the original send (conversion input), even though they are in different transactions.

---

## Important behaviors

- **Wallet-only.** Returns an error if the txid is not in the node's wallet. Use `getrawtransaction` for non-wallet transactions.
- **`amount` reflects native currency only.** Token sends show `amount: 0` at the top level even though tokens were transferred. The actual token amounts appear in `details[].reservetransfer.currencyvalues` (send side) or `details[].smartoutput.reserveoutput.currencyvalues` (receive side).
- **Conversions produce two transactions.** The send creates a `reservetransfer` in one block. The conversion executes in the next block, producing an import transaction with the output. Both appear in `listtransactions` — the send with `reservetransfer` and the receive with `smartoutput.reserveoutput` and `fromimport`.
- **`details` may be empty.** Token-only sends between wallet addresses can show `details: []` at the top level while the transfer data is encoded in the raw hex. The `amount` and `fee` fields at the top level still reflect the transaction.
- **Identity transactions show `amount: 0`.** Identity registrations and updates appear as receives with `smartoutput.identityprimary` containing the full identity definition. The `spendableoutput: false` flag indicates this is a protocol output, not a balance.
- **`walletconflicts` tracks double-spend candidates.** Non-empty when the node has seen conflicting transactions spending the same inputs. Common during transaction replacement or reorgs.
- **`fee` is always negative.** Both the top-level `fee` and per-detail `fee` are expressed as negative numbers (e.g., `-0.0001`).

---

## Examples

### Simple token send

```
gettransaction "dfbd50e309ffca603174b9028dda698bd9ac15cfc45657c3183afdcbc30b1316"
```

```json
{
  "amount": 0,
  "fee": -0.0001,
  "confirmations": 12,
  "blockhash": "000000000f6914ab5b8280f81e21373ba62c5271f9d3610860cc190a00901a04",
  "blockindex": 2,
  "blocktime": 1774979877,
  "expiryheight": 1000189,
  "txid": "dfbd50e309ffca603174b9028dda698bd9ac15cfc45657c3183afdcbc30b1316",
  "walletconflicts": [],
  "time": 1774979757,
  "timereceived": 1774979757,
  "details": [],
  "hex": "0400008085..."
}
```

- `amount: 0` — no native currency moved (this was a token send of 1 mcp3)
- `fee: -0.0001` — mining fee in VRSCTEST
- `details: []` — token transfer data is in the raw transaction, not surfaced in the details array for wallet-internal sends

### Conversion send (VRSCTEST → Kaiju)

```
gettransaction "c6cef6a28988caffc2e5cee83a0088f7892413df4a5189d523a8a299a22e3217"
```

```json
{
  "amount": -0.1002001,
  "fee": -0.0001,
  "confirmations": 12,
  "details": [
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
        "destinationcurrencyid": "iHBwQo7LUmb7QKKqbsd8Kw9BxdQvgTdK9f",
        "destination": {
          "address": "iQ18U7oU9c9NU2Weh87ER3a4D7ZRQq1PwE",
          "type": 68
        }
      },
      "vout": 0,
      "fee": -0.0001
    }
  ]
}
```

- `amount: -0.1002001` — 0.1 VRSCTEST for conversion + 0.0002001 transfer fee
- `reservetransfer.convert: true` — this is a conversion, not a simple send
- `reservetransfer.currencyvalues` — 0.1 VRSCTEST being converted
- `reservetransfer.destinationcurrencyid` — Kaiju's i-address (`iHBwQo7LUmb7QKKqbsd8Kw9BxdQvgTdK9f`)

### Conversion receive (Kaiju output)

The conversion output arrives in a separate import transaction one block later:

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
      "flags": 3,
      "convert": true,
      "destinationcurrencyid": "iHBwQo7LUmb7QKKqbsd8Kw9BxdQvgTdK9f"
    }
  },
  "amount": 0
}
```

- `smartoutput.reserve_balance` — 3.33 Kaiju received from converting 0.1 VRSCTEST
- `fromimport.sourcetransfer` — links back to the original conversion request
- `amount: 0` — no native currency in this output (the value is in Kaiju tokens)

### Identity registration

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
  "amount": 0
}
```

- `smartoutput.identityprimary` — full identity definition embedded in the transaction
- `spendableoutput: false` — identity outputs are protocol-controlled, not directly spendable
- `amount: 0` — identity registrations carry no transferable value in the output

---

## See also

- [`z_getoperationstatus`](z_getoperationstatus.md) — get the txid from an async operation
- [`listtransactions`](listtransactions.md) — browse recent wallet transactions
- [`sendcurrency`](sendcurrency.md) — create sends, conversions, and transfers
- [`getcurrency`](getcurrency.md) — look up currency i-addresses referenced in `reservetransfer` and `smartoutput`
