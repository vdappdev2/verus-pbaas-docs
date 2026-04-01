# How to Convert Between Currencies

This guide covers discovering conversion paths, estimating output, executing conversions, and verifying results.

**Prerequisites:**
- A funded wallet with the source currency
- A VerusID or R-address to receive the converted currency

---

## Step 1: Find a conversion path

### Option A: Find baskets that connect two currencies

```
getcurrencyconverters '{"vrsc":"vUSDT.vETH"}'
```

Returns baskets that hold both currencies as reserves, sorted by reserve depth. Use the basket with the deepest reserves for the best price stability.

### Option B: Browse available baskets

```
listcurrencies '{"launchstate":"launched","systemtype":"local"}'
```

Filter for launched currencies on the local system. Check which reserves a basket holds with:

```
getcurrency "kaiju"
```

The `currencies` array and `reservecurrencies` in `bestcurrencystate` show what the basket can convert between.

---

## Step 2: Estimate the conversion

Before committing funds, preview the expected output:

### Reserve to basket (direct)

```
estimateconversion '{"currency":"vrsctest","amount":10,"convertto":"kaiju"}'
```

### Basket to reserve (direct)

```
estimateconversion '{"currency":"kaiju","amount":100,"convertto":"vrsctest"}'
```

### Reserve to reserve (via basket)

```
estimateconversion '{"currency":"vrsctest","amount":10,"convertto":"USD","via":"kaiju"}'
```

The return includes `estimatedcurrencyout` (expected output), `netinputamount` (input after fees), and `conversionfee`.

---

## Step 3: Execute the conversion

### Reserve to basket

```
sendcurrency "myid@" '[{"currency":"vrsctest","amount":10,"convertto":"kaiju","address":"myid@"}]'
```

### Basket to reserve

```
sendcurrency "myid@" '[{"currency":"kaiju","amount":100,"convertto":"vrsctest","address":"myid@"}]'
```

### Reserve to reserve via basket

```
sendcurrency "myid@" '[{"currency":"vrsctest","amount":10,"convertto":"USD","via":"kaiju","address":"myid@"}]'
```

This is a two-hop conversion: VRSCTEST → kaiju → USD. Both hops happen atomically in the same block. Only a single basket hop per conversion is supported.

### Multiple conversions in one transaction

Multiple conversions through different baskets can be batched in a single call:

```
sendcurrency "myid@" '[{"currency":"vrsctest","amount":5,"convertto":"kaiju","address":"myid@"},{"currency":"vrsctest","amount":5,"convertto":"testidx","address":"myid@"}]'
```

---

## Step 4: Track the operation

`sendcurrency` returns an operation ID. Check its status:

```
z_getoperationstatus
```

Wait for `"status": "success"` and note the `txid` in the result.

---

## Step 5: Verify the result

### Check your balances

```
getcurrencybalance "myid@"
```

### Inspect the transaction

```
gettransaction "txid-from-z_getoperationstatus"
```

---

## Fees

| Conversion type | Fee |
|-----------------|-----|
| Direct (reserve ↔ basket) | 0.025% |
| Via (reserve → reserve) | 0.05% (two hops) |

Fees are deducted from the input amount. To have the full amount arrive after conversion, add `"addconversionfees": true` to the output object — fees are then charged on top of the specified amount.

---

## Preconversion (before a currency launches)

To participate in a currency launch before its `startblock`:

```
sendcurrency "myid@" '[{"currency":"vrsctest","amount":5,"convertto":"TB1","preconvert":true,"address":"myid@"}]'
```

- `preconvert: true` — required; without it, the conversion is rejected because the currency is not yet active
- All preconverters get the same fair price — no advantage to early or late participation
- If `minpreconversion` thresholds are not met, the launch fails and funds are automatically refunded
- Use `refundto` to specify a refund address different from `fromaddress`

---

## See also

- [How to Launch a Decentralized Basket Currency](launch-basket-currency.md) — create a basket for others to convert through
- [`sendcurrency`](../../reference/multichain/sendcurrency.md) — full conversion parameter reference
- [`estimateconversion`](../../reference/multichain/estimateconversion.md) — preview conversion output
- [`getcurrencyconverters`](../../reference/multichain/getcurrencyconverters.md) — find conversion paths
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how basket pricing works
- [MEV-Resistant DeFi](../../concepts/mev-resistant-defi.md) — why all conversions in a block get the same price
