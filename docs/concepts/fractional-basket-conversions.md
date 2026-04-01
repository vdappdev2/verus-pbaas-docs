# Fractional Basket Conversions

Fractional basket currencies on Verus hold reserves of other currencies and maintain their own supply. The ratio of reserves to supply, adjusted by configured weights, determines the conversion price. This is the mechanism that powers on-chain currency exchange on Verus.

---

## How baskets work

A basket currency is created with [`definecurrency`](../reference/multichain/definecurrency.md) using `options: 33` (TOKEN + FRACTIONAL). It holds 1–10 reserve currencies and has a supply of its own token. The basket's price is derived from the relationship between its reserves and supply.

When someone converts a reserve currency into the basket, they add to the reserves and receive newly minted basket tokens. When someone converts basket tokens back to a reserve, the tokens are burned and reserves are released. The price adjusts with each block based on the net flow.

---

## Reserve weights

Each reserve currency has a **weight** that determines how much influence it has on the basket's price. Weights must sum to 1.0 at definition time.

- Default: equal weights (e.g., 3 reserves → 0.333... each)
- Minimum per weight: 0.05 (5%)
- Custom weights let you create baskets that track one reserve more heavily than others

A basket with `weights: [0.8, 0.1, 0.1]` is 80% influenced by its first reserve and 10% each by the other two.

---

## Conversion types

### Direct conversion

Convert between a reserve currency and the basket itself:

- **Reserve → Basket:** Add reserves, receive basket tokens
- **Basket → Reserve:** Burn basket tokens, receive reserves

```
sendcurrency "alice@" '[{"currency":"vrsctest","amount":10,"convertto":"kaiju","address":"alice@"}]'
```

### Via conversion

Convert between two reserve currencies that share the same basket, using the basket as an intermediary:

```
sendcurrency "alice@" '[{"currency":"vrsctest","amount":10,"convertto":"usd","via":"kaiju","address":"alice@"}]'
```

This is a two-hop conversion: VRSCTEST → kaiju basket → USD. Both hops happen atomically in the same block. The user chooses which basket to route through — if multiple baskets hold both reserves, applications must implement their own best-path selection.

Only a single basket hop per conversion is supported. You cannot chain through multiple baskets.

---

## Pricing fairness

All conversions within the same block execute at the same price. The daemon batches all conversion requests, calculates the aggregate effect on reserves and supply, and applies one price to every participant. There is no ordering advantage within a block.

This eliminates front-running, sandwich attacks, and all forms of MEV extraction. See [MEV-Resistant DeFi](mev-resistant-defi.md) for details.

---

## Fees

| Conversion type | Fee | Why |
|-----------------|-----|-----|
| Direct (reserve ↔ basket) | 0.025% | Single hop |
| Via (reserve → reserve) | 0.05% | Two hops (0.025% × 2) |

Fees are deducted from the input amount by default. To have the full amount arrive after conversion, set `addconversionfees: true` in the [`sendcurrency`](../reference/multichain/sendcurrency.md) output — fees are then charged on top.

Fees go to the basket's reserves, making the basket marginally more valuable for all holders.

---

## Reserve ratio

The reserve ratio is the relationship between reserves held and supply outstanding, adjusted by weights. It determines price stability:

- **Higher ratio (closer to 100%):** More stable, less volatile. Each conversion has a smaller price impact.
- **Lower ratio (closer to 5%):** More volatile. Small conversions can move the price significantly.

The daemon enforces a minimum reserve ratio of 5%. Several parameters at launch affect the initial reserve ratio:
- `preallocations` — mint tokens without adding reserves (lowers ratio)
- `prelaunchcarveout` — redirect a fraction of preconverted reserves to the rootID (lowers ratio)
- `prelaunchdiscount` — give preconverters a discount, lowering post-launch ratio

Currency designers should model the combined effect of these parameters before launch.

---

## Discovering conversion paths

- **`getcurrencyconverters`** — find baskets that can convert between two specific currencies (returns high-reserve baskets)
- **`listcurrencies`** — broader discovery including low-reserve baskets
- **`estimateconversion`** — preview the expected output amount and price before committing

---

## See also

- [MEV-Resistant DeFi](mev-resistant-defi.md) — why same-block pricing eliminates front-running
- [Currency Launch Lifecycle](currency-launch-lifecycle.md) — how baskets are created and launched
- [`sendcurrency`](../reference/multichain/sendcurrency.md) — executing conversions
- [`definecurrency`](../reference/multichain/definecurrency.md) — creating basket currencies
- [How to Convert Between Currencies](../how-to/currency/convert-currencies.md) — step-by-step conversion guide
- [How to Launch a Decentralized Basket Currency](../how-to/currency/launch-basket-currency.md) — basket launch walkthrough
