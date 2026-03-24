# closeoffers

> Cancel open offers and reclaim locked funds.

**Category:** Marketplace
**Daemon help:** `verus help closeoffers`

---

## Summary

Cancel active offers or reclaim funds from expired offers. When an offer is created with [`makeoffer`](makeoffer.md), the offered asset is locked on-chain. `closeoffers` releases those assets back to the maker.

---

## Syntax

```
closeoffers '["txid",...]' (destination) (privatefundsdestination)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `txids` | string[] | No | `[]` | Array of offer transaction IDs to close. If omitted or empty, closes **all expired offers** (cleanup mode). |
| 2 | `destination` | string | No | original source | Transparent address or identity to send reclaimed funds/tokens to. |
| 3 | `privatefundsdestination` | string | No | — | Sapling z-address for native coin routing. When specified, native coin goes here instead of `destination`. |

---

## Return value

Returns `null` on success.

---

## Important behaviors

- **No `txids`** = close all expired offers. If no expired offers exist, succeeds silently (returns `null`).
- **Specific `txids`** = cancel those active (unexpired) offers and reclaim locked assets.
- Reclaimed funds go to `destination`, or back to the original source address if no destination is specified.
- `destination` receives tokens and currency. `privatefundsdestination` optionally routes native coin to a shielded address.
- Can only close your own offers.

---

## Examples

### Cancel a specific active offer

```
closeoffers '["ca6c5f5e1abc6e3a6d319ce0255f39dedd2c94d85a1e1ea8f872135c768f1769"]'
```

### Reclaim all expired offers

```
closeoffers '[]'
```

### Reclaim to a specific address

```
closeoffers '["ca6c5f5e..."]' "myid@"
```

### Route native coin to a private address

```
closeoffers '["ca6c5f5e..."]' "myid@" "zs1abc..."
```

---

## See also

- [`listopenoffers`](listopenoffers.md) — find your open offers and their txids
- [`makeoffer`](makeoffer.md) — create offers
- [How to List an Identity for Sale](../../how-to/marketplace/list-identity-for-sale.md) — includes cancellation workflow
