# getreservedeposits

> Return all reserve deposits held under the control of a currency or chain.

**Category:** Multichain
**Daemon help:** `verus help getreservedeposits`

---

## Summary

`getreservedeposits` returns the reserve currency balances held in UTXOs under the control of a specified currency. These are the actual on-chain deposits backing a fractional basket or bridge currency. Optionally returns the individual UTXOs that compose those balances.

For external system currencies (bridges), all deposits are reported under the control of that system, not its independent currencies.

---

## Syntax

```
getreservedeposits "currencyname" (returnutxos)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `currencyname` | string | Yes | — | Full name or i-address of the controlling currency. |
| 2 | `returnutxos` | boolean | No | `false` | If `true`, include a `utxos` array with individual UTXO details. |

---

## Return value

An object with reserve currency i-addresses as keys and total deposit amounts as values:

| Field | Type | Description |
|-------|------|-------------|
| `utxos` | array | Present only when `returnutxos` is `true`. Each entry contains `txid`, `voutnum`, and `values` (an object mapping currency i-addresses to amounts for that UTXO). |
| `<currency i-address>` | number | Total deposit amount for this reserve currency across all UTXOs. |

Returns an empty object `{}` when the currency has no deposits — for example, a failed launch with no remaining preconversions or a currency with no reserves.

---

## Important behaviors

- **Keys are i-addresses, not currency names.** Use [`getcurrency`](getcurrency.md) to resolve i-addresses to human-readable names via the `currencynames` field.
- **Multiple UTXOs can contribute to one currency total.** The top-level per-currency values are the sum across all UTXOs containing that currency.
- **Failed launches return empty.** Currencies with `flags: 37` (launch failed) and `supply: 0` return `{}` — either no preconversions were made or refunds have already been claimed.
- **Bridge currencies report system-level deposits.** For currencies of an external system (e.g., `Bridge.vETH`), deposits are under the control of the bridge system, not the individual mapped currencies.
- **Deposits differ from reserves in currency state.** The values here represent actual UTXOs on-chain. The `reserves` field in [`getcurrency`](getcurrency.md) or [`getcurrencystate`](getcurrencystate.md) reflects the consensus-tracked reserve balance, which may differ from UTXO totals due to pending conversions, fees, or in-flight transactions.

---

## Examples

### Active basket — summary only

```
getreservedeposits "kaiju"
```

```json
{
  "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV": 8186.25756182,
  "i9nwxtKuVYX4MSbeULLiK2ttVi6rUEhh4X": 2.86161192,
  "i9oCSqKALwJtcv49xUKS2U2i79h1kX6NEY": 6012.82530913,
  "iS8TfRPfVpKo5FVfSUzfHBQxo9KuzpnqLU": 0.08866980
}
```

- Four reserve currencies with their total deposit balances
- Currency names (from `getcurrency "kaiju"`): VRSC, vETH, vUSDT.vETH, tBTC.vETH

### Active basket — with UTXO details

```
getreservedeposits "kaiju" true
```

```json
{
  "utxos": [
    {
      "txid": "561b7f64c7786be93e57e28d418d9d088fa3bada5ebcfe6c69d31e7535b7c022",
      "voutnum": 11,
      "values": {
        "i9oCSqKALwJtcv49xUKS2U2i79h1kX6NEY": 6002.98438503
      }
    },
    {
      "txid": "48d6733ffc52bd3ecd7a2c3ddf6faec898e4187c2898c97e5ce62335cd58ee84",
      "voutnum": 5,
      "values": {
        "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV": 8186.25756182
      }
    },
    {
      "txid": "48d6733ffc52bd3ecd7a2c3ddf6faec898e4187c2898c97e5ce62335cd58ee84",
      "voutnum": 6,
      "values": {
        "i9nwxtKuVYX4MSbeULLiK2ttVi6rUEhh4X": 2.86161192
      }
    },
    {
      "txid": "048fb049b9a13f96da07ff70063ca4a8a2da33c27f237f750cbd7c86764604a9",
      "voutnum": 10,
      "values": {
        "iS8TfRPfVpKo5FVfSUzfHBQxo9KuzpnqLU": 0.08866980
      }
    },
    {
      "txid": "50fdb3da66b5673690081980835f253b2b439e9a32f8296852a4b6e58c7de6af",
      "voutnum": 9,
      "values": {
        "i9oCSqKALwJtcv49xUKS2U2i79h1kX6NEY": 9.84092410
      }
    }
  ],
  "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV": 8186.25756182,
  "i9nwxtKuVYX4MSbeULLiK2ttVi6rUEhh4X": 2.86161192,
  "i9oCSqKALwJtcv49xUKS2U2i79h1kX6NEY": 6012.82530913,
  "iS8TfRPfVpKo5FVfSUzfHBQxo9KuzpnqLU": 0.08866980
}
```

- Two UTXOs contribute to the vUSDT.vETH total: 6002.98 + 9.84 = 6012.83
- Two currencies (VRSC and vETH) share the same transaction but different output indices

### Bridge currency

```
getreservedeposits "Bridge.vETH"
```

```json
{
  "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV": 63191.46071445,
  "i9nwxtKuVYX4MSbeULLiK2ttVi6rUEhh4X": 22.10272547,
  "iCkKJuJScy4Z6NSDK7Mt42ZAB2NEnAE1o4": 26.26980621,
  "iGBs4DWztRNvNEJBt4mqHszLxfKTNHTkhM": 46352.13343338
}
```

- All deposits are under the bridge system's control
- Significantly larger balances than individual baskets — the bridge holds reserves for all cross-chain value

### Failed launch — empty result

```
getreservedeposits "Bots"
```

```json
{
}
```

- Bots has `flags: 37` (launch failed) and `supply: 0` — no deposits remain

---

## See also

- [`getcurrency`](getcurrency.md) — full definition and current state, including `currencynames` to resolve i-addresses
- [`getcurrencystate`](getcurrencystate.md) — consensus-tracked reserve balances at any height
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — how currencies move through prelaunch, launch, and failure
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how reserves and prices interact
