# z_listreceivedbyaddress

> List transactions and data received at a shielded z-address.

**Category:** Wallet
**Daemon help:** `verus help z_listreceivedbyaddress`

---

## Summary

Return all transactions received at a Sapling z-address in the wallet. This is the first step in the data retrieval pipeline: list received data, extract the data descriptor from the memo, then pass it to [`decryptdata`](decryptdata.md).

Data transactions appear with `amount: 0` (or `amount > 0` if value was sent alongside data) and a structured `memo` containing the data descriptor. Value-only transactions appear with their amount and a standard hex memo.

---

## Syntax

```
z_listreceivedbyaddress "zs1..." (minconf)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `address` | string | Yes | — | Sapling z-address (`zs1...`) to list received transactions for |
| 2 | `minconf` | number | No | 1 | Only include transactions confirmed at least this many times |

---

## Return value

Returns an array of received transaction objects:

```json
[
  {
    "txid": "f1113235...",
    "amount": 0,
    "memo": [
      {
        "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
          "version": 1,
          "flags": 0,
          "objectdata": {
            "iP3euVSzNcXUrLNHnQnR9G6q8jeYuGSxgw": {
              "type": 0, "version": 1, "flags": 1,
              "output": {"txid": "0000...0000", "voutnum": 0},
              "objectnum": 0, "subobject": 0
            }
          }
        }
      },
      "0000...0000"
    ],
    "outindex": 0,
    "confirmations": 2815,
    "change": false
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `txid` | string | Transaction ID |
| `amount` | number | Amount received. `0` for data-only transactions. Can be `> 0` when data and value are sent together. |
| `memo` | string or array | For data transactions: array containing the data descriptor JSON object and trailing zero-padding. For value transactions: hex string (standard Zcash memo format). |
| `outindex` | number | The Sapling output index within the transaction |
| `confirmations` | number | Number of block confirmations |
| `change` | boolean | True if this output was change back to the sender |

### Memo structure for data transactions

The `memo` array contains two elements:

1. **Data descriptor** — a JSON object keyed by the VDXF DataDescriptor key (`i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv`). Inside is a `crosschaindataref` (`iP3euVSzNcXUrLNHnQnR9G6q8jeYuGSxgw`) that points to the transaction containing the encrypted data. An all-zero `txid` with `voutnum: 0` means the data is in the same transaction as the descriptor.

2. **Zero padding** — trailing hex zeros filling the remaining memo space.

Extract the object under `i4GC1YGEVD21...` and pass it as `datadescriptor` to [`decryptdata`](decryptdata.md).

### Memo for file data

File data stored with `-enablefileencryption` produces the same descriptor structure as message data. The descriptor does not indicate the content type — it uses the same `crosschaindataref` pattern. The decrypted output is the raw file content as hex.

Older file data may include a `mimetype` field in the descriptor (e.g., `"mimetype": "image/jpeg"`, `flags: 64`).

### Data + value coexistence

When `sendcurrency` is called with both `amount > 0` and a `data` parameter, the transaction appears with the amount AND a data descriptor in the memo. Both are present in the same output. Confirmed on vrsctest, 2026-03-24: `amount: 0.01` with message data, txid `0599600b...`.

---

## Important behaviors

- **Data and value transactions are mixed in the output.** Filter by `amount: 0` to find data-only transactions, or check whether `memo` is an array (data) vs. a hex string (value-only).
- **All historical data is returned.** There is no built-in filtering by date or block height. Use `confirmations` to determine age, or filter client-side.
- **Change outputs are included.** Set `change: true` filters are not available — filter client-side if needed.
- **The z-address must be in the wallet.** This RPC only works for addresses the wallet can decrypt.

---

## Example

```
z_listreceivedbyaddress "zs18wsahlhespjr3rj8sq5zy57psy0nf9854szl4r3w727mwhzpcxzxemsggg9ngq2jyt8tj2gmzsw"
```

---

## See also

- [`decryptdata`](decryptdata.md) — decrypt the data descriptor returned in the memo
- [`z_exportviewingkey`](z_exportviewingkey.md) — export a viewing key for sharing read access
- [`z_viewtransaction`](z_viewtransaction.md) — inspect a specific data transaction in detail
- [`sendcurrency`](../multichain/sendcurrency.md) — store data on-chain via the `data` parameter
- [How to Store and Retrieve Private Data](../../how-to/data/store-and-retrieve-private-data.md) — full round-trip guide
