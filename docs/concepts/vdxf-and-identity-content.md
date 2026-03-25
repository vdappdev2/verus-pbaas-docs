# VDXF and Identity Content

VDXF (Verus Data Exchange Format) is a universal addressing system for on-chain data. It provides a way to store, categorize, and retrieve structured data on VerusIDs using deterministic key addresses. Applications use VDXF keys to read and write typed data without ambiguity.

---

## How VDXF keys work

A VDXF key is an i-address derived from a URI string. The derivation is deterministic — the same URI always produces the same i-address, on any chain, at any time. Resolve URIs to i-addresses with [`getvdxfid`](../reference/identity/getvdxfid.md).

### URI format

```
namespace::group.subgroup.name
```

- **`vrsc::data.type.string`** — system-defined string type under the VRSC namespace
- **`vtimestamp.vrsc::proof.basic`** — application-defined key under the vtimestamp namespace
- **`mcp3.vrsctest::test.vdxfformat`** — custom key under the mcp3 currency namespace

Any ID namespace can define keys. This means applications and communities can create their own structured data schemas without coordination.

### System-defined keys

The `vrsc::` namespace provides standard primitive types and structural types:

**Primitive types** — tell applications how to interpret raw values:

| URI | i-address | Use |
|-----|-----------|-----|
| `vrsc::data.type.string` | `iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c` | UTF-8 text |
| `vrsc::data.type.uint256` | `i8k7g7z6grtGYrNZmZr5TQ872aHssXuuua` | 256-bit hash |
| `vrsc::data.type.bytevector` | `iKMhRLX1JHQihVZx2t2pAWW2uzmK6AzwW3` | Raw bytes |

**Object types** — structured objects with metadata:

| URI | i-address | Use |
|-----|-----------|-----|
| `vrsc::data.type.object.datadescriptor` | `i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv` | Data with version, flags, mimetype, label |
| `vrsc::data.type.object.url` | `iJ7xdhJTJAvJubNnSJFXyA3jujzqGxjLuZ` | URL reference |

**Identity keys** — special operations on identity content:

| URI | i-address | Use |
|-----|-----------|-----|
| `vrsc::identity.multimapremove` | `i5Zkx5Z7tEfh42xtKfwbJ5LgEWE9rEgpFY` | Mark content as deleted |
| `vrsc::identity.profile.media` | `iEYsp2njSt1M4EVYi9uuAPBU2wpKmThkkr` | Profile media |

---

## Contentmultimap

The `contentmultimap` field on a VerusID is where VDXF-keyed data lives. Its structure:

```json
{
  "outerVdxfKey": [entry, entry, ...],
  "anotherVdxfKey": [entry, ...]
}
```

- **Outer key:** A VDXF key i-address identifying the application or category of content
- **Array of entries:** Multiple entries per key are supported

### Entry formats

Two confirmed valid formats for entries:

**1. Simple typed value**

```json
{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c": "hello world"}
```

The VDXF type key (`vrsc::data.type.string` in this case) tells applications how to interpret the value.

**2. DataDescriptor**

```json
{
  "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
    "version": 1,
    "objectdata": {"message": "some text"},
    "mimetype": "text/plain",
    "label": "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c"
  }
}
```

The DataDescriptor key (`i4GC1YGEVD21...`) wraps richer metadata: version, mimetype, label, and optional encryption fields (`salt`, `epk`, `ssk`). The daemon auto-sets `flags: 96` when mimetype and label are present.

**Best practice for labels:** Use the i-address of the VDXF key (e.g., `"label": "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c"`) rather than the string name. The i-address is deterministic and avoids resolution at read time.

### Invalid formats

- **Bare strings** in the array (e.g., `["text"]`) are silently ignored — the transaction confirms but content is not stored
- **`{message: "text"}`** (the `signdata` format) is NOT valid for contentmultimap entries — the daemon rejects with a misleading "contentmultimap may only be set in identities when PBaaS is active" error

---

## Reading content

Three RPCs provide different views:

| RPC | View | Use case |
|-----|------|----------|
| [`getidentity`](../reference/identity/getidentity.md) | Most recent update only | Quick check of current content |
| [`getidentitycontent`](../reference/identity/getidentitycontent.md) | Cumulative across all updates | Full content history, VDXF key filtering |
| [`getidentityhistory`](../reference/identity/getidentityhistory.md) | Per-revision snapshots | Audit trail, see exactly what each update changed |

### VDXF key filtering

[`getidentitycontent`](../reference/identity/getidentitycontent.md) accepts a `vdxfkey` parameter to return only entries under a specific outer key. This is how applications retrieve their own data without parsing unrelated content from other applications.

---

## Writing content

Use [`updateidentity`](../reference/identity/updateidentity.md) with `contentmultimap` in the identity JSON.

**Critical rule:** Omitting `contentmultimap` in an update **clears it to `{}`**. To add content without losing existing content, read the current state first with `getidentity`, merge your changes, then update with the combined content.

Content can also be set at registration time by including `contentmultimap` in the identity definition passed to [`registeridentity`](../reference/identity/registeridentity.md).

---

## Immutability model

Content is an **immutable append-only ledger**. Each `updateidentity` writes a new revision. Previous content is never deleted from the chain — only the most recent revision is visible in `getidentity`. All prior content remains accessible through `getidentitycontent` (cumulative) and `getidentityhistory` (per-revision).

Deletion is done via the `vrsc::identity.multimapremove` key — it marks content as deleted without removing it from the chain. `getidentitycontent` with `keepdeleted: false` (default) hides deleted items.

---

## Encrypted content

Content in `contentmultimap` is public by default — anyone can read it. For encrypted content:

1. Call [`signdata`](../reference/data/signdata.md) with `encrypttoaddress` (a Sapling z-address) to produce an encrypted DataDescriptor
2. Store the encrypted DataDescriptor in `contentmultimap` via `updateidentity`
3. Decrypt with `decryptdata` using the original descriptor from step 1

The `encrypttoaddress` field is a processing instruction for `signdata` — putting it directly in a contentmultimap DataDescriptor does nothing (silently ignored).

Decryption access control:
- **Wallet keys:** If the wallet holds the z-address, `decryptdata` auto-decrypts
- **Extended viewing key (EVK):** Grants read access without spending authority
- **Specific symmetric key (SSK):** Decrypts only the specific object it was generated for — enables selective disclosure

---

## See also

- [On-Chain Data Storage and Encryption](on-chain-data-storage-and-encryption.md) — the two storage paths (z-address vs. identity) and the encryption model
- [`getvdxfid`](../reference/identity/getvdxfid.md) — resolve VDXF URIs to i-addresses
- [`updateidentity`](../reference/identity/updateidentity.md) — write content
- [`getidentitycontent`](../reference/identity/getidentitycontent.md) — read content with filtering
- [How to Store and Read Identity Content](../how-to/identity/store-and-read-content.md) — step-by-step guide for public content
- [How to Encrypt Data on a Public Identity](../how-to/data/encrypt-data-on-public-identity.md) — step-by-step guide for encrypted content
