# updateidentity

> Update an existing VerusID's addresses, content, authorities, or other mutable properties.

**Category:** Identity
**Daemon help:** `verus help updateidentity`

---

## Summary

Modify any mutable property of an existing VerusID: primary addresses, signing threshold, revocation/recovery authorities, private address, and content. The identity must be active (not revoked) and controlled by the current wallet.

---

## Syntax

```
updateidentity "jsonidentity" (returntx) (tokenupdate) (feeoffer) (sourceoffunds)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `jsonidentity` | object | Yes | — | Identity JSON with desired changes. See [Identity object](#identity-object). |
| 2 | `returntx` | boolean | No | `false` | Return hex instead of broadcasting. |
| 3 | `tokenupdate` | boolean | No | `false` | Use ID control token for authority instead of primary addresses. |
| 4 | `feeoffer` | number | No | standard fee | Non-standard fee. |
| 5 | `sourceoffunds` | string | No | `"*"` | Funding source. Supports wildcards (`"*"`, `"R*"`, `"i*"`, `"z*"`), specific addresses, and VerusID names. z-address works (fees are native coin). |

### Identity object

| Field | Type | Required | Behavior when omitted |
|-------|------|----------|----------------------|
| `name` | string | Yes | — |
| `parent` | string | Yes | — (always include for correct resolution) |
| `primaryaddresses` | string[] | No | **Preserved** from current state |
| `minimumsignatures` | number | No | **Preserved** |
| `revocationauthority` | string | No | **Preserved** |
| `recoveryauthority` | string | No | **Preserved** |
| `privateaddress` | string | No | **Preserved** (special: `null` clears it, `""` preserves it) |
| `contentmultimap` | object | No | **Cleared to `{}`** |
| `timelock` | number | No | **Preserved** (but cannot be removed via update — only via revoke+recover) |

---

## Field carry-over rules

Most fields are preserved when omitted — they carry over from the current on-chain state. The critical exception:

- **`contentmultimap`** — omitting it **clears content to `{}`**. To preserve existing content across updates, read it first with [`getidentity`](getidentity.md) and include it in the update.

**`privateaddress` has unique semantics:**

| Value | Effect |
|-------|--------|
| Omitted | Preserved (existing address kept) |
| `""` (empty string) | Preserved (same as omitting) |
| `null` | **Cleared** (address removed) |
| `"zs1..."` | **Changed** to new address |

**`name` casing** can be changed (e.g., `"fourth"` → `"FOURTH"`). The i-address is unchanged. `parent` casing cannot be changed — it resolves by i-address.

---

## Return value

- **Default:** Transaction ID string
- **With `returntx: true`:** Hex string

---

## Important behaviors

- **Do not set `flags` to the revoked value (32768) directly.** Use [`revokeidentity`](revokeidentity.md) to revoke an identity. Setting the revoked flag via `updateidentity` bypasses the intended revocation workflow.
- **Cannot modify `timelock` via update.** Including `timelock: 0` on a locked identity is rejected. Use [`setidentitytimelock`](setidentitytimelock.md) or revoke+recover.
- **`contentmultimap` is an append-only ledger.** Previous content remains on-chain and accessible via [`getidentitycontent`](getidentitycontent.md) even after an update clears the current multimap.
- **Invalid contentmultimap formats** are silently ignored — the transaction confirms but content is not stored. The `{message: "text"}` format (used by `signdata`) is not valid for contentmultimap entries.
- **z-address as `sourceoffunds`** works for all identity write operations (update, revoke, recover) since fees are native coin.

---

## Examples

### Update content only

```
updateidentity '{"name":"alice","parent":"iJhCez...","contentmultimap":{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c":[{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c":"hello world"}]}}'
```

### Change primary addresses

```
updateidentity '{"name":"alice","parent":"iJhCez...","primaryaddresses":["RNewAddr..."],"minimumsignatures":1}'
```

### Delegate authorities

```
updateidentity '{"name":"alice","parent":"iJhCez...","revocationauthority":"guardian@","recoveryauthority":"backup@"}'
```

---

## See also

- [`getidentity`](getidentity.md) — read current state before updating
- [`getidentitycontent`](getidentitycontent.md) — read cumulative content history
- [VerusID Authority Model](../../concepts/verusid-authority-model.md) — safe authority configuration
- [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md) — contentmultimap format
- [How to Store and Read Identity Content](../../how-to/identity/store-and-read-content.md) — step-by-step guide
