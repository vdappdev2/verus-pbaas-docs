# getidentity

> Look up a VerusID's current state.

**Category:** Identity
**Daemon help:** `verus help getidentity`

---

## Summary

Returns the full definition and current status of a VerusID: primary addresses, authorities, content, flags, timelock, and whether it can spend and sign. This is the primary read operation for identities.

---

## Syntax

```
getidentity "identity" (height) (txproof) (txproofheight)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `identity` | string | Yes | — | VerusID name (`"alice@"`) or i-address |
| 2 | `height` | number | No | current | Return identity state at this block height. `-1` for mempool. |
| 3 | `txproof` | boolean | No | `false` | Include a proof |
| 4 | `txproofheight` | number | No | — | Height for proof generation |

---

## Return value

```json
{
  "identity": {
    "version": 3,
    "flags": 0,
    "primaryaddresses": ["R..."],
    "minimumsignatures": 1,
    "name": "alice",
    "identityaddress": "i...",
    "parent": "i...",
    "systemid": "i...",
    "contentmap": {},
    "contentmultimap": {},
    "revocationauthority": "i...",
    "recoveryauthority": "i...",
    "privateaddress": "zs1...",
    "timelock": 0
  },
  "status": "active",
  "canspendfor": true,
  "cansignfor": true,
  "blockheight": 989000,
  "txid": "abc123...",
  "vout": 0
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"active"` or `"revoked"` |
| `canspendfor` | boolean | Whether the current wallet can authorize spending for this identity |
| `cansignfor` | boolean | Whether the current wallet can sign for this identity |
| `identity.flags` | number | `0` = normal, `2` = delay lock active, `5` = has control token, `32768` = revoked |
| `identity.contentmultimap` | object | Only shows the **most recent update's** content. Use [`getidentitycontent`](getidentitycontent.md) for cumulative history. |
| `identity.timelock` | number | `0` = unlocked. When `flags: 2`: delay in blocks. When `flags: 0` and `timelock > 0`: absolute block height lock. |

---

## Example

```
getidentity "alice@"
```

At a specific height:

```
getidentity "alice@" 985000
```

---

## See also

- [`getidentitycontent`](getidentitycontent.md) — cumulative content across all updates
- [`getidentityhistory`](getidentityhistory.md) — per-revision snapshots
- [`updateidentity`](updateidentity.md) — modify an identity
