# sendcurrency

> Send, convert, export, mint, burn, or store data — the primary RPC for moving value on Verus.

**Category:** Multichain
**Daemon help:** `verus help sendcurrency`

---

## Summary

`sendcurrency` handles simple sends, currency conversions through fractional baskets, cross-chain transfers, currency and identity exports, minting, burning, and on-chain data storage — all through a single command with a flexible output array.

Returns an async operation ID. Poll with `z_getoperationstatus` to get the resulting transaction ID.

---

## Syntax

```
sendcurrency "fromaddress" '[{"address":... ,"amount":...},...]' (minconfs) (feeamount) (returntxtemplate)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `fromaddress` | string | Yes | — | Source of funds. See [fromaddress details](#fromaddress-details). |
| 2 | `outputs` | array | Yes | — | Array of output objects. Multiple outputs are batched into a single transaction. See [Output object](#output-object). |
| 3 | `minconfs` | number | No | `1` | Minimum confirmations on source UTXOs. `0` allows spending unconfirmed outputs, useful for chaining operations quickly. |
| 4 | `feeamount` | number | No | default miner fee | Explicit fee amount. For cross-chain sends, can be denominated in the destination chain's native currency (via `feecurrency` in the output). |
| 5 | `returntxtemplate` | boolean | No | `false` | Return unsigned transaction hex and required input totals instead of broadcasting. For multi-sig, distributed authority signing, or inspection. |

### fromaddress details

| Format | Description |
|--------|-------------|
| `"*"` | All wallet UTXOs |
| `"R*"` | All transparent address UTXOs |
| `"i*"` | All identity address UTXOs |
| `"alice@"` | Only UTXOs on this identity's i-address |
| `"alice@:private"` | Source from the identity's assigned z-address. Enables the `memo` field. |
| `"zs1..."` | Specific Sapling z-address |
| `"R..."` | Specific transparent address |

`fromaddress` also serves as the default change address — any unspent value (including non-sent currencies in multi-currency UTXOs) returns here.

Acts as an authorization gate for certain modes: `mintnew` requires the currency's own identity as `fromaddress`.

### Output object

Each object in the `outputs` array describes one destination and operation.

#### Core fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `address` | string | Yes | — | Destination: VerusID (`alice@`), i-address, R-address, or z-address. For cross-chain: must be valid on the destination chain. |
| `amount` | number | Yes | — | Amount in the specified `currency` (or native if omitted). 8 decimal places. Can be `0` for export-only or data-only operations. |
| `currency` | string | No | native | Source currency name or i-address. Case-insensitive. |

#### Conversion fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `convertto` | string | — | Target currency for on-chain conversion. Valid: reserve-to-basket, basket-to-reserve, or reserve-to-reserve (requires `via`). |
| `via` | string | — | Fractional basket to convert through for reserve-to-reserve conversion. Only a single basket hop per conversion. |
| `addconversionfees` | boolean | `false` | Add fees on top so the full `amount` arrives after conversion. Without this, fees are deducted from `amount`. |
| `preconvert` | boolean | `false` | Participate in a currency launch by converting before the start block. |

#### Cross-chain fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `exportto` | string | — | Destination chain or system (e.g., `"vDEX"`, `"vETH"`, any PBaaS chain). |
| `exportcurrency` | boolean | `false` | Export the currency definition to the destination chain. One-time per currency per destination. Anyone can do this. |
| `exportid` | boolean | `false` | Export the VerusID definition to the destination chain. The exported ID becomes an independent copy. |
| `feecurrency` | string | auto | Currency to pay fees in. Same-chain: only native is valid. Cross-chain: can be the destination chain's native currency. |

#### Supply control fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mintnew` | boolean | `false` | Mint new supply. `fromaddress` must be the currency's own identity. Currency must be centralized (`proofprotocol: 2`). |
| `burn` | boolean | `false` | Permanently destroy tokens. Anyone holding the token can burn. Currency must be a token, not a native chain currency. |

