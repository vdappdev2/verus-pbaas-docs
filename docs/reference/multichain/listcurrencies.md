# listcurrencies

> Discover currencies registered on the blockchain, filtered by launch state, system type, or reserve composition.

**Category:** Multichain
**Daemon help:** `verus help listcurrencies`

---

## Summary

`listcurrencies` returns an array of currency definitions with their current state. Supports filtering by launch state, system type, origin system, and reserve currency composition. Without a query object, returns all currencies on the local chain — use filters on mainnet to avoid very large result sets.

Each result includes the full currency definition, current state, and (for cross-chain currencies) notarization data with remote chain state.

---

## Syntax

```
listcurrencies ({query}) (startblock) (endblock)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `query` | object | No | `{}` | Filter object. See [Query object](#query-object). |
| 2 | `startblock` | number | No | `0` | Only return currencies defined at or after this block height. |
| 3 | `endblock` | number | No | current height | Only return currencies defined at or before this block height. |

### Query object

All fields are optional. Combine multiple fields to narrow results.

| Field | Type | Values | Description |
|-------|------|--------|-------------|
| `launchstate` | string | `"prelaunch"`, `"launched"`, `"refund"`, `"complete"` | Filter by currency lifecycle state. |
| `systemtype` | string | `"local"`, `"imported"`, `"gateway"`, `"pbaas"` | Filter by system type. |
| `fromsystem` | string | system name or i-address | Filter by origin system. Default is the local chain. |
| `converter` | array | array of currency names | Return only fractional baskets that hold **all** listed currencies as reserves. |

---

## Return value

Returns an array. Each element is an object with these top-level fields:

| Field | Type | Description |
|-------|------|-------------|
| `currencydefinition` | object | The static currency definition — same fields as [`getcurrency`](getcurrency.md) (options, currencies, weights, fees, etc.). |
| `bestheight` | number | Most recent block height with state data. |
| `besttxid` | string | Transaction ID of the best state update. |
| `besttxout` | number | Output index of the best state transaction. |
| `bestcurrencystate` | object | Currency state at `bestheight`. Same structure as `getcurrency`'s `bestcurrencystate`. |
| `lastconfirmedheight` | number | Most recent confirmed (notarized) block height. |
| `lastconfirmedtxid` | string | Transaction ID of the last confirmed state. |
| `lastconfirmedtxout` | number | Output index of the last confirmed transaction. |
| `lastconfirmedcurrencystate` | object | Currency state at `lastconfirmedheight`. |
| `lastconfirmednotarization` | object | Full notarization object (PBaaS/cross-chain currencies only). See [Notarization object](#notarization-object). |

### Notarization object

Present for PBaaS chains and cross-chain currencies. Contains cross-chain state that is not available through `getcurrency` alone.

| Field | Type | Description |
|-------|------|-------------|
| `version` | number | Notarization format version. |
| `launchcleared` | boolean | Whether the launch has been cleared. |
| `launchconfirmed` | boolean | Whether the launch has been confirmed. |
| `launchcomplete` | boolean | Whether the launch process is fully complete. |
| `ismirror` | boolean | Whether this is a mirrored notarization from the remote chain. |
| `proposer` | object | Identity and address of the notarization proposer. |
| `currencyid` | string | Currency being notarized. |
| `notarizationheight` | number | Block height on the remote chain this notarization covers. |
| `currencystate` | object | State of the currency at the notarization height. |
| `currencystates` | array | States of related currencies (e.g., the gateway converter basket) on the remote chain. |
| `proofroots` | array | Proof roots from both chains, including state roots, block hashes, and proof-of-work. |
| `nodes` | array | Network addresses for connecting to the remote chain. |

The `currencystates` array is particularly valuable — it includes the state of the gateway converter basket on the remote chain, showing cross-chain reserve balances and conversion prices.

---

## Important behaviors

- **Without filters, returns all currencies.** On mainnet this can be a very large result set. Always use filters in production.
- **`converter` requires all listed currencies.** `{"converter": ["VRSC", "vETH"]}` returns only baskets that hold both VRSC **and** vETH as reserves. A basket holding only one does not match. Unlike [`getcurrencyconverters`](getcurrencyconverters.md), there is no minimum reserve threshold — all baskets are returned regardless of liquidity.
- **`startblock`/`endblock` filter by definition time**, not launch time. A currency defined at block 3,000,000 with `startblock: 3,000,020` would match `startblock: 2,999,000` because the definition transaction was mined in that range.
- **Return structure wraps the definition.** Unlike `getcurrency` which returns definition and state as flat top-level fields, `listcurrencies` nests the definition under `currencydefinition`. Code that parses both RPCs must account for this.
- **Notarization data is included** for PBaaS chains and cross-chain currencies, providing remote chain state not available via `getcurrency`.

---

## Examples

### List all PBaaS chains

```
listcurrencies '{"systemtype":"pbaas"}'
```

Returns all registered PBaaS chains (VRSC, vDEX, vARRR, CHIPS, etc.), each with full chain parameters (eras, notaries, fees) and their latest notarization state.

### List all gateway systems

```
listcurrencies '{"systemtype":"gateway"}'
```

Returns gateway systems like vETH — Ethereum-mapped chains that bridge external networks.

### Find baskets that can convert between VRSC and vETH

```
listcurrencies '{"converter":["VRSC","vETH"]}'
```

Returns all fractional baskets holding both VRSC and vETH as reserves. On mainnet, this includes Bridge.vETH, Kaiju, NATI🦉, cybermoney, SUPER🛒, Floralis, and Bridge.vDEX — each with different weights, reserve compositions, and liquidity levels. Use this to discover conversion paths, then [`estimateconversion`](estimateconversion.md) to compare prices.

### Find currencies launched in a block range

```
listcurrencies '{"launchstate":"launched"}' 3000000 3100000
```

Returns currencies whose definition transaction was mined between blocks 3,000,000 and 3,100,000.

### List currencies in prelaunch

```
listcurrencies '{"launchstate":"prelaunch"}'
```

Returns currencies currently in the preconversion window — not yet launched but accepting preconversions.

---

## See also

- [`getcurrency`](getcurrency.md) — detailed lookup for a single currency by name
- [`getcurrencyconverters`](getcurrencyconverters.md) — find high-liquidity baskets for a specific conversion pair
- [`estimateconversion`](estimateconversion.md) — preview conversion output for a specific basket
- [`definecurrency`](definecurrency.md) — create a new currency
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — how currencies move through prelaunch, launch, and active states
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how reserve weights and prices work
