# Atomic Swaps on Verus

Verus includes a built-in on-chain marketplace for trustless, peer-to-peer trading of currencies and identities. Every trade is an atomic swap — both sides of the exchange execute in a single transaction, or neither does. There is no escrow, no intermediary, and no counterparty risk.

---

## How atomic swaps work

A trade has two participants: a **maker** and a **taker**.

1. **The maker creates an offer.** The offered asset (currency or identity) is locked on-chain in a special output that can only be spent in two ways: by a valid counterparty completing the swap, or by the maker reclaiming it (cancellation or expiry).

2. **The taker completes the swap.** The taker constructs a transaction that simultaneously delivers what the maker requested and claims what the maker offered. The daemon validates that both sides match exactly. If they do, a single transaction moves both assets to their new owners. If anything is wrong — wrong amount, wrong asset, offer expired — the transaction is rejected and nothing moves.

This is not a two-step process where one side could fail to deliver. The swap is a single atomic operation at the blockchain level. Both transfers happen in the same transaction, enforced by consensus rules.

### What can be traded

The marketplace supports four combinations:

| Offer | For | Example |
|-------|-----|---------|
| Currency | Currency | Trade 1 BTC for 5000 VRSC |
| Currency | Identity | Buy a VerusID with currency |
| Identity | Currency | Sell a VerusID for currency |
| Identity | Identity | Swap one VerusID for another |

All four combinations use the same [`makeoffer`](../reference/marketplace/makeoffer.md) / [`takeoffer`](../reference/marketplace/takeoffer.md) workflow.

---

## Offer lifecycle

An offer progresses through a simple lifecycle:

```
Created → [1 confirmation] → Available → Taken  (swap executed)
                                       → Expired (reclaim with closeoffers)
                                       → Closed  (cancelled by maker)
```

- **Created:** The `makeoffer` transaction is broadcast. The offered asset is locked.
- **Available:** After 1 confirmation, the offer appears in [`getoffers`](../reference/marketplace/getoffers.md) and can be taken.
- **Taken:** A taker calls [`takeoffer`](../reference/marketplace/takeoffer.md). The atomic swap executes.
- **Expired:** The offer reaches its `expiryheight` block without being taken. The maker calls [`closeoffers`](../reference/marketplace/closeoffers.md) to reclaim the locked asset.
- **Closed:** The maker cancels the offer before expiry using `closeoffers`.

### Expiry

Every offer has an `expiryheight` — the block height at which it becomes invalid. The default is 20 blocks (~20 minutes), which is suitable for coordinated trades but too short for marketplace listings. For real listings, set a longer expiry:

| Duration | Blocks (approximate) |
|----------|---------------------|
| 1 hour | ~60 |
| 1 day | ~1,440 |
| 1 week | ~10,080 |
| 1 month | ~43,200 |

There is no upper bound on `expiryheight`. The trade-off: a longer expiry means the asset is locked for longer if the offer is not taken and the maker does not cancel.

---

## ID control tokens and the marketplace {#id-control-tokens}

Some VerusIDs have an associated **control token** — a currency with the same name as the identity and a total supply of exactly 1 satoshi (0.00000001). The control token grants revoke and recover authority over the identity. Check for a control token by looking at `flags: 5` in [`getidentity`](../reference/identity/getidentity.md) output, or by querying `getcurrency` with the identity name.

When an identity has a control token, the marketplace enforces a rule: **the identity cannot be offered or requested directly**. Instead, trade the control token as a currency:

```json
{"currency": "idname.parent", "amount": 0.00000001}
```

This is because the control token *is* the authority over the identity. Trading the token transfers revoke/recover authority, which is the meaningful form of ownership transfer for these identities.

Identities *without* control tokens can be traded directly using the identity format in `makeoffer` / `takeoffer`.

---

## Dust minimum

Every offer output includes a small amount of native currency (~0.0001) alongside the main offered asset. This is a requirement of the UTXO model — every on-chain output must contain a minimum amount of native coin to be valid. The dust amount is handled automatically by the daemon and does not affect the trade terms.

---

## Off-chain offer passing

The standard flow broadcasts the offer on-chain, where anyone can discover and take it. For private or coordinated trades, an alternative exists:

1. The maker calls [`makeoffer`](../reference/marketplace/makeoffer.md) with `returntx: true`. This returns the signed offer as raw hex without broadcasting — nothing is locked on-chain.
2. The maker sends the hex to the counterparty through any out-of-band channel.
3. The counterparty calls [`takeoffer`](../reference/marketplace/takeoffer.md) with the `tx` parameter (instead of `txid`) to complete and broadcast the swap.

This flow skips the 1-confirmation waiting period since no on-chain offer exists until the taker broadcasts the completed swap.

---

## Same-chain only

The marketplace operates on a single chain. To trade assets across chains, export the asset to the target chain first (using [`sendcurrency`](../reference/multichain/sendcurrency.md) with `exportcurrency` or `exportid`), then trade on that chain's marketplace.

---

## See also

- **Reference:** [`makeoffer`](../reference/marketplace/makeoffer.md), [`takeoffer`](../reference/marketplace/takeoffer.md), [`getoffers`](../reference/marketplace/getoffers.md), [`listopenoffers`](../reference/marketplace/listopenoffers.md), [`closeoffers`](../reference/marketplace/closeoffers.md)
- **How-to:** [How to List an Identity for Sale](../how-to/marketplace/list-identity-for-sale.md)
