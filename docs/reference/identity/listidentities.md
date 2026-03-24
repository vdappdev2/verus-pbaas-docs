# listidentities

> List identities in the current wallet.

**Category:** Identity
**Daemon help:** `verus help listidentities`

---

## Summary

Returns all VerusIDs that the current wallet can spend for, sign for, or watch. Useful for discovering which identities are available in the wallet.

---

## Syntax

```
listidentities (includecanspend) (includecansign) (includewatchonly)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `includecanspend` | boolean | No | `true` | Include identities the wallet can spend for |
| 2 | `includecansign` | boolean | No | `true` | Include identities the wallet can sign for |
| 3 | `includewatchonly` | boolean | No | `false` | Include watch-only identities |

---

## Return value

An array of identity objects, each in the same format as [`getidentity`](getidentity.md) output (including `identity`, `status`, `canspendfor`, `cansignfor`).

---

## Example

```
listidentities
```

Watch-only only:

```
listidentities false false true
```

---

## See also

- [`getidentity`](getidentity.md) — look up a specific identity
