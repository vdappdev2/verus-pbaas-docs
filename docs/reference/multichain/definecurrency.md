# definecurrency

> Define and launch a new currency on the Verus network.

**Category:** Multichain
**Daemon help:** `verus help definecurrency`

---

## Summary

Create simple tokens, fractional basket currencies, Ethereum-mapped tokens, or ID control tokens. The currency definition is returned as a hex transaction which must be broadcast via `sendrawtransaction`.

A VerusID is required as the namespace — the currency name matches the VerusID name, and only the controller of that ID can define the currency. Each ID can define at most one currency, and only once (even if the launch fails).

After broadcast, there is a minimum 20-block preconversion window before the currency launches.

---

## Syntax

```
definecurrency '{"name":"currencyname","options":n,...}'
```

Returns `{txid, tx, hex}`. Broadcast with:

```
sendrawtransaction "hex"
```

---

## Currency types

| Options value | Type | Description |
|---------------|------|-------------|
| 32 | Simple token | No reserves, no conversion. Can have preconversion as a funding mechanism. |
| 33 | Basket currency | Fractional reserves (1–10 currencies). Supports on-chain conversion. Requires `currencies` and `initialsupply`. |
| 2048 | ID control token | Single-satoshi token granting control of the VerusID. Sub-IDs can also define these. |

