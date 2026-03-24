# getidentityhistory

> View per-revision identity snapshots.

**Category:** Identity
**Daemon help:** `verus help getidentityhistory`

---

## Summary

Returns an array of identity snapshots, one per update transaction, within a block height range. Each entry's `contentmultimap` shows only what was set in that specific update — not the cumulative state. Use this for auditing changes over time.

For cumulative content, use [`getidentitycontent`](getidentitycontent.md) instead.

---

## Syntax

```
getidentityhistory "identity" (heightstart) (heightend) (txproofs)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `identity` | string | Yes | — | VerusID name or i-address |
| 2 | `heightstart` | number | No | 0 | Start of block height range |
| 3 | `heightend` | number | No | max | End of block height range |
| 4 | `txproofs` | boolean | No | `false` | Include transaction proofs |

---

## Return value

An array of identity snapshot objects. Each entry contains the full identity JSON as it existed at that revision, including `contentmultimap`, `primaryaddresses`, authorities, and all other fields.

---

## Important behaviors

- **Per-revision view.** Each entry's `contentmultimap` contains exactly what was set in that specific `updateidentity` call. If an update omitted `contentmultimap`, that revision shows `{}`.
- **Includes all state changes** — authority changes, address changes, content updates, revocations, and recoveries all appear as separate entries.
- **Useful for auditing** transfers (primaryaddress changes), content history, and authority changes.

---

## Example

```
getidentityhistory "alice@" 985000 990000
```

---

## See also

- [`getidentity`](getidentity.md) — current state only
- [`getidentitycontent`](getidentitycontent.md) — cumulative content with VDXF key filtering