#### Other fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `memo` | string | — | Encrypted message attached to shielded outputs, visible only to recipient. Max 512 bytes. Only valid when `fromaddress` is a z-address or `"ID@:private"`. |
| `data` | object | — | Store data on-chain using the `signdata` object format. Requires a z-address destination — transparent and identity addresses are rejected. `amount` can be `0`. |
| `refundto` | string | `fromaddress` | Refund destination if the operation fails (e.g., currency launch doesn't meet minimum). |

---

## Return value

**Default:**

```
"opid-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

An operation ID. The transaction is broadcast asynchronously.

**With `returntxtemplate: true`:**

```json
{
  "outputtotals": {"currencyname": amount, ...},
  "hextx": "hexstring"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `outputtotals` | object | Map of currency names to total amounts needed as inputs |
| `hextx` | string | Raw transaction hex with outputs and no inputs. Sign with `signrawtransaction`, broadcast with `sendrawtransaction`. |

---

## Async workflow

`sendcurrency` does not return a txid directly:

1. **Call `sendcurrency`** — returns an `opid`
2. **Poll `z_getoperationstatus`** with the opid — status cycles through `"queued"` → `"executing"` → `"success"` or `"failed"`
3. **On success:** `result.txid` contains the broadcast transaction ID
4. **Optionally call `gettransaction`** with the txid for full details

---

## Modes of operation

### 1. Simple send

Transfer currency on the same chain.

**Required:** `address`, `amount`

```
sendcurrency "*" '[{"currency":"vrsctest","amount":10,"address":"alice@"}]'
```

### 2. On-chain conversion

Convert through a fractional basket.

**Required:** `address`, `amount`, `currency`, `convertto`

**Direct** (reserve to basket or vice versa):

```
sendcurrency "alice@" '[{"currency":"vrsctest","amount":0.1,"convertto":"kaiju","address":"alice@"}]'
```

**Via** (reserve to reserve through a basket):

```
sendcurrency "alice@" '[{"currency":"vrsctest","amount":2,"convertto":"usd","via":"kaiju","address":"alice@"}]'
```

All conversions in the same block execute at the same price — no front-running is possible. See [MEV-Resistant DeFi](../../concepts/mev-resistant-defi.md). Use `estimateconversion` to preview expected output before committing.

### 3. Cross-chain transfer

Send currency to another chain.

**Required:** `address`, `amount`, `currency`, `exportto`

The currency definition must already exist on the destination chain. If not, export it first (mode 5).

```
sendcurrency "alice@" '[{"currency":"vrsctest","amount":100,"exportto":"vDEX","address":"alice@"}]'
```

### 4. Cross-chain conversion

Combine conversion and cross-chain transfer.

**Required:** `address`, `amount`, `currency`, `convertto`, `exportto`

### 5. Currency definition export

Export a currency definition to another chain. One-time per currency per destination. No value is transferred.

```
sendcurrency "alice@" '[{"currency":"mycurrency","amount":0,"exportto":"vDEX","exportcurrency":true,"address":"alice@"}]'
```

### 6. Identity export

Export a VerusID to another chain. The currency must already exist on the destination chain.

```
sendcurrency "alice@" '[{"currency":"vrsc","amount":0,"exportto":"vDEX","exportid":true,"address":"alice@"}]'
```

The exported ID becomes an independent copy with its own history, authorities, and content from that point forward.

### 7. Pre-conversion

Participate in a currency launch by converting at the initial price.

**Required:** `address`, `amount`, `currency`, `convertto`, `preconvert: true`

Only valid if the transaction is mined before the currency's start block. If the launch fails (minimum pre-conversion not met), funds are automatically refunded to `refundto` (or `fromaddress`).

### 8. Minting

Create new supply of a centralized currency.

**Required:** `address`, `amount`, `currency`, `mintnew: true`
**Constraint:** `fromaddress` must be the currency's own identity. Currency must have `proofprotocol: 2`.

```
sendcurrency "broom@" '[{"address":"broom@","amount":100,"currency":"broom","mintnew":true}]'
```

Can mint to any address:

```
sendcurrency "broom@" '[{"address":"alice@","amount":50,"currency":"broom","mintnew":true}]'
```

Minting is processed via reserve transfers — tokens appear several blocks after the transaction confirms (observed: 3-5 blocks), not in the next block. If a currency specifies an `endblock`, minting ends permanently at that block.

### 9. Burning

Permanently destroy tokens.

**Required:** `address`, `amount`, `currency`, `burn: true`

```
sendcurrency "alice@" '[{"address":"alice@","amount":10,"currency":"broom","burn":true}]'
```

Balance is deducted immediately; supply update in `getcurrency` takes several blocks.

### 10. Data storage

Store arbitrary data on-chain. No value transfer required.

**Required:** `address` (must be a z-address), `data`

```
sendcurrency "alice@" '[{"address":"zs1...","currency":"vrsctest","amount":0,"data":{"address":"alice@","message":"hello"}}]'
```

The `data` object follows the [`signdata`](../data/signdata.md) format. Transparent and identity addresses are rejected as destinations — use a z-address directly or `"ID@:private"`. `fromaddress` does NOT need to be a z-address; the z-address requirement is on the destination only.

---

## Important behaviors

### Conversion mechanics

- All conversions in the same block execute at the same price — no ordering advantage, no front-running. See [MEV-Resistant DeFi](../../concepts/mev-resistant-defi.md).
- Conversion fees: **0.025%** for direct conversions (reserve-to-basket or vice versa), **0.05%** for via conversions (two hops). Fees are deducted from the input amount by default. Set `addconversionfees: true` to add fees on top.
- Converted output typically settles within 2–10 blocks depending on network activity.

### Multi-currency UTXO change

A single UTXO on Verus can hold multiple currencies (native coin + tokens). When a UTXO is spent, only the specified currency and amount go to the destination. All other currencies in the UTXO return as change to `fromaddress`.

Tokens in multi-currency UTXOs are never lost — they are automatically returned as change.

### Multiple outputs

The `outputs` array can contain multiple entries batched into one transaction: multiple recipients, multiple conversions (including through different baskets), or mixed operations (e.g., a send + a conversion).

### ID control tokens

Identities with `flags: 5` (combined bitfield: activated + control token) have a control token — a currency with the same name and a supply of exactly 0.00000001 (1 satoshi). Sending it via `sendcurrency` transfers revoke/recover authority. If an identity is exported before a control token is created, the exported copy cannot be controlled by the token.

### feecurrency restrictions

Same-chain sends only accept native currency for fees — non-native currencies are rejected ("Invalid fee currency specified"). `feecurrency` is only meaningful for cross-chain sends, where it can be set to the destination chain's native currency.

---

## Examples

### Simple send

```
sendcurrency "testidx@" '[{"currency":"vrsctest","amount":10,"address":"third.testidx@"}]'
```

Result: `"opid-f9791771-88fa-431f-8068-cbb305565675"`

### Conversion through fractional basket

```
sendcurrency "third.testidx@" '[{"currency":"vrsctest","amount":0.1,"convertto":"testidx","address":"third.testidx@"}]'
```

Polling `z_getoperationstatus`:

```json
{
  "id": "opid-f4422247-f37a-4e74-85ad-158202a45e49",
  "status": "success",
  "result": {
    "txid": "54fe6081008cc9553c30e3a34b4303bb79256804f6a0575735007085e1064de1"
  },
  "execution_secs": 0.315431548
}
```

### Via conversion (reserve-to-reserve)

```
sendcurrency "third.testidx@" '[{"currency":"vrsctest","amount":2,"convertto":"usd","via":"kaiju","address":"third.testidx@"}]'
```

### Minting

```
sendcurrency "broom@" '[{"address":"broom@","amount":100,"currency":"broom","mintnew":true}]'
```

Supply increased from 1000 to 1100. Processing took ~4 blocks.

### Burning

```
sendcurrency "broom@" '[{"address":"broom@","amount":10,"currency":"broom","burn":true}]'
```

Supply decreased from 1150 to 1140.

---

## See also

- `z_getoperationstatus` — poll for async result after `sendcurrency`
- `gettransaction` — full details of the resulting transaction
- `estimateconversion` — preview conversion output before sending
- `getcurrencyconverters` — discover fractional baskets for conversion paths
- `getcurrencybalance` — check balances before sending
- `getcurrency` — inspect currency definitions
- [`signdata`](../data/signdata.md) — data object format used by the `data` output field
- [MEV-Resistant DeFi](../../concepts/mev-resistant-defi.md) — why same-block pricing eliminates front-running
- [Fractional Basket Conversions](../../concepts/fractional-basket-conversions.md) — how reserve pricing works
- [Currency Launch Lifecycle](../../concepts/currency-launch-lifecycle.md) — preconversion, launch, and minting
- [Atomic Swaps on Verus](../../concepts/atomic-swaps.md) — alternative for trustless trading
