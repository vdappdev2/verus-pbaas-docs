# How to List an Identity for Sale

This guide walks through listing a VerusID for sale on the on-chain marketplace, monitoring the listing, and cancelling if needed.

**Prerequisites:**
- The identity you want to sell is in your wallet
- Your wallet has a small amount of native coin for fees (~0.0001)
- You know the identity's name and whether it has a control token

---

## Step 1: Confirm ownership

Verify that your wallet controls the identity:

```
getidentity "myname@"
```

Check the output:
- `canspendfor: true` — you can transfer this identity
- `status: "active"` — the identity is not revoked
- `flags` — note the value:
  - `flags: 0` or `flags: 1` → no control token; offer the identity directly
  - `flags: 5` → has a control token; you must offer the control token as a currency instead

> **Why does this matter?** Identities with control tokens cannot be traded directly on the marketplace. The control token is a 1-satoshi currency that grants revoke/recover authority — trading it transfers that authority. See [ID control tokens and the marketplace](../../concepts/atomic-swaps.md#id-control-tokens).

---

## Step 2: Check existing offers

See if anyone is already offering to buy this identity:

```
getoffers "myname@" false
```

If offers exist, you might be able to accept one with [`takeoffer`](../../reference/marketplace/takeoffer.md) instead of creating your own listing.

---

## Step 3: Choose an expiry

The default expiry is 20 blocks (~20 minutes) — too short for a real listing. Decide how long the listing should remain active:

| Duration | Blocks |
|----------|--------|
| 1 day | ~1,440 |
| 1 week | ~10,080 |
| 1 month | ~43,200 |

Add your chosen duration to the current block height:

```
getblockcount
```

If the current height is 989,000 and you want a 1-week listing: `989000 + 10080 = 999080`.

> While the listing is active, the identity is locked on-chain and cannot be used for spending, signing, or content updates. Plan the expiry accordingly.

---

## Step 4: Create the offer

### Identity without a control token

```
makeoffer "fundingid@" '{"identity":"myname@"}' '{"currency":"vrsc","amount":100,"address":"receivingid@"}' "fundingid@" 999080
```

- `"fundingid@"` — pays the fee (~0.0001 native coin); does not need to be the identity being sold
- `{"identity":"myname@"}` — the identity you are selling
- `{"currency":"vrsc","amount":100,"address":"receivingid@"}` — the price (100 VRSC) and where you receive payment
- `"fundingid@"` — change address
- `999080` — expiry block height

### Identity with a control token (`flags: 5`)

```
makeoffer "fundingid@" '{"currency":"myname.parentchain","amount":0.00000001}' '{"currency":"vrsc","amount":100,"address":"receivingid@"}' "fundingid@" 999080
```

The only difference: the offer is `{"currency":"myname.parentchain","amount":0.00000001}` — the control token treated as a currency — instead of `{"identity":"myname@"}`.

### Response

```json
{
  "txid": "a1b2c3d4...",
  "oprettxid": "e5f6a7b8..."
}
```

Save the `txid` — you need it to cancel the offer later.

---

## Step 5: Verify the listing

Wait for 1 confirmation, then confirm your offer is visible:

```
listopenoffers
```

Your offer appears with its `expires` block height and full details.

You can also check from the buyer's perspective:

```
getoffers "myname@" false
```

---

## Step 6: Monitor and manage

### Check if the offer was taken

Run `getoffers` periodically. If your offer disappears, it was taken — the swap executed and you received payment at the address specified in step 4.

Verify payment:

```
getcurrencybalance "receivingid@"
```

### Cancel the listing

To cancel before expiry:

```
closeoffers '["a1b2c3d4..."]'
```

The identity is released back to your wallet.

### Reclaim after expiry

If the listing expired without being taken:

```
closeoffers '[]'
```

This reclaims assets from all expired offers.

To send reclaimed assets to a specific address:

```
closeoffers '["a1b2c3d4..."]' "myid@"
```

---

## Off-chain offers {#off-chain-offers}

For private sales where you already have a buyer, skip the on-chain listing:

1. Create the offer without broadcasting:
   ```
   makeoffer "fundingid@" '{"identity":"myname@"}' '{"currency":"vrsc","amount":100,"address":"receivingid@"}' "fundingid@" 999080 true
   ```
   The `true` at the end is `returntx` — returns hex instead of broadcasting.

2. Send the returned `hex` to the buyer through any channel.

3. The buyer completes the swap:
   ```
   takeoffer "buyerid@" "hexstring..." "buyerid@" '{"currency":"vrsc","amount":100}' '{"name":"myname","parent":"parentchain","primaryaddresses":["Rbuyer..."],"minimumsignatures":1}'
   ```

The identity is never locked on-chain until the buyer broadcasts the completed swap.

---

## See also

- [Atomic Swaps on Verus](../../concepts/atomic-swaps.md) — how the marketplace works
- [`makeoffer`](../../reference/marketplace/makeoffer.md) — full parameter reference
- [`takeoffer`](../../reference/marketplace/takeoffer.md) — accepting offers
- [`closeoffers`](../../reference/marketplace/closeoffers.md) — cancelling offers
- [`getoffers`](../../reference/marketplace/getoffers.md) — discovering offers