Combine with additional option bits for sub-ID restrictions (2), referral discounts (8), required referrals (16). See [Options bitfield](#options-bitfield).

---

## Parameters

### Core

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Must match an existing VerusID the wallet controls. This ID becomes the currency's namespace and rootID. |
| `options` | number | Yes | Bitfield defining currency type and features. See [Options bitfield](#options-bitfield). |
| `proofprotocol` | number | No | `1` = decentralized (default), `2` = centralized, `3` = Ethereum-mapped. See [Proof protocol](#proof-protocol). |

### Options bitfield

Options are combined by adding their values.

| Value | Name | Description |
|-------|------|-------------|
| 1 | FRACTIONAL | Has reserves. Combined with 32 for basket currencies. |
| 2 | IDRESTRICTED | Only rootID can register sub-IDs under this namespace. |
| 4 | IDSTAKING | Only sub-IDs of this namespace can stake (PBaaS chains). |
| 8 | IDREFERRALS | Referral discounts enabled for sub-ID registration. |
| 16 | IDREFERRALSREQUIRED | A referral is required to register a sub-ID. |
| 32 | TOKEN | Simple token. Also the base for baskets (combined with 1). |
| 2048 | ID control token | Single-satoshi token granting control of the VerusID. Sub-IDs can also define these. |

**Common combinations:**

| Value | Meaning |
|-------|---------|
| 32 | Simple token |
| 33 | Basket currency (TOKEN + FRACTIONAL) |
| 40 | Simple token with referral discounts (TOKEN + IDREFERRALS) |
| 41 | Basket with referral discounts |

### Proof protocol

| Value | Meaning | Sub-ID fee destination | Minting |
|-------|---------|------------------------|---------|
| 1 | Decentralized (default) | Fees are burned (deflationary) | No minting by rootID |
| 2 | Centralized | Fees go to rootID | RootID can mint and burn |
| 3 | Ethereum-mapped | — | 1:1 mapped to an Ethereum token contract |

For centralized currencies, minting continues indefinitely unless `endblock` is set, at which point centralized control ends permanently.

### Conversion and supply

| Parameter | Type | Applies to | Description |
|-----------|------|------------|-------------|
| `currencies` | string[] | Both | Reserve currencies (baskets) or preconversion source currencies (simple tokens). For baskets: must include the chain's native currency. 1–10 currencies. |
| `conversions` | number[] | Simple tokens | Preconversion price per token. Array matches `currencies` order. `[0.1]` means 1 source unit buys 10 tokens. Proceeds go to rootID. A 0.025% fee is deducted from preconversions. |
| `initialsupply` | number | Baskets | Initial supply distributed proportionally among preconverters. Required for baskets. |
| `minpreconversion` | number[] | Both | Minimum preconversion per reserve. If not met, launch fails and all preconversions are automatically refunded. Array matches `currencies` order. |
| `maxpreconversion` | number[] | Both | Maximum preconversion per reserve. Excess is automatically refunded after launch. |
| `initialcontributions` | number[] | Both | RootID contributes funds at definition time. Funds must be in the rootID's i-address. RootID receives proportional currency after launch. |
| `preallocations` | object[] | Both | Mint tokens to specified addresses at launch. Format: `[{"name@": amount}, ...]`. **Requires VerusID names** — R-addresses are rejected. |
| `prelaunchcarveout` | number | Baskets | Fraction (0–1) of preconverted reserves redirected to rootID at launch. Lowers reserve ratio. |
| `prelaunchdiscount` | number | Baskets | Discount percentage (0–1) for preconverters. After launch, conversion price is higher by this amount. Lowers reserve ratio. |
| `weights` | number[] | Baskets | Reserve weights. Must sum to 1.0. Minimum per weight: 0.05. Default: equal weights. Array matches `currencies` order. |

### Lifecycle

| Parameter | Type | Description |
|-----------|------|-------------|
| `startblock` | number | Block height at which the currency launches. Preconversion window runs from broadcast to startblock. Minimum window: 20 blocks. |
| `endblock` | number | For centralized currencies: block at which centralized control ends **permanently** — no more minting. Minimum 480 blocks after `startblock`. For decentralized currencies: no effect. |

### Sub-ID economy

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `idregistrationfees` | number | 100 | Cost to register a sub-ID. For non-basket currencies: denominated in the currency itself. For baskets: denominated in whichever reserve `idimportfees` points to. |
| `idreferrallevels` | number | 3 | Referral levels (0–5). Max 5 enforced by daemon. Requires option 8. Referral share per level = 1/(levels+1) of total fee. Remainder goes to rootID (centralized) or burn (decentralized). |
| `idimportfees` | number | 0.02 | For baskets: a satoshi-scale index encoding which reserve the registration fee is denominated in (`0.00000000` = first reserve, `0.00000001` = second, etc.). |

### Ethereum mapping

For Ethereum-mapped currencies (`proofprotocol: 3`):

| Parameter | Type | Description |
|-----------|------|-------------|
| `nativecurrencyid` | object | Ethereum token contract mapping. Format: `{"type": 9, "address": "0x..."}`. Type 9 for ERC-20. Other token standards (ERC-721, ERC-1155) are also supported but not yet tested. |
| `systemid` | string | Gateway system (e.g., `"veth"`). |
| `parent` | string | Parent system (e.g., `"vrsctest"`). |
| `launchsystemid` | string | System where the definition is broadcast (e.g., `"vrsctest"`). |

---

## Return value

```json
{
  "txid": "hexstring",
  "tx": { ... },
  "hex": "hexstring"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `txid` | string | Transaction ID of the definition |
| `tx` | object | Decoded transaction JSON |
| `hex` | string | Signed raw transaction — broadcast this with `sendrawtransaction` |

---

## Launch process

1. **Ensure the rootID exists** with sufficient funds:
   - Currency launch fee: 200 VRSC on mainnet (varies by chain)
   - Plus `initialcontributions` if specified
   - Plus transaction fees (~0.0002)

2. **Define the currency:**
   ```
   definecurrency '{"name":"mybrand","options":32,...}'
   ```

3. **Broadcast:**
   ```
   sendrawtransaction "hex"
   ```

4. **Preconversion window** (minimum 20 blocks, or until `startblock`):
   - Participants use [`sendcurrency`](sendcurrency.md) with `preconvert: true`
   - `minpreconversion` must be met or launch fails with automatic refunds
   - `maxpreconversion` caps are enforced, excess refunded

5. **Launch:** Currency becomes active. Verify with `getcurrency "name"`.

---

## Costs

| Item | Cost on Verus | Notes |
|------|---------------|-------|
| VerusID registration | 20–100 VRSC | 80 with referral |
| Currency launch | 200 VRSC | Always in chain's native coin |
| Transaction fees | ~0.0002 VRSC | Standard miner fees |

---

## Supply limits

Maximum for all currency supplies: **10 billion minus 1 satoshi** (9,999,999,999.99999999) with 8 decimal places.

---

## Important behaviors

### Reserve ratio

Preallocations, `prelaunchcarveout`, and `prelaunchdiscount` all lower the reserve ratio because they add supply or remove reserves without a corresponding offset. This makes the basket more volatile.

**Range:** 5% to 100%. The daemon enforces a minimum of 5%.

### Decentralized fee burning

For decentralized baskets (proofprotocol: 1), sub-ID registration fees are burned — less supply, same reserves — making the basket worth more over time. This is a deflationary mechanism built into the sub-ID economy.

### Centralized fee collection

For centralized currencies (proofprotocol: 2), sub-ID registration fees go to the rootID. When rootID pays itself (no referral), net balance effect is zero.

Self-referral fails — using the rootID as referral for its own sub-IDs causes "mandatory-script-verify-flag-failed". Use a different identity as referral.

### Preconversion fee

A 0.025% fee is deducted from preconversions. This must be accounted for when setting `minpreconversion` thresholds.

### endblock transition

When a centralized currency reaches its `endblock`, minting is permanently disabled ("minting is disallowed, even by currency ID after the currency endblock"). The `proofprotocol` still reads 2, but centralized operations are blocked. Sub-ID fee routing (fees to rootID) persists — only minting/burning ends.

### One currency per ID

Each VerusID can define at most one currency, and only once. Even if the launch fails (minpreconversion not met), the name cannot be reused for another currency definition.

---

## Examples

### Simple centralized token with referrals

```
definecurrency '{"name":"broom","options":40,"proofprotocol":2,"preallocations":[{"broom@":1000}],"idregistrationfees":10,"idreferrallevels":3}'
```

Options 40 = TOKEN (32) + IDREFERRALS (8). Centralized. 1000 tokens preallocated to broom@. Sub-ID registration costs 10 broom with 3 referral levels.

### Centralized token with multi-currency preconversion

```
definecurrency '{"name":"mcp3","options":40,"proofprotocol":2,"currencies":["vrsctest","kaiju"],"conversions":[0.1,0.5],"preallocations":[{"mcp3@":100},{"broom@":200},{"testidx@":200}],"idregistrationfees":50,"idreferrallevels":4,"endblock":989500}'
```

Two preconversion currencies at different rates: 1 VRSCTEST buys 10 tokens (rate 0.1), 1 kaiju buys 2 tokens (rate 0.5). `endblock` set for centralized-to-decentralized transition.

### Simple token with funding mechanism

```
definecurrency '{"name":"coolbrand","options":32,"currencies":["vrsctest"],"conversions":[0.1],"minpreconversion":[1000]}'
```

1000 VRSCTEST minimum preconversion goes to rootID as funding. Preconverters receive tokens at 10:1.

### Basket currency with multiple reserves

```
definecurrency '{"name":"communityx","options":33,"currencies":["vrsctest","mybrand","influencercoin"],"minpreconversion":[10,50,10],"initialsupply":100}'
```

Three reserves. Minimum preconversions must be met or launch fails. 100 initial supply distributed to preconverters.

### Basket with initial contributions

```
definecurrency '{"name":"communitybasket","options":33,"currencies":["vrsctest","coincommunity"],"initialcontributions":[10,200],"initialsupply":100,"preallocations":[{"jane@":100},{"john@":50}]}'
```

RootID contributes 10 VRSCTEST and 200 CoinCommunity to reserves. Preallocations lower reserve ratio.

### Basket with prelaunch discount

```
definecurrency '{"name":"discountbrand","options":33,"currencies":["vrsctest"],"initialsupply":100,"prelaunchdiscount":0.5}'
```

50% discount for preconverters. After launch, price is 50% higher and reserve ratio is 50% lower.

### Basket with custom weights

```
definecurrency '{"name":"mybusiness","options":33,"currencies":["vrsctest","businessbrand","discountbrand"],"initialsupply":100,"weights":[0.5,0.25,0.25]}'
```

Custom reserve weights: 50% VRSCTEST, 25% each for the other two.

### Ethereum-mapped token

```
definecurrency '{"name":"myusdc","options":32,"systemid":"veth","parent":"vrsctest","launchsystemid":"vrsctest","nativecurrencyid":{"type":9,"address":"0x98339D8C260052B7ad81c28c16C0b98420f2B46a"},"initialsupply":0,"proofprotocol":3}'
```

1:1 mapped to an Ethereum token contract. Interchangeable through the Verus-Ethereum bridge.

---

## See also

- [`sendcurrency`](sendcurrency.md) — send, convert, mint, burn, preconvert
- `sendrawtransaction` — broadcast the definition transaction
- `getcurrency` — inspect currency definitions and state
- `estimateconversion` — preview conversion pricing
- `listcurrencies` — discover currencies and their launch state
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — how the launch process works
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how basket pricing works
- [MEV-Resistant DeFi](../../concepts/mev-resistant-defi.md) — why conversions are front-running resistant
- [How to Launch a Centralized Token](../../how-to/currency/launch-centralized-token.md) — step-by-step guide
