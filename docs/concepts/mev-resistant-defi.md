# MEV-Resistant DeFi

Verus eliminates Miner Extractable Value (MEV) from on-chain currency conversion by design. All conversions within the same block execute at the same price, regardless of transaction order. There is no front-running, no sandwich attacks, and no ordering advantage.

---

## The MEV problem

On most DeFi platforms, the order of transactions within a block determines the price each trader gets. This creates a perverse incentive: miners and validators can reorder, insert, or delay transactions to extract profit at other traders' expense.

Common MEV attacks:
- **Front-running:** Seeing a large buy in the mempool, placing a buy before it (driving the price up), then selling after (profiting from the price impact the victim caused)
- **Sandwich attacks:** Placing a buy before *and* a sell after a victim's trade, capturing the spread
- **Just-in-time liquidity:** Adding and removing liquidity around a specific trade to capture fees

These attacks are possible because transaction ordering within a block affects the execution price.

---

## How Verus eliminates MEV

On Verus, all conversions submitted in the same block are **batched and executed at a single price**. The daemon collects all conversion requests, calculates the net effect on reserves and supply, and applies one price to every participant.

This means:
- **No ordering advantage.** Whether your transaction is first or last in the block, you get the same price.
- **No front-running.** Seeing a pending conversion in the mempool provides no exploitable information — you cannot get a better price by racing ahead of it.
- **No sandwich attacks.** There is no "before" and "after" within a block's conversions — they all resolve simultaneously.
- **No MEV extraction by miners.** Transaction ordering is irrelevant to conversion pricing, so miners gain nothing by reordering.

This is not a mitigation or a partial defense — it is a structural elimination of the problem. The protocol does not allow per-transaction pricing within a block.

---

## How it works

Verus uses **fractional reserve basket currencies** for on-chain conversion. A basket holds reserves of one or more currencies and maintains a supply of its own token. The conversion price is determined by the ratio of reserves to supply, weighted by each reserve's configured weight.

When multiple conversions arrive in the same block:

1. The daemon collects all conversion requests (buys, sells, via conversions)
2. It calculates the **aggregate net effect** on each reserve and the basket supply
3. It derives a single price that reflects all conversions together
4. Every participant receives tokens at that price

The converted output typically settles within 2–10 blocks depending on network activity. This slight delay is part of the batching mechanism that makes fair pricing possible.

This is fundamentally different from the sequential execution model used by Ethereum AMMs (Uniswap, Curve, etc.), where each swap changes the price for the next swap within the same block.

---

## Conversion fees

Conversion fees on Verus are protocol-level and fixed:

| Conversion type | Fee |
|-----------------|-----|
| Direct (reserve to basket or basket to reserve) | 0.025% |
| Via (reserve to reserve through a basket) | 0.05% (two hops) |

Fees are deducted from the conversion amount. There is no variable fee, no fee auction, and no fee extracted by miners — the fee goes to the basket's reserves (making the basket marginally more valuable for all holders).

---

## Practical implications

**For traders:** You get the same price as everyone else in your block. There is no advantage to sophisticated transaction timing, gas bidding, or private mempools. A retail user submitting a conversion gets the same rate as a whale.

**For developers:** Applications built on Verus conversions do not need MEV protection layers (Flashbots, private transaction pools, commit-reveal schemes). The protocol handles fairness at the consensus level.

**For liquidity providers:** Basket currencies on Verus are not subject to the toxic flow problem that plagues Ethereum LPs, where MEV bots systematically extract value from liquidity pools by trading on stale prices.

**For currency designers:** When creating a basket currency with [`definecurrency`](../reference/multichain/definecurrency.md), the MEV resistance is automatic. No additional configuration is needed — it is inherent to how Verus processes conversions.

---

## See also

- [Fractional Basket Conversions](fractional-basket-conversions.md) — how reserves, weights, and pricing work
- [`sendcurrency`](../reference/multichain/sendcurrency.md) — executing conversions
- [`definecurrency`](../reference/multichain/definecurrency.md) — creating basket currencies
- `estimateconversion` — previewing conversion output
