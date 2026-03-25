# z_viewtransaction

> View detailed shielded information about an in-wallet transaction.

**Category:** Wallet
**Daemon help:** `verus help z_viewtransaction`

---

## Summary

Get detailed information about a shielded transaction in the wallet, including addresses, amounts, memos, and output indices for all Sapling spends and outputs. Useful for inspecting data-carrying transactions to understand their structure before decryption.

---

## Syntax

```
z_viewtransaction "txid"
```

---

## Parameters

| # | Name | Type | Required | Description |
|---|------|------|----------|-------------|
| 1 | `txid` | string | Yes | The transaction ID to inspect. Must be in the wallet. |

---

## Return value

```json
{
  "txid": "f1113235...",
  "spends": [],
  "outputs": [
    {
      "type": "sapling",
      "output": 0,
      "recovered": false,
      "address": "zs18wsahlhes...",
      "value": 0,
      "valueZat": 0,
      "memoStr": "\b...",
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
      ]
    }
  ]
}
```

### Top level

| Field | Type | Description |
|-------|------|-------------|
| `txid` | string | Transaction ID |
| `spends` | array | Shielded spends (inputs) in this transaction |
| `outputs` | array | Shielded outputs in this transaction |

### Spend objects

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"sapling"` or `"sprout"` |
| `spend` | number | Index of the spend within `vShieldedSpend` (sapling) |
| `txidPrev` | string | Transaction ID where the spent note was created |
| `outputPrev` | number | Output index of the spent note in the previous transaction |
| `address` | string | The z-address that spent |
| `value` | number | Amount spent |
| `valueZat` | number | Amount in zatoshis |

### Output objects

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"sapling"` or `"sprout"` |
| `output` | number | Index within `vShieldedOutput` (sapling) |
| `recovered` | boolean | True if the output is not for a wallet address (recovered via viewing key) |
| `address` | string | Destination z-address |
| `value` | number | Amount. `0` for data-only outputs. |
| `valueZat` | number | Amount in zatoshis |
| `memoStr` | string | Raw memo as UTF-8 (may contain binary garbage for data transactions) |
| `memo` | array | Parsed memo: for data transactions, contains the JSON data descriptor and trailing zero-padding. Same structure as [`z_listreceivedbyaddress`](z_listreceivedbyaddress.md) memo. |

---

## Important behaviors

- **The transaction must be in the wallet.** Only works for transactions involving addresses the wallet can decrypt.
- **`spends` is empty for transparent-funded data transactions.** When `sendcurrency` is called from a transparent address or VerusID, there are no shielded spends — only a shielded output.
- **`memoStr` is unreliable for data transactions.** Data descriptors are binary-encoded and produce garbled UTF-8. Use the parsed `memo` array instead.
- **Same memo structure as `z_listreceivedbyaddress`.** The data descriptor in the `memo` array is identical — extract it the same way.

---

## Example

```
z_viewtransaction "f1113235b529c73645c4cab66965204abc20b14eed98e0541984fed31f49a562"
```

> Output confirmed on vrsctest, 2026-03-24. Transaction contained a message stored via `sendcurrency` from a transparent VerusID. Shows: no spends (transparent funding), one sapling output at the destination z-address with `value: 0`, parsed data descriptor in memo.

---

## See also

- [`z_listreceivedbyaddress`](z_listreceivedbyaddress.md) — list all received transactions at a z-address
- [`decryptdata`](decryptdata.md) — decrypt the data descriptor
- [`sendcurrency`](../multichain/sendcurrency.md) — store data via the `data` parameter
