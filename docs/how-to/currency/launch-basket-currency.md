# How to Launch a Decentralized Basket Currency

This guide walks through launching a fractional basket currency on Verus — a currency that holds reserves and supports on-chain conversions between them.

**Prerequisites:**
- A VerusID to use as the currency namespace (see [How to Register a VerusID](../identity/register-identity.md))
- Wallet funds: 200 VRSC launch fee + `initialcontributions` amounts + transaction fees (~0.0002)
- The reserve currencies must exist on the chain (native currency is always available; others must have been imported if cross-chain)

---

## Step 1: Choose reserves and weights

Decide which currencies to include as reserves and how to weight them. Weights must sum to 1.0 and each weight must be at least 0.05.

| Parameter | What it controls |
|-----------|-----------------|
| `currencies` | Reserve currencies (1–10). Must include the chain's native currency (e.g., VRSC). |
| `weights` | Influence of each reserve on the basket's price. Higher weight = more influence. |
| `initialsupply` | Total basket tokens distributed to preconverters at launch. |
| `initialcontributions` | Funds the rootID contributes to reserves at definition time. |

---

## Step 2: Define the currency

### Basic basket with two reserves

```
definecurrency '{"name":"TB1","options":41,"currencies":["vrsctest","USD"],"weights":[0.5,0.5],"initialsupply":100,"initialcontributions":[5,29.47],"idregistrationfees":5,"idreferrallevels":2}'
```

- `options: 41` — TOKEN (32) + FRACTIONAL (1) + IDREFERRALS (8)
- `currencies` — two reserves: VRSCTEST and USD, equally weighted
- `initialsupply: 100` — distributed to preconverters proportionally at launch
- `initialcontributions` — rootID seeds the basket with 5 VRSCTEST and 29.47 USD (must be in the rootID's address at broadcast)
- `idregistrationfees: 5` — sub-IDs under TB1 cost 5 of the first reserve currency
- `idreferrallevels: 2` — referral discounts with 2 levels

### Without referrals

```
definecurrency '{"name":"mybasket","options":33,"currencies":["vrsctest","USD"],"weights":[0.5,0.5],"initialsupply":1000,"initialcontributions":[10,50]}'
```

- `options: 33` — TOKEN (32) + FRACTIONAL (1), no referral system

### With minimum preconversion thresholds

```
definecurrency '{"name":"mybasket","options":33,"currencies":["vrsctest","USD","tBTC.vETH"],"weights":[0.4,0.4,0.2],"initialsupply":5000,"minpreconversion":[100,500,0.001]}'
```

- `minpreconversion` — if any threshold is not met, the launch fails and all preconversions are automatically refunded
- Three reserves with custom weights (40%, 40%, 20%)

---

## Step 3: Broadcast

`definecurrency` returns `{txid, tx, hex}`. Broadcast the hex:

```
sendrawtransaction "hex-from-definecurrency"
```

---

## Step 4: Preconvert (optional)

Between broadcast and `startblock` (minimum 20 blocks), anyone can preconvert:

```
sendcurrency "myid@" '[{"currency":"vrsctest","amount":2,"convertto":"TB1","preconvert":true,"address":"myid@"}]'
```

- Preconversions can be in any of the basket's reserve currencies
- All preconverters get the same fair price — there is no advantage to early or late participation within the window
- A 0.025% fee is deducted from preconversion amounts

Track the operation:

```
z_getoperationstatus
```

---

## Step 5: Monitor the launch

### Check preconversion progress

Query the currency state during the preconversion window to see reserves accumulating:

```
getcurrencystate "TB1" "1000390,1000413,5"
```

| Height | VRSCTEST reserves | USD reserves | Event |
|--------|------------------|-------------|-------|
| 1000390 | 0 | 0 | Just defined — no preconversions processed yet |
| 1000400 | 5 | 29.47 | `initialcontributions` processed |
| 1000410 | 6.9995 | 41.257 | External preconversions landed |

### Check launch status

After `startblock`:

```
getcurrency "TB1"
```

- `flags: 49` in `bestcurrencystate` — active fractional basket
- `flags: 3` — still in prelaunch (before `startblock`)
- `flags: 37` — launch failed (minpreconversion not met)

---

## Step 6: Verify

Confirm the basket is active and check the initial state:

```
getcurrency "TB1"
```

Check your balance:

```
getcurrencybalance "myid@"
```

Check reserve deposits:

```
getreservedeposits "TB1"
```

---

## After launch

### Convert reserve into basket tokens

```
sendcurrency "myid@" '[{"currency":"vrsctest","amount":1,"convertto":"TB1","address":"myid@"}]'
```

### Convert basket tokens back to a reserve

```
sendcurrency "myid@" '[{"currency":"TB1","amount":10,"convertto":"vrsctest","address":"myid@"}]'
```

### Convert between reserves via the basket

```
sendcurrency "myid@" '[{"currency":"vrsctest","amount":1,"convertto":"USD","via":"TB1","address":"myid@"}]'
```

### Estimate a conversion before committing

```
estimateconversion '{"currency":"vrsctest","amount":1,"convertto":"TB1"}'
```

---

## Design considerations

### Reserve ratio

The ratio of reserves to supply determines price stability. Parameters that lower the reserve ratio make the basket more volatile:

- `preallocations` — mint tokens without adding reserves
- `prelaunchcarveout` — redirect a fraction of preconverted reserves to rootID
- `prelaunchdiscount` — give preconverters a discount

The daemon enforces a minimum reserve ratio of 5%. Model the combined effect of these parameters before launch.

### Decentralized fee burning

With `proofprotocol: 1` (the default), sub-ID registration fees are burned — less supply with the same reserves makes the basket worth more. This is a deflationary mechanism built into the sub-ID economy.

### One ID, one currency, once

Each VerusID can define at most one currency, and only once — even if the launch fails. The currency name is permanently consumed.

---

## See also

- [How to Launch a Centralized Token](launch-centralized-token.md) — simpler token without reserves
- [`definecurrency`](../../reference/multichain/definecurrency.md) — full parameter reference
- [`sendcurrency`](../../reference/multichain/sendcurrency.md) — preconverting, converting, sending
- [`getcurrencystate`](../../reference/multichain/getcurrencystate.md) — track reserve state over time
- [`getreservedeposits`](../../reference/multichain/getreservedeposits.md) — check reserve UTXOs
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — how the launch process works
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how basket pricing works
