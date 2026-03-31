# getcurrencystate

> Query the on-chain state of a currency at a specific height, across a range of heights, or at the current tip.

**Category:** Multichain
**Daemon help:** `verus help getcurrencystate`

---

## Summary

`getcurrencystate` returns the currency state at one or more block heights. Unlike [`getcurrency`](getcurrency.md) which only returns the latest state, `getcurrencystate` can query historical states — at a single height, across a range with a step interval, or at the current tip. This makes it essential for tracking how reserves, supply, and prices evolve over time.

Optionally returns OHLCV market data (open/high/low/close/volume) for conversion pairs when a `conversiondatacurrency` is specified.

---

## Syntax

```
getcurrencystate "currencynameorid" ("n") ("conversiondatacurrency")
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `currencynameorid` | string | Yes | — | Currency name or i-address. |
| 2 | `height` | string or number | No | current height | Height specification. See [Height formats](#height-formats). |
| 3 | `conversiondatacurrency` | string | No | — | If present, include OHLCV market data denominated in this currency. |

### Height formats

| Format | Description | Example |
|--------|-------------|---------|
| `n` | Single height | `"1000400"` |
| `"m,n"` | Inclusive range (every block from m to n) | `"1000400,1000410"` |
| `"m,n,o"` | Range with step interval | `"1000390,1000413,5"` |
| omitted | Current tip | — |

---

## Return value

Returns an array. Each element represents the state at one height:

| Field | Type | Description |
|-------|------|-------------|
| `height` | number | Block height of this state snapshot. |
| `blocktime` | number | Block timestamp (Unix epoch seconds). |
| `currencystate` | object | Full currency state. Same structure as `bestcurrencystate` in [`getcurrency`](getcurrency.md#currency-state-fields). |

The `currencystate` object contains `flags`, `reservecurrencies` (with per-reserve `reserves`, `weight`, `priceinreserve`), `supply`, `initialsupply`, `emitted`, and per-currency conversion activity (`reservein`, `reserveout`, `lastconversionprice`, `fees`, `conversionfees`).

### Conversion data (when conversiondatacurrency is specified)

When the third parameter is provided, each element may also include:

| Field | Type | Description |
|-------|------|-------------|
| `conversiondata` | object | Market data for the queried interval. |

The `conversiondata` object contains:

| Field | Type | Description |
|-------|------|-------------|
| `volumecurrency` | string | Currency in which volumes are denominated. |
| `volumethisinterval` | number | Total conversion volume in the interval. |
| `volumepairs` | array | Per-pair OHLCV data. See below. |

Each entry in `volumepairs`:

| Field | Type | Description |
|-------|------|-------------|
| `currency` | string | Source currency of the conversion. |
| `convertto` | string | Destination currency. |
| `volume` | number | Volume denominated in `volumecurrency`. |
| `open` | number | Opening conversion rate. |
| `high` | number | Highest conversion rate in the interval. |
| `low` | number | Lowest conversion rate in the interval. |
| `close` | number | Closing conversion rate. |

---

## Important behaviors

- **Historical queries work for any height after the currency was defined.** Heights before the definition transaction return the initial state with zero reserves.
- **Range queries return one entry per step.** `"1000390,1000413,5"` returns entries at heights 1000390, 1000395, 1000400, 1000405, 1000410, and 1000413. The step is approximate — the final height in the range is always included.
- **State at a height reflects all conversions processed up to and including that block.** The `reservein`, `reserveout`, and `fees` fields show activity in the most recent block with conversions, not cumulative totals across all blocks.
- **`currencystate` is identical to the state object in `getcurrency`.** The same `reservecurrencies`, `currencies`, `supply`, and `flags` fields appear. The difference is that `getcurrencystate` can return it at any height, not just the latest.
- **Flags track the launch lifecycle.** Prelaunch: `flags: 3`. At `startblock`: `flags: 27` (launch clearing — one block). Active: `flags: 49` for baskets, `flags: 48` for non-basket tokens. Query across `startblock` to observe the transition.
- **Conversion data requires a range query with a step.** The `conversiondata` object only appears when using the `"m,n,o"` range format with a step interval. It aggregates OHLCV data across each step interval.
- **`lastconversionprice` persists.** Even in blocks with no new conversions, `lastconversionprice` retains the value from the most recent block that had conversions.

---

## Examples

### Current state of an active basket

```
getcurrencystate "kaiju"
```

```json
[
  {
    "height": 1000414,
    "blocktime": 1774993751,
    "currencystate": {
      "flags": 49,
      "currencyid": "iHBwQo7LUmb7QKKqbsd8Kw9BxdQvgTdK9f",
      "reservecurrencies": [
        { "currencyid": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq", "weight": 0.25, "reserves": 7724.26355753, "priceinreserve": 0.02998848 },
        { "currencyid": "iFawzbS99RqGs7J2TNxME1TmmayBGuRkA2", "weight": 0.25, "reserves": 45527.90762215, "priceinreserve": 0.17675639 },
        { "currencyid": "iBBRjDbPf3wdFpghLotJQ3ESjtPBxn6NS3", "weight": 0.25, "reserves": 0.70411374, "priceinreserve": 0.00000273 },
        { "currencyid": "i8xcvrfKTw8eEiTMjt1dAQxpWcbZeaVMFV", "weight": 0.25, "reserves": 102184.5462534, "priceinreserve": 0.3967187 }
      ],
      "supply": 1030297.24298552
    }
  }
]
```

- `flags: 49` — active fractional basket
- Four reserves with current balances and per-reserve prices

### Track a prelaunch currency over time

Query the TB1 basket from definition through preconversion using a step of 5 blocks:

```
getcurrencystate "TB1" "1000390,1000413,5"
```

| Height | VRSCTEST reserves | USD reserves | Event |
|--------|------------------|-------------|-------|
| 1000390 | 0 | 0 | Just defined — no preconversions processed yet |
| 1000395 | 0 | 0 | Waiting for preconversions |
| 1000400 | 5 | 29.47 | `initialcontributions` processed (5 VRSCTEST + 29.47 USD) |
| 1000405 | 5 | 29.47 | Stable — no new preconversions |
| 1000410 | 6.9995 | 41.257 | External preconversions landed (~2 VRSCTEST + ~11.79 USD) |
| 1000413 | 6.9995 | 41.257 | Stable |

The `priceinreserve` evolves as reserves accumulate:
- At height 1000400: 0.1 VRSCTEST, 0.5894 USD per TB1 (from initialcontributions only)
- At height 1000410: 0.13999 VRSCTEST, 0.82514 USD per TB1 (after external preconversions)

### Query a single historical height

```
getcurrencystate "kaiju" "1000375"
```

Returns the Kaiju basket state at exactly block 1000375.

### OHLCV market data over a wide range

Query the Pure basket across 100,000 blocks with daily steps (~1440 blocks), denominated in VRSC:

```
getcurrencystate "pure" "3200000,3300000,1440" "VRSC"
```

Each entry in the result includes a `conversiondata` object aggregating conversion activity across that step interval:

```json
{
  "height": 3201440,
  "blocktime": 1743980960,
  "currencystate": { ... },
  "conversiondata": {
    "volumecurrency": "VRSC",
    "volumethisinterval": 6294.42657320,
    "volumepairs": [
      {
        "currency": "VRSC",
        "convertto": "tBTC.vETH",
        "volume": 4997.50,
        "open": 34303.71798440,
        "high": 34303.71798440,
        "low": 34303.71798440,
        "close": 34303.71798440
      },
      {
        "currency": "Pure",
        "convertto": "VRSC",
        "volume": 870.65228971,
        "open": 0.26976326,
        "high": 0.26976326,
        "low": 0.26976326,
        "close": 0.26976326
      },
      {
        "currency": "tBTC.vETH",
        "convertto": "VRSC",
        "volume": 426.27428349,
        "open": 0.00002902,
        "high": 0.00002904,
        "low": 0.00002902,
        "close": 0.00002904
      }
    ]
  }
}
```

- `volumethisinterval` — total conversion volume across all pairs in the step interval, denominated in VRSC
- Each `volumepairs` entry shows one conversion direction with its volume and OHLC rates
- The step size determines the aggregation window — choose based on the level of conversion activity and desired chart resolution
- The third parameter determines the denomination currency for volumes — use a reserve currency of the basket

---

## See also

- [`getcurrency`](getcurrency.md) — current state and full definition (no historical queries)
- [`estimateconversion`](estimateconversion.md) — preview conversion output at current state
- [`listcurrencies`](listcurrencies.md) — discover currencies by launch state or reserve composition
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — how currencies move through prelaunch and launch
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how reserves and prices work
