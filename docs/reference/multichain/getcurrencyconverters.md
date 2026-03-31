# getcurrencyconverters

> Find fractional baskets that can convert between specified currencies.

**Category:** Multichain
**Daemon help:** `verus help getcurrencyconverters`

---

## Summary

`getcurrencyconverters` returns all fractional baskets that hold the listed currencies as reserves, along with their current state and notarization data. Use it to discover conversion paths before calling [`estimateconversion`](estimateconversion.md) or [`sendcurrency`](sendcurrency.md) with `convertto`.

Supports two input modes: **simple** (pass currency names) and **advanced** (pass a query object with target price and slippage constraints).

---

## Syntax

### Simple mode

```
getcurrencyconverters "currency1" "currency2" ...
```

### Advanced mode

```
getcurrencyconverters '{"convertto":"dest","fromcurrency":"source","amount":n,"slippage":0.01}'
```

---

## Parameters

### Simple mode

Pass one or more currency names as separate string arguments. Returns all baskets holding **all** listed currencies as reserves.

### Advanced mode

Pass a single JSON object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `convertto` | string | Yes | Target currency name or i-address. |
| `fromcurrency` | string or array | Yes | Source currency name(s). Can be a string or array of objects with `currency` and `targetprice`. |
| `amount` | number | No | Amount of target currency needed. |
| `targetprice` | number or array | No | Target price(s) — only return baskets that can satisfy this price within slippage. |
| `slippage` | number | No | Maximum acceptable slippage as a decimal (e.g., `0.01` = 1%). Max: `50000000` (50%). |

---

## Return value

Returns an array. Each element is an object keyed by the basket's i-address, with these fields:

| Field | Type | Description |
|-------|------|-------------|
| (i-address key) | object | The basket's currency definition — same fields as in [`getcurrency`](getcurrency.md). |
| `fullyqualifiedname` | string | The basket's full name (e.g., `"Bridge.vETH"`, `"Kaiju"`). |
| `height` | number | Block height of the latest state. |
| `output` | object | Contains `txid` and `voutnum` of the state output. |
| `lastnotarization` | object | Notarization object with `currencystate` containing current reserves, supply, and conversion prices. |

The `lastnotarization.currencystate` is the key data — it contains the `reservecurrencies` array with current reserves and `priceinreserve` for each reserve, allowing comparison across baskets.

---

## Important behaviors

- **Filters by reserve threshold (~1000 VRSC).** Unlike [`listcurrencies`](listcurrencies.md) with the `converter` filter (which returns all baskets holding the listed reserves), `getcurrencyconverters` excludes baskets below approximately 1000 VRSC in reserves. For comprehensive discovery including low-liquidity baskets, use `listcurrencies`.
- **All listed currencies must be reserves.** Passing `"VRSC" "vETH"` only returns baskets holding both. A basket holding only one does not match.
- **Return structure differs from getcurrency and listcurrencies.** Each result is keyed by i-address (not wrapped in `currencydefinition`). Code parsing multiple RPC responses must account for the different structures.
- **Advanced mode enables price-aware filtering.** With `targetprice` and `slippage`, only baskets that can satisfy the conversion at the given price constraints are returned. This avoids returning baskets where reserve ratios make the conversion uneconomical.

---

## Examples

### Find baskets that convert between VRSC and vETH

```
getcurrencyconverters "VRSC" "vETH"
```

Returns baskets holding both VRSC and vETH as reserves. On mainnet, this includes:

| Basket | VRSC reserves | vETH reserves | VRSC price per basket token |
|--------|--------------|---------------|----------------------------|
| Bridge.vETH | ~62,889 | ~22.22 | ~15.89 |
| SUPER🛒 | ~354,998 | ~49.73 | ~47.12 |
| Kaiju | ~8,189 | ~2.87 | ~1.47 |
| Floralis | ~69,521 | ~15.14 | ~20.17 |
| NATI🦉 | ~3,738,079 | ~1,311 | ~212.41 |

Higher reserves generally mean less slippage for large conversions. Use [`estimateconversion`](estimateconversion.md) with each basket's name as `via` to compare actual output amounts.

### Comparison with listcurrencies converter filter

```
listcurrencies '{"converter":["VRSC","vETH"]}'
```

Returns 7 baskets (vs 5 from getcurrencyconverters), including cybermoney (~130 VRSC in reserves, below the ~1000 VRSC threshold) and Bridge.vDEX (runs on the vDEX system). Use `listcurrencies` for comprehensive discovery; use `getcurrencyconverters` for baskets with meaningful liquidity.

---

## See also

- [`estimateconversion`](estimateconversion.md) — preview conversion output for a specific basket
- [`listcurrencies`](listcurrencies.md) — broader discovery including low-liquidity baskets
- [`sendcurrency`](sendcurrency.md) — execute the conversion
- [`getcurrency`](getcurrency.md) — detailed lookup for a single currency
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how reserve-based pricing works
