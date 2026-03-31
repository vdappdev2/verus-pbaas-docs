# getcurrencybalance

> Return the balance in all currencies held at an address.

**Category:** Multichain
**Daemon help:** `verus help getcurrencybalance`

---

## Summary

`getcurrencybalance` returns the multi-currency balance for any address in the wallet — transparent addresses, z-addresses, VerusIDs, or wildcards. The result is a map of currency names (or i-addresses) to amounts, showing every currency held at that address.

---

## Syntax

```
getcurrencybalance "address" (minconf) (friendlynames) (includeshared)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `address` | string or object | Yes | — | Address to query. See [Address formats](#address-formats). |
| 2 | `minconf` | number | No | `1` | Minimum confirmations required. `0` includes unconfirmed transactions. |
| 3 | `friendlynames` | boolean | No | `true` | Use currency names as keys. `false` uses i-addresses. |
| 4 | `includeshared` | boolean | No | `false` | Include outputs that can also be spent by others (e.g., shared custody). |

### Address formats

| Format | Description |
|--------|-------------|
| `"*"` | All wallet UTXOs — transparent, identity, and z-address balances combined. |
| `"R*"` | All transparent address UTXOs only. |
| `"i*"` | All identity address UTXOs only. |
| `"alice@"` | Specific VerusID balance. |
| `"R..."` | Specific transparent address. |
| `"zs1..."` | Specific Sapling z-address. |
| `{"address":"alice@","currency":"kaiju"}` | Object form: filter to a specific currency at a specific address. |

---

## Return value

Returns an object mapping currency names (or i-addresses if `friendlynames` is `false`) to amounts:

```json
{
  "VRSCTEST": 1670.83559684,
  "Kaiju": 815.95477887,
  "USD": 679.00157505,
  "bro.Kaiju": 1e-8
}
```

Each key is a currency name, each value is the balance in that currency at the queried address. All currencies with non-zero balances are included.

---

## Important behaviors

- **Only shows currencies with non-zero balances.** Currencies you don't hold are not listed.
- **Wildcard scoping affects results.** `"*"` includes all wallet addresses; `"i*"` includes only identity addresses; `"R*"` includes only transparent addresses. Balances may differ significantly between wildcards because different currencies may be held at different address types.
- **ID control tokens appear as satoshi balances.** Currencies like `bro.Kaiju: 0.00000001` (1 satoshi) are identity control tokens — whoever holds this satoshi controls the associated identity on the marketplace.
- **z-address balances require the spending key or viewing key.** If the wallet has only a viewing key for a z-address, spends cannot be detected and the reported balance may be higher than the actual balance.
- **Object form filters to one currency.** Passing `{"address":"alice@","currency":"kaiju"}` returns only the Kaiju balance at that address, which is more efficient than fetching all balances and filtering client-side.
- **`minconf: 0` includes unconfirmed.** Useful for checking pending conversions or recent sends that haven't confirmed yet.

---

## Examples

### All wallet balances

```
getcurrencybalance "*"
```

Returns every currency held across all wallet addresses. On a vrsctest wallet this might show:

```json
{
  "VRSCTEST": 1670.83559684,
  "Kaiju": 815.95477887,
  "USD": 679.00157505,
  "zuo": 4996.75,
  "bro.Kaiju": 1e-8
}
```

### Identity-only balances

```
getcurrencybalance "i*"
```

Returns balances held only at identity addresses — typically fewer currencies and lower amounts than `"*"` since some funds may be at transparent or z-addresses.

### Specific identity

```
getcurrencybalance "alice@"
```

### Filter to a specific currency

```
getcurrencybalance '{"address":"alice@","currency":"kaiju"}'
```

Returns only the Kaiju balance at alice@.

### Include unconfirmed

```
getcurrencybalance "*" 0
```

Includes balances from unconfirmed transactions (0 confirmations).

### Use i-addresses instead of names

```
getcurrencybalance "*" 1 false
```

Returns balances keyed by i-address:

```json
{
  "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV": 1670.83559684,
  "i9kVWKU2VwARALpbXn4RS9zvrhvNRaUibb": 815.95477887
}
```

---

## See also

- [`sendcurrency`](sendcurrency.md) — send, convert, or export currencies
- [`getcurrency`](getcurrency.md) — inspect a currency's definition and state
- [`listtransactions`](listtransactions.md) — view transaction history
