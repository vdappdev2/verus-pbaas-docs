# z_importviewingkey

> Import a viewing key to grant read-only decryption access for a z-address.

**Category:** Wallet
**Daemon help:** `verus help z_importviewingkey`

---

## Summary

Import an extended viewing key (EVK) obtained from [`z_exportviewingkey`](z_exportviewingkey.md). After import, the wallet can decrypt all data encrypted to that z-address — `decryptdata` works without passing the `evk` parameter explicitly. The imported key grants read-only access; it cannot spend funds.

Optionally triggers a wallet rescan to discover historical transactions associated with the viewing key.

---

## Syntax

```
z_importviewingkey "vkey" (rescan) (startHeight)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `vkey` | string | Yes | — | The viewing key to import (from `z_exportviewingkey`). Starts with `zxviews1q` for Sapling keys. |
| 2 | `rescan` | string | No | `"whenkeyisnew"` | Whether to rescan the wallet for transactions: `"yes"`, `"no"`, or `"whenkeyisnew"`. |
| 3 | `startHeight` | number | No | `0` | Block height to start the rescan from. Use this to skip scanning early blocks when the key is known to have no activity before a certain height. |

### Rescan options

| Value | Behavior |
|-------|----------|
| `"whenkeyisnew"` | Rescan only if this key has not been imported before. Avoids redundant rescans on re-import. |
| `"yes"` | Always rescan, even if the key was previously imported. Use when re-importing with a lower `startHeight` to discover older transactions. |
| `"no"` | Skip rescan entirely. The key is added but historical transactions are not discovered. New transactions will be detected going forward. |

> Rescanning can take minutes if scanning a large block range. Use `startHeight` to limit the range when possible.

---

## Return value

```json
{
  "type": "sapling",
  "address": "zs1..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Key type: `"sapling"` or `"sprout"` |
| `address` | string | The z-address corresponding to the imported viewing key (for Sapling, the default address) |

---

## Important behaviors

- **Read-only.** The viewing key decrypts data but cannot spend funds or sign transactions.
- **Enables implicit decryption.** After import, `decryptdata` auto-decrypts data at this z-address without requiring the `evk` parameter — the wallet recognizes the key.
- **Scope is the entire z-address.** All past and future data encrypted to the z-address becomes readable. For per-object access, use an SSK instead (see [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md)).
- **Rescan is blocking.** When rescan is enabled, the call blocks until the scan completes. For large block ranges, this can take several minutes.

---

## Examples

### Import with default rescan

```
z_importviewingkey "zxviews1qd8ugja...5cf9"
```

Rescans if the key is new. Returns the z-address associated with the key.

### Import without rescan

```
z_importviewingkey "zxviews1qd8ugja...5cf9" no
```

### Import with rescan from a specific height

```
z_importviewingkey "zxviews1qd8ugja...5cf9" yes 900000
```

Scans only blocks from height 900000 onward — useful when you know the key had no activity before that point.

---

## See also

- [`z_exportviewingkey`](z_exportviewingkey.md) — export the viewing key for a z-address
- [`decryptdata`](decryptdata.md) — decrypt data using wallet keys or explicit EVK/SSK
- [`z_listreceivedbyaddress`](z_listreceivedbyaddress.md) — list data received at a z-address (works after importing the viewing key)
- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — the three access control levels (spending key, EVK, SSK)
- [How to Store and Retrieve Private Data](../../how-to/data/store-and-retrieve-private-data.md) — full round-trip using EVK for decryption
