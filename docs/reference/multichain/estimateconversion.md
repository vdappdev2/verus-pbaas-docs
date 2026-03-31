# estimateconversion

> Preview the expected output of a currency conversion without broadcasting a transaction.

**Category:** Multichain
**Daemon help:** `verus help estimateconversion`

---

## Summary

`estimateconversion` is a read-only estimate of converting one currency to another through a fractional basket. It accounts for pending conversions in the current block, protocol fees, and slippage. Use it before [`sendcurrency`](sendcurrency.md) with `convertto` to preview expected output.

Supports single conversions or an array of conversions through the same basket, allowing batch estimation.

---

## Syntax

```
estimateconversion '{"currency":"name","convertto":"name","amount":n}' | '[array]'
```

The argument can be a single conversion object or a JSON array of conversion objects sharing the same basket.

---

## Parameters

The parameter is a conversion object (or array of them). Each object has these fields:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `currency` | string | Yes | — | Source currency name or i-address. |
| `amount` | number | Yes | — | Amount of source currency to convert. |
| `convertto` | string | Yes | — | Destination currency name or i-address. Must be a reserve of a basket, or the basket itself. |
| `via` | string | No | — | Fractional basket to convert through. **Required** when both `currency` and `convertto` are reserves of a basket (reserve-to-reserve conversion). |
| `preconvert` | boolean | No | `false` | Estimate conversion at prelaunch rates. Only valid before the currency's `startblock`. |

---

## Return value

| Field | Type | Description |
|-------|------|-------------|
| `inputcurrencyid` | string | i-address of the source currency. |
| `netinputamount` | number | Amount actually entering the conversion after fees are deducted from the input. |
| `outputcurrencyid` | string | i-address of the destination currency. |
| `estimatedcurrencyout` | number | Estimated amount received in the destination currency. |
| `estimatedcurrencystate` | object | Full basket state after the estimated conversion would execute. Same structure as `bestcurrencystate` in [`getcurrency`](getcurrency.md). |

When an array of conversions is passed, the result contains a `conversions` array with one entry per input object, plus the `estimatedcurrencystate` reflecting all conversions combined.

---

## Important behaviors

- **Fees are deducted from the input amount.** The `netinputamount` reflects the amount after the 0.025% (direct) or 0.05% (via) fee. For example, converting 100 VRSC directly yields `netinputamount: 99.975`; converting 100 VRSC via a basket yields `netinputamount: 99.95`.
- **Via is required for reserve-to-reserve.** If both source and destination are reserves of a basket (neither is the basket itself), you must specify `via` with the basket name. Omitting it produces an error. Use [`getcurrencyconverters`](getcurrencyconverters.md) or [`listcurrencies`](listcurrencies.md) with the `converter` filter to find baskets that hold both currencies.
- **The estimate includes pending conversions.** Other conversions in the same block are factored in. Since all conversions in a block execute at the same price, the estimate reflects the aggregate effect.
- **`estimatedcurrencystate` identifies the basket.** The `currencyid` field in the estimated state shows which basket was used — useful when the basket differs from both the source and destination (as in via conversions).
- **Estimates can differ from actual execution** if additional conversions enter the same block between estimation and broadcast. The price mechanism is fair (all participants get the same price), but the exact price may shift.
- **Array form estimates combined impact.** Passing multiple conversions in an array shows what would happen if they all executed in the same block through the same basket.

---

## Examples

### Reserve to basket (VRSC → Kaiju)

```
estimateconversion '{"currency":"VRSC","amount":100,"convertto":"kaiju"}'
```

```json
{
  "inputcurrencyid": "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV",
  "netinputamount": 99.975,
  "outputcurrencyid": "i9kVWKU2VwARALpbXn4RS9zvrhvNRaUibb",
  "estimatedcurrencyout": 67.32747418,
  "estimatedcurrencystate": { ... }
}
```

- 100 VRSC in, 0.025% fee deducted → 99.975 VRSC net
- Estimated output: ~67.33 Kaiju tokens
- Effective price: ~1.49 VRSC per Kaiju

### Basket to reserve (Kaiju → VRSC)

```
estimateconversion '{"currency":"kaiju","amount":10,"convertto":"VRSC"}'
```

```json
{
  "inputcurrencyid": "i9kVWKU2VwARALpbXn4RS9zvrhvNRaUibb",
  "netinputamount": 9.9975,
  "outputcurrencyid": "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV",
  "estimatedcurrencyout": 14.75344021,
  "estimatedcurrencystate": { ... }
}
```

- 10 Kaiju in, 0.025% fee deducted → 9.9975 Kaiju net
- Estimated output: ~14.75 VRSC
- Effective price: ~1.47 VRSC per Kaiju

### Reserve to reserve via basket (VRSC → vETH via Bridge.vETH)

```
estimateconversion '{"currency":"VRSC","amount":100,"convertto":"vETH","via":"Bridge.vETH"}'
```

```json
{
  "inputcurrencyid": "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV",
  "netinputamount": 99.95,
  "outputcurrencyid": "i9nwxtKuVYX4MSbeULLiK2ttVi6rUEhh4X",
  "estimatedcurrencyout": 0.03499627,
  "estimatedcurrencystate": { ... }
}
```

- 100 VRSC in, 0.05% fee deducted (two hops) → 99.95 VRSC net
- Estimated output: ~0.035 vETH
- `estimatedcurrencystate.currencyid` is Bridge.vETH — the intermediary basket
- Effective rate: ~2,857 VRSC per vETH

---

## See also

- [`getcurrencyconverters`](getcurrencyconverters.md) — find baskets that can convert between two currencies
- [`listcurrencies`](listcurrencies.md) — broader discovery including low-liquidity baskets
- [`sendcurrency`](sendcurrency.md) — execute the conversion
- [`getcurrency`](getcurrency.md) — inspect current basket state and prices
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how pricing and fees work
- [MEV-Resistant DeFi](../../concepts/mev-resistant-defi.md) — why all conversions in a block get the same price
