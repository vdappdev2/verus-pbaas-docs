# getcurrency

> Return the full definition and current on-chain state of a currency or chain.

**Category:** Multichain
**Daemon help:** `verus help getcurrency`

---

## Summary

`getcurrency` returns everything about a currency: its static definition (reserves, weights, fees, preallocations) and its live state (supply, reserve balances, conversion prices). Use it to check whether a currency exists, understand its structure (simple token vs. fractional basket vs. PBaaS chain), read current conversion prices, or look up registration fees before defining sub-currencies or registering IDs.

With no argument, returns the definition of the current chain the daemon is running on.

---

## Syntax

```
getcurrency "currencyname"
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `currencyname` | string | No | current chain | Currency name (e.g., `"kaiju"`), fully qualified name (e.g., `"Bridge.vETH"`), i-address (e.g., `"i9kVWKU2VwARALpbXn4RS9zvrhvNRaUibb"`), or hex format (`"hex:885cfcfe62d96ae7d64142cfc4dac7ea37cdd544"`). Case-insensitive for names. |

---

## Return value

The return object has two parts: the **definition** (static, set at creation) and the **state** (dynamic, updated each block).

### Definition fields

| Field | Type | Description |
|-------|------|-------------|
| `version` | number | Definition format version. |
| `options` | number | Bitfield encoding the currency type. See [Options bitfield](#options-bitfield). |
| `name` | string | Currency name as defined. |
| `fullyqualifiedname` | string | Full name including parent namespace (e.g., `"Bridge.vETH"`). |
| `currencyid` | string | i-address that uniquely identifies this currency. |
| `currencyidhex` | string | Hex representation of the currency ID. |
| `parent` | string | i-address of the parent chain. Absent for root chains. |
| `systemid` | string | i-address of the system this currency runs on. |
| `launchsystemid` | string | i-address of the system this currency was launched from. |
| `notarizationprotocol` | number | Protocol for cross-chain notarizations. `1` = standard. |
| `proofprotocol` | number | `1` = decentralized, `2` = centralized (rootID can mint/burn), `3` = Ethereum-mapped. |
| `startblock` | number | Block height at which the currency launched. `0` for root chains. |
| `endblock` | number | Block after which minting is permanently disabled. `0` = no end. |
| `definitiontxid` | string | Transaction that created this currency. |
| `definitiontxout` | number | Output index within the definition transaction. |
| `magicnumber` | number | Unique identifier for protocol-level operations. |

### Reserve and conversion fields (baskets only)

| Field | Type | Description |
|-------|------|-------------|
| `currencies` | array | i-addresses of reserve currencies. |
| `currencynames` | object | Map of reserve i-addresses to their fully qualified names. |
| `weights` | array | Reserve weights (sum to 1.0). Determines each reserve's influence on price. |
| `conversions` | array | Pre-launch conversion rates for non-fractional currencies. `0` for baskets (price is derived from reserves). |
| `initialsupply` | number | Initial basket supply before preallocations. |
| `prelaunchcarveout` | number | Fraction of preconverted reserves redirected to the rootID at launch. |
| `initialcontributions` | array | Per-reserve amounts reserved for the launching identity. |
| `minpreconversion` | array | Minimum per-reserve preconversion for launch to proceed. |
| `maxpreconversion` | array | Maximum per-reserve preconversion accepted. |
| `preallocations` | array | VerusIDs and amounts minted at launch without adding reserves. |

### Identity and fee fields

| Field | Type | Description |
|-------|------|-------------|
| `idregistrationfees` | number | Base cost to register a sub-ID under this currency's namespace. |
| `idreferrallevels` | number | Levels of ID referral discounts. `0` = no referrals. |
| `idimportfees` | number | Fee to import an ID to this system. For baskets, a satoshi-scale value encodes which reserve currency the fee is denominated in: `0.00000000` = first reserve (index 0), `0.00000001` = second reserve (index 1), etc. |

### Chain-level fields (PBaaS chains only)

| Field | Type | Description |
|-------|------|-------------|
| `currencyregistrationfee` | number | Fee to define a new currency on this chain. |
| `pbaassystemregistrationfee` | number | Fee to register a PBaaS system on this chain. |
| `currencyimportfee` | number | Fee to import a currency definition to this chain. |
| `transactionimportfee` | number | Per-transaction import fee. |
| `transactionexportfee` | number | Per-transaction export fee. |
| `eras` | array | Mining/staking reward schedule. Each era has `reward`, `decay`, `halving`, `eraend`, and `eraoptions`. |
| `nodes` | array | Bootstrap nodes with `networkaddress` and `nodeidentity`. |
| `notaries` | array | Notary identity i-addresses (for cross-chain systems). |
| `minnotariesconfirm` | number | Minimum notary confirmations required. |

### Gateway fields

| Field | Type | Description |
|-------|------|-------------|
| `gateway` | string | i-address of the gateway system this currency bridges to. |
| `gatewayconverterid` | string | i-address of the basket that serves as the gateway converter. |
| `gatewayconvertername` | string | Name of the gateway converter basket. |
| `gatewayconverterissuance` | number | Tokens issued to the gateway converter at launch. |
| `nativecurrencyid` | object | For Ethereum-mapped currencies: contains `address` (contract address) and `type`. |

### Height and confirmation fields

| Field | Type | Description |
|-------|------|-------------|
| `bestheight` | number | Most recent block height with state data. |
| `besttxid` | string | Transaction ID of the best state update. Present for cross-chain currencies. |
| `lastconfirmedheight` | number | Most recent confirmed (notarized) block height. |
| `lastconfirmedtxid` | string | Transaction ID of the last confirmed state. Present for cross-chain currencies. |

### Currency state objects

Two state snapshots are included:

| Field | Description |
|-------|-------------|
| `bestcurrencystate` | State at `bestheight` — the latest known state, which may not yet be confirmed by notarization. |
| `lastconfirmedcurrencystate` | State at `lastconfirmedheight` — the most recent notarization-confirmed state. |

For same-chain currencies, these are typically identical. For cross-chain currencies, `bestcurrencystate` may be ahead of `lastconfirmedcurrencystate` by several blocks.

#### Currency state fields

| Field | Type | Description |
|-------|------|-------------|
| `flags` | number | State flags indicating lifecycle stage. See [State flags](#state-flags). |
| `version` | number | State format version. |
| `currencyid` | string | i-address of this currency. |
| `initialsupply` | number | Supply at launch. |
| `emitted` | number | Tokens emitted through mining/staking (chains only). |
| `supply` | number | Current total supply. For baskets, decreases as tokens are converted out (burned). For ID control tokens, always `0.00000001` (1 satoshi). |
| `primarycurrencyfees` | number | Fees collected in the primary currency this block. |
| `primarycurrencyconversionfees` | number | Conversion fees in the primary currency this block. |
| `primarycurrencyout` | number | Primary currency sent out this block. |
| `preconvertedout` | number | Amount preconverted out this block. |

#### launchcurrencies array (non-basket currencies)

Non-basket currencies (simple tokens, ID control tokens, PBaaS chains) include `launchcurrencies` instead of `reservecurrencies`. Each entry records the launch-time relationship to a reference currency:

| Field | Type | Description |
|-------|------|-------------|
| `currencyid` | string | i-address of the reference currency. |
| `weight` | number | Always `0` for non-basket currencies. |
| `reserves` | number | Reserves held (typically `0` for simple tokens). |
| `priceinreserve` | number | Launch conversion price (`0` if no preconversion was configured). |

#### reservecurrencies array (baskets only)

Each entry in `reservecurrencies` summarizes one reserve:

| Field | Type | Description |
|-------|------|-------------|
| `currencyid` | string | i-address of the reserve currency. |
| `weight` | number | This reserve's weight (matches the definition). |
| `reserves` | number | Current amount of this reserve held by the basket. |
| `priceinreserve` | number | Current price of one basket token denominated in this reserve. |

#### currencies object (per-reserve activity this block)

The `currencies` object is keyed by reserve i-address. Each entry describes conversion activity in the most recent block:

| Field | Type | Description |
|-------|------|-------------|
| `reservein` | number | Amount of this reserve converted into the basket this block. |
| `primarycurrencyin` | number | Amount of primary currency contributed via this reserve this block. |
| `reserveout` | number | Amount of this reserve converted out of the basket this block. |
| `lastconversionprice` | number | Conversion price used in the most recent block with activity. |
| `viaconversionprice` | number | Conversion price for via (reserve-to-reserve) conversions. |
| `fees` | number | Total fees collected in this reserve this block. |
| `conversionfees` | number | Conversion fees collected in this reserve this block. |
| `priorweights` | number | Weight of this reserve in the prior block. |

---

## State flags

The `flags` field in `currencystate` indicates the currency's lifecycle stage. It changes as the currency moves through launch:

| Value | Stage | Description |
|-------|-------|-------------|
| `3` | Prelaunch | Currency is defined and accepting preconversions. Before `startblock`. |
| `27` | Launch clearing | Transitional state at `startblock`. Launch is being processed (one block). |
| `49` | Active (basket) | Fractional basket is live and accepting conversions. |
| `48` | Active (non-basket) | Simple token or PBaaS chain is live. |
| `16` | Active (root chain) | Root chain state (e.g., VRSC, VRSCTEST). |

Use [`getcurrencystate`](getcurrencystate.md) with a height range spanning `startblock` to observe the transition.

---

## Options bitfield

The `options` field is a bitfield that encodes the currency type. Common values:

| Value | Meaning | Example |
|-------|---------|---------|
| `32` | Simple token | A standalone token with no reserves |
| `33` | Fractional basket (TOKEN + FRACTIONAL) | Kaiju, Bridge.vETH |
| `41` | Fractional basket with ID referrals | Kaiju (`33 + 8`) |
| `128` | PBaaS system | vETH |
| `264` | PBaaS chain with ID referrals | VRSC (`256 + 8`) |
| `545` | Gateway basket (GATEWAY + TOKEN + FRACTIONAL) | Bridge.vETH (`512 + 32 + 1`) |
| `2080` | ID control token (ID_ISSUANCE + TOKEN) | verustrading.bitcoins (`2048 + 32`) |

Individual bits:

| Bit | Value | Flag |
|-----|-------|------|
| 0 | 1 | FRACTIONAL — basket with reserves and derived pricing |
| 3 | 8 | ID_REFERRALS — sub-ID referral system enabled |
| 5 | 32 | TOKEN — currency is a token (not a chain) |
| 7 | 128 | PBAAS — PBaaS-capable system |
| 8 | 256 | IS_PBAAS_CHAIN — native PBaaS chain |
| 9 | 512 | GATEWAY — gateway currency for cross-chain bridging |
| 11 | 2048 | ID_ISSUANCE — identity control tokens enabled |

See [`definecurrency`](definecurrency.md) for the full options reference used at creation time.

---

## Important behaviors

- **Name resolution is case-insensitive.** `"kaiju"`, `"Kaiju"`, and `"KAIJU"` all return the same currency.
- **Fully qualified names** are required for sub-currencies: `"Bridge.vETH"`, not `"Bridge"` alone (which would look for a root-level currency named Bridge).
- **`bestcurrencystate` vs `lastconfirmedcurrencystate`** — for same-chain currencies these are identical. For cross-chain currencies (like vETH viewed from VRSC), `bestcurrencystate` reflects the latest known state and `lastconfirmedcurrencystate` reflects the last notarized state. Use `lastconfirmedcurrencystate` when you need finality guarantees.
- **Reserve prices are per-basket-token.** `priceinreserve: 15.99` means one basket token costs ~16 units of that reserve. To get the price of one reserve unit in basket tokens, take the reciprocal.
- **Supply tracks net issuance.** For baskets, supply decreases when tokens are burned via conversions out. The relationship between `supply`, `reserves`, and `weights` determines the conversion price.
- **Cross-system currencies may show `bestheight: 0`** when queried from a chain that is not their home system. For example, NATI.vETH (`systemid` = vETH) returns `bestheight: 0` when queried from VRSC because its state lives on the vETH chain.
- **Returns an error** if the currency name does not exist on the queried chain.

---

## Examples

### Query a fractional basket (Kaiju on mainnet)

```
getcurrency "kaiju"
```

Key fields in the response:

- `options: 41` — fractional basket with ID referrals (TOKEN + FRACTIONAL + ID_REFERRALS)
- `currencies` / `currencynames` — four reserves: VRSC, vUSDT.vETH, vETH, tBTC.vETH
- `weights: [0.25, 0.25, 0.25, 0.25]` — equal-weighted
- `bestcurrencystate.supply: 22208.60872604` — current circulating supply
- `bestcurrencystate.reservecurrencies[0].reserves: 8198.91670074` — VRSC held in reserves
- `bestcurrencystate.reservecurrencies[0].priceinreserve: 1.47670964` — one Kaiju costs ~1.48 VRSC

### Query a gateway basket (Bridge.vETH on mainnet)

```
getcurrency "Bridge.vETH"
```

- `options: 545` — gateway basket (GATEWAY + TOKEN + FRACTIONAL)
- `gateway: "i9nwxtKuVYX4MSbeULLiK2ttVi6rUEhh4X"` — bridges to the vETH system
- Four reserves: VRSC, DAI.vETH, MKR.vETH, vETH
- `bestcurrencystate.supply: 15834.05025718`
- `bestcurrencystate.reservecurrencies[0].priceinreserve: 15.99931884` — one Bridge.vETH costs ~16 VRSC

### Query a PBaaS chain (vETH on mainnet)

```
getcurrency "vETH"
```

- `options: 128` — PBaaS system
- `proofprotocol: 3` — Ethereum-mapped
- `nativecurrencyid.address: "0x71518580f36feceffe0721f06ba4703218cd7f63"` — Ethereum contract
- `notaries` — 15 notary identities
- `minnotariesconfirm: 8` — 8 of 15 required
- `gatewayconverterid` / `gatewayconvertername` — points to Bridge.vETH as the cross-chain converter
- `bestheight` and `lastconfirmedheight` differ — cross-chain state lags behind

### Query an ID control token (verustrading.bitcoins on mainnet)

```
getcurrency "verustrading.bitcoins"
```

- `options: 2080` — ID control token (ID_ISSUANCE + TOKEN)
- `parent: "i7ekXxHYzXW7uAfu5BtWZhd1MjXcWU5Rn3"` — child of the bitcoins namespace
- `bestcurrencystate.supply: 0.00000001` — exactly 1 satoshi exists (the control token)
- `preallocations` — the single satoshi was preallocated to a VerusID at launch
- `launchcurrencies` instead of `reservecurrencies` — non-basket state format
- Whoever holds this satoshi controls the associated identity on the marketplace. See [Atomic Swaps](../../concepts/atomic-swaps.md#id-control-tokens)

### Query an Ethereum-mapped simple token (NATI.vETH on mainnet)

```
getcurrency "nati.veth"
```

- `options: 32` — simple token
- `proofprotocol: 3` — Ethereum-mapped
- `nativecurrencyid.address: "0x4f14e88b5037f0ca24348fa707e4a7ee5318d9d5"` — ERC-20 contract on Ethereum
- `systemid` points to vETH (not VRSC) — this token lives on the vETH system
- `launchsystemid` points to VRSC — it was defined on the VRSC chain
- `bestheight: 0` — no state data on VRSC because the token's state lives on vETH

### Query the root chain

```
getcurrency "VRSC"
```

- `options: 264` — PBaaS chain with ID referrals
- `idregistrationfees: 100` — 100 VRSC to register an ID
- `currencyregistrationfee: 200` — 200 VRSC to launch a currency
- `eras` — three-era mining schedule with halving
- No `reservecurrencies` — VRSC is not a basket

---

## See also

- [`listcurrencies`](listcurrencies.md) — discover currencies by type, launch state, or reserve composition
- [`getcurrencystate`](getcurrencystate.md) — query currency state at a specific block height
- [`estimateconversion`](estimateconversion.md) — preview conversion output based on current state
- [`definecurrency`](definecurrency.md) — create a new currency (full parameter reference)
- [`sendcurrency`](sendcurrency.md) — send, convert, or export currencies
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how reserve weights and prices work
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — how currencies are created and launched
