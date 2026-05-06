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
| 5 | `sourceoffunds` | string | No | `"*"` | Funding source. Supports wildcards (`"*"`, `"R*"`, `"i*"`), specific addresses (including z-addresses), and VerusID names. |

### Identity object

| Field | Type | Required | Behavior when omitted |
|-------|------|----------|----------------------|
| `name` | string | Yes | — |
| `parent` | string | Yes | — (always include for correct resolution) |
| `primaryaddresses` | string[] | No | **Preserved** from current state. 1–25 entries when set. R-addresses only. |
| `minimumsignatures` | number | No | **Preserved**. 1–13 when set, and must be ≤ `primaryaddresses.length`. |
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
- **Multisig caps are enforced at update time.** The same `M ≤ 13`, `N ≤ 25`, `M ≤ N` rules that apply at registration also apply here. Invalid combinations are rejected with `"Invalid identity"`. See [VerusID Multisig](../../concepts/verusid-multisig.md).
- **`contentmultimap` is an append-only ledger.** Previous content remains on-chain and accessible via [`getidentitycontent`](getidentitycontent.md) even after an update clears the current multimap.
- **Invalid contentmultimap formats** are silently ignored — the transaction confirms but content is not stored. A bare `{message: "text"}` entry is silently stripped, but the `{data: {...}}` envelope below is valid and recognized.
- **z-address as `sourceoffunds`** works for all identity write operations (update, revoke, recover) since fees are native coin.

---

## The `{data: {...}}` envelope

`contentmultimap` entries support a daemon-managed shorthand for storing structured or encrypted content: wrap a [`signdata`](../data/signdata.md)-style input object inside a `data` field.

```json
{
  "<outer-vdxfid>": [
    { "data": { "message": "..." } }
  ]
}
```

Inside `data: {...}` you can use any of `signdata`'s input keys: `message`, `messagehex`, `filename`, `vdxfdata`, `datahash`, plus optionally `encrypttoaddress`.

The daemon expands the envelope at write time:

| Input shape | On-chain DataDescriptor | Privacy |
|---|---|---|
| `{data: {<input>}}` (no `encrypttoaddress`) | `flags: 13` — encrypted, with `epk` and `ivk` published | Public-encrypted (opt-in viewing). Daemon generates a fresh ephemeral z-address, encrypts to it, discards the spending key, publishes the IVK. |
| `{data: {<input>, "encrypttoaddress": "zs1..."}}` | `flags: 5` — encrypted, with `epk` only | Private. Recipient must hold the z-address's spending key, EVK, or IVK to decrypt. |

The daemon auto-wraps the resulting DataDescriptor in the `i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv` inner key. Decrypt with [`decryptdata`](../data/decryptdata.md) — pass the descriptor + the `updateidentity` txid + `retrieve: true`.

**MMR support.** The envelope also accepts `createmmr` and `mmrdata` for batched leaf storage. The on-chain entry is a single flags:13 wrapper containing the full MMR descriptor; every leaf decrypts with the same outer IVK (no per-leaf SSK is surfaced).

### Envelope field acceptance

The envelope reuses [`signdata`](../data/signdata.md)'s input vocabulary, but not all fields work. The daemon enforces this because the envelope's encryption is signed with the funding wallet's pubkey rather than a VerusID — anything that requires identity-bound signing is rejected.

| Field | Envelope behavior |
|---|---|
| `message`, `messagehex`, `filename`, `vdxfdata`, `datahash` | Accepted — payload sources |
| `encrypttoaddress` | Accepted — switches to flags:5 (recipient-only, no IVK published) |
| `createmmr`, `mmrdata` | Accepted — produces a single flags:13 wrapper containing the full MMR descriptor; every leaf decrypts with the outer IVK |
| `vdxfkeys`, `vdxfkeynames`, `boundhashes`, multi-sig `signature` | **Rejected** with: *"When signing with public key and not identity, cannot include vdxf keys, vdxf key names, bound hashes, or multisig"* |
| `hashtype`, `prefixstring` | **Silently ignored** — accepted by the daemon but no observable effect on the encrypted output, on-chain shape, or decrypted plaintext |

> **Distinct from the manual `signdata` → `updateidentity` path.** The envelope is a single-call alternative; the manual path (encrypt with `signdata --encrypttoaddress` first, then store the resulting flags:5 DataDescriptor) is still required for binding, SSK-based selective disclosure, or VerusID-bound signing. The manual path also exhibits a flag mutation on storage (5 → 37) that the envelope path does not.

> Confirmed end-to-end on vrsctest, 2026-05-06: `message`, `filename` (1.3 KB PNG, byte-perfect round-trip), `createmmr` + 3-leaf `mmrdata`, and the rejection / silent-ignore behaviors above.

See [How to Publish Encrypted Data on an Identity](../../how-to/data/publish-encrypted-data-on-identity.md) for the full workflow.

---

## Examples

### Update content only

```
updateidentity '{"name":"alice","parent":"iJhCez...","contentmultimap":{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c":[{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c":"hello world"}]}}'
```

### Encrypted content via the `{data: {...}}` envelope

```
updateidentity '{"name":"alice","parent":"iJhCez...","contentmultimap":{"<your-vdxfid>":[{"data":{"message":"public-encrypted message"}}]}}'
```

Produces a `flags:13` DataDescriptor with on-chain `ivk`. Add `"encrypttoaddress":"zs1..."` inside `data` to switch to recipient-targeted mode (flags:5, no IVK published).

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
- [VerusID Multisig](../../concepts/verusid-multisig.md) — multisig caps and signing model
- [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md) — contentmultimap format
- [How to Store and Read Identity Content](../../how-to/identity/store-and-read-content.md) — step-by-step guide
- [How to Publish Encrypted Data on an Identity](../../how-to/data/publish-encrypted-data-on-identity.md) — the `{data: {...}}` envelope shorthand
