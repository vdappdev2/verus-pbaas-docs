# getidentitycontent

> Read cumulative identity content with optional VDXF key filter.

**Category:** Identity
**Daemon help:** `verus help getidentitycontent`

---

## Summary

Returns the cumulative content stored on a VerusID across all updates within a block height range. Unlike [`getidentity`](getidentity.md) which shows only the most recent update's `contentmultimap`, this RPC aggregates content from all revisions — providing the full content history.

Supports filtering by VDXF key to retrieve only specific application data.

---

## Syntax

```
getidentitycontent "identity" (heightstart) (heightend) (txproofs) (txproofheight) (vdxfkey) (keepdeleted)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `identity` | string | Yes | — | VerusID name or i-address |
| 2 | `heightstart` | number | No | 0 | Start of block height range |
| 3 | `heightend` | number | No | max | End of block height range. `-1` for mempool. |
| 4 | `txproofs` | boolean | No | `false` | Include transaction proofs |
| 5 | `txproofheight` | number | No | — | Height for proof generation |
| 6 | `vdxfkey` | string | No | — | Filter to a specific VDXF key (i-address). Only entries under this outer key are returned. |
| 7 | `keepdeleted` | boolean | No | `false` | Include items marked as deleted |

---

## Return value

Returns an object with cumulative content aggregated across all updates in the height range. The structure mirrors `contentmultimap` — outer VDXF keys mapping to arrays of entries.

When `vdxfkey` is specified, only entries under that key are returned, across all updates.

---

## Important behaviors

- **Cumulative view.** Content from earlier updates is included even if a later update cleared the `contentmultimap`. This is because content is an immutable append-only ledger — updates add new revisions but never erase prior ones from the chain.
- **VDXF key filter** works correctly across updates. If update 1 wrote content under key A and update 2 wrote content under key B, filtering by key A returns only update 1's content.
- **Deleted content** is excluded by default. Pass `keepdeleted: true` to include items marked with the `vrsc::identity.multimapremove` key.

> **Warning:** Malformed `multimapremove` entries can permanently break `getidentitycontent` for an identity (causes "CBaseDataStream::read(): end of data" error). `getidentity` and `getidentityhistory` still work in this case.

---

## Examples

### All content

```
getidentitycontent "alice@"
```

### Filtered by VDXF key

```
getidentitycontent "alice@" 0 -1 false "iJvkQ3uTKmRoFiE3rtP8YJxryLBKu8enmX"
```

Returns only entries under the specified key (e.g., vtimestamp proof data).

---

## See also

- [`getidentity`](getidentity.md) — current state (most recent update only)
- [`getidentityhistory`](getidentityhistory.md) — per-revision snapshots
- [`getvdxfid`](getvdxfid.md) — resolve VDXF key names to i-addresses for filtering
- [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md) — content format and VDXF key system
