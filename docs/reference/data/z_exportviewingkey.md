# z_exportviewingkey

> Export the viewing key for a shielded z-address, granting read-only decryption access.

**Category:** Wallet
**Daemon help:** `verus help z_exportviewingkey`

---

## Summary

Export the extended viewing key (EVK) for a Sapling z-address. The viewing key allows decryption of all data encrypted to this address without granting spending authority. Share it to grant read-only access to a third party.

Pass the returned key as the `evk` parameter to [`decryptdata`](decryptdata.md), or import it on another node with `z_importviewingkey`.

---

## Syntax

```
z_exportviewingkey "zs1..."
```

---

## Parameters

| # | Name | Type | Required | Description |
|---|------|------|----------|-------------|
| 1 | `zaddr` | string | Yes | The Sapling z-address to export the viewing key for. Must be in the wallet. |

---

## Return value

```
"zxviews1q..."
```

A string containing the extended full viewing key. The key format starts with `zxviews1q` for Sapling addresses.

---

## Important behaviors

- **Read-only access.** The viewing key decrypts all data encrypted to this z-address but cannot spend funds or sign transactions.
- **Scope is the entire z-address.** Anyone with the EVK can decrypt all past and future data sent to this z-address. For per-object access control, use the SSK (specific symmetric key) from [`signdata`](signdata.md) output instead.
- **The z-address must be in the wallet.** You can only export viewing keys for addresses the wallet holds the spending key for.

---

## Example

```
z_exportviewingkey "zs18wsahlhespjr3rj8sq5zy57psy0nf9854szl4r3w727mwhzpcxzxemsggg9ngq2jyt8tj2gmzsw"
```

Returns:

```
"zxviews1qd8ugja...5cf9"
```

> Output from vrsctest, 2026-03-24.

---

## See also

- [`decryptdata`](decryptdata.md) — pass the EVK as the `evk` parameter for decryption
- [`z_listreceivedbyaddress`](z_listreceivedbyaddress.md) — list data received at a z-address
- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — the three access control levels (spending key, EVK, SSK)
- [How to Store and Retrieve Private Data](../../how-to/data/store-and-retrieve-private-data.md) — uses EVK in the decryption step
