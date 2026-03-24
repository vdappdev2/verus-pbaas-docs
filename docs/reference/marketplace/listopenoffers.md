# listopenoffers

> List open offers from the current wallet.

**Category:** Marketplace
**Daemon help:** `verus help listopenoffers`

---

## Summary

Shows what this wallet has offered on the on-chain marketplace. Can filter by expired/unexpired status. Provides more detail than `getoffers` (full identity definitions, content) because the offers belong to the local wallet.

---

## Syntax

```
listopenoffers (unexpired) (expired)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `unexpired` | boolean | No | `true` | Include active (unexpired) offers |
| 2 | `expired` | boolean | No | `true` | Include expired offers |

---

## Return value

An array of offer objects, or `null` if no offers match the filter.

### Offer object

| Field | Type | Description |
|-------|------|-------------|
| `expires` | number | Block height at which this offer expires. Compare against `getblockcount` to determine if still active. |
| `txid` | string | Transaction ID of the offer. Pass to [`closeoffers`](closeoffers.md) to cancel. |
| `offer` | object | What the wallet is offering — full output details (see below) |
| `for` | object | What the wallet wants in return — full output details (see below) |

### Output detail object

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Output type (e.g., `"cryptocondition"`) |
| `identityprimary` | object | For identity offers: full identity definition including `name`, `identityaddress`, `parent`, `systemid`, `primaryaddresses`, `minimumsignatures`, `revocationauthority`, `recoveryauthority`, `contentmap`, `contentmultimap`, `privateaddress`, `timelock` |
| `commitmenthash` | object | For currency offers: includes `currencyvalues` map with currency amounts |
| `reserveoutput` | object | For currency requests in the `for` side: includes `currencyvalues` map |
| `spendableoutput` | boolean | Whether the output is spendable |
| `nativeout` | number | Native currency amount in the output |

### Edge cases

- **Both filters false** (`unexpired: false, expired: false`): Returns `null`.
- **Newly created offers** do not appear until they have at least 1 confirmation.

---

## Examples

### Identity-for-identity offer

```
listopenoffers
```

```json
[
  {
    "expires": 4444888,
    "txid": "ca6c5f5e1abc6e3a6d319ce0255f39dedd2c94d85a1e1ea8f872135c768f1769",
    "offer": {
      "type": "cryptocondition",
      "identityprimary": {
        "version": 3,
        "flags": 1,
        "primaryaddresses": ["RGGfvnyjF1is1jGwPKVSNSDNov6V4aBDMF"],
        "minimumsignatures": 1,
        "name": "uzo",
        "identityaddress": "iLWMiUUPoNCgJtJ54KVFaJrqujjxy5fuPv",
        "parent": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
        "systemid": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
        "contentmap": {},
        "contentmultimap": {},
        "revocationauthority": "iLWMiUUPoNCgJtJ54KVFaJrqujjxy5fuPv",
        "recoveryauthority": "iLWMiUUPoNCgJtJ54KVFaJrqujjxy5fuPv",
        "timelock": 0
      },
      "spendableoutput": false,
      "nativeout": 0.00010000
    },
    "for": {
      "type": "cryptocondition",
      "identityprimary": {
        "version": 3,
        "flags": 1,
        "primaryaddresses": ["RLvbLo42FjR3PMQZg6281zgLGGq3FgMG5Q"],
        "minimumsignatures": 1,
        "name": "zuo",
        "identityaddress": "i4tbBJFfXTNJHY3iTd2wrVYCKUf8cc3uAm",
        "parent": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
        "systemid": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
        "contentmap": {},
        "contentmultimap": {},
        "revocationauthority": "i6mRtX44LuUqrvAJ2tap3deziWxN3kAcQE",
        "recoveryauthority": "i6mRtX44LuUqrvAJ2tap3deziWxN3kAcQE",
        "privateaddress": "zs1nz3h63whuywwn0tfdzjp9908rl0v2apet089k9fhxu86wv8udn08ryly3aptcmrx4jugsxgjz2c",
        "timelock": 0
      },
      "spendableoutput": false,
      "nativeout": 0.00000000
    }
  }
]
```

### Only expired offers

```
listopenoffers false true
```

---

## See also

- [`getoffers`](getoffers.md) — discover offers from all participants, not just your wallet
- [`closeoffers`](closeoffers.md) — cancel active offers or reclaim funds from expired ones
- [`makeoffer`](makeoffer.md) — create a new offer
