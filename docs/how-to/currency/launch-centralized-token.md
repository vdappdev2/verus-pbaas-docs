# How to Launch a Centralized Token

This guide walks through launching a centralized simple token on Verus — a token where the rootID can mint and burn supply after launch.

**Prerequisites:**
- A VerusID to use as the currency namespace (see [How to Register a VerusID](../identity/register-identity.md))
- Wallet funds: 200 VRSC launch fee + transaction fees (~0.0002)

---

## Step 1: Define the currency

### Basic token with preallocated supply

```
definecurrency '{"name":"mybrand","options":32,"proofprotocol":2,"preallocations":[{"mybrand@":10000}]}'
```

- `options: 32` — simple token
- `proofprotocol: 2` — centralized (rootID can mint/burn)
- `preallocations` — 10,000 tokens go to mybrand@ at launch

### With preconversion funding

Accept preconversions as a funding mechanism — proceeds go to the rootID:

```
definecurrency '{"name":"mybrand","options":32,"proofprotocol":2,"currencies":["vrsctest"],"conversions":[0.1],"minpreconversion":[100],"preallocations":[{"mybrand@":5000}]}'
```

- `currencies` + `conversions` — accept VRSCTEST at 10 tokens per 1 VRSCTEST
- `minpreconversion` — launch fails if less than 100 VRSCTEST is preconverted

### With referral discounts for sub-IDs

```
definecurrency '{"name":"mybrand","options":40,"proofprotocol":2,"preallocations":[{"mybrand@":10000}],"idregistrationfees":50,"idreferrallevels":3}'
```

- `options: 40` — TOKEN (32) + IDREFERRALS (8)
- Sub-IDs cost 50 mybrand tokens, with 3 referral levels

### With an endblock (planned decentralization)

```
definecurrency '{"name":"mybrand","options":32,"proofprotocol":2,"preallocations":[{"mybrand@":10000}],"endblock":1000000}'
```

After block 1,000,000, minting is permanently disabled. Minimum 480 blocks between `startblock` and `endblock`.

---

## Step 2: Broadcast

`definecurrency` returns `{txid, tx, hex}`. Broadcast the hex:

```
sendrawtransaction "hex-from-definecurrency"
```

---

## Step 3: Wait for launch

The currency launches at `startblock` (defaults to ~20 blocks after broadcast). If you defined `currencies` and `conversions`, participants can preconvert during this window.

---

## Step 4: Verify

After `startblock`:

```
getcurrency "mybrand"
```

Check your balance:

```
getcurrencybalance "mybrand@"
```

---

## After launch

### Mint more tokens

```
sendcurrency "mybrand@" '[{"address":"mybrand@","amount":1000,"currency":"mybrand","mintnew":true}]'
```

Can mint to any address:

```
sendcurrency "mybrand@" '[{"address":"alice@","amount":500,"currency":"mybrand","mintnew":true}]'
```

`fromaddress` must be the currency's own identity. Minting takes a few blocks to process.

### Burn tokens

Anyone holding the token can burn:

```
sendcurrency "mybrand@" '[{"address":"mybrand@","amount":100,"currency":"mybrand","burn":true}]'
```

### Send tokens

```
sendcurrency "mybrand@" '[{"address":"alice@","amount":50,"currency":"mybrand"}]'
```

---

## See also

- [`definecurrency`](../../reference/multichain/definecurrency.md) — full parameter reference
- [`sendcurrency`](../../reference/multichain/sendcurrency.md) — sending, minting, burning, preconverting
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — how the launch process works
