# How to Publish Encrypted Data on an Identity

Store encrypted data in a VerusID's `contentmultimap` in a single `updateidentity` call using the `{data: {...}}` envelope. The daemon handles encryption, ephemeral key generation, and on-chain key publication.

Two modes, picked by whether you include `encrypttoaddress`:

| Mode | Input | On-chain | Who decrypts |
|---|---|---|---|
| **Public-encrypted** (default) | `{data: {<input>}}` | `flags: 13` with on-chain `ivk` | Anyone — the IVK is published |
| **Private-encrypted** | `{data: {<input>, "encrypttoaddress": "zs1..."}}` | `flags: 5` with `epk` only | Only the recipient z-address holder |

For the underlying model and trade-offs (why the IVK is safe to publish in default mode, when to combine with `signdata`, etc.), see [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md). For the full envelope field reference, see [`updateidentity`](../../reference/identity/updateidentity.md#the-data-envelope).

**Prerequisites:**
- A VerusID you control
- Sufficient funds at the funding source for the `updateidentity` transaction
- For the file input mode: the daemon started with `-enablefileencryption`

---

## Step 1: Choose your VDXF outer key

Pick a VDXF key under which to store the entry. Either reuse an existing application key or mint your own:

```
getvdxfid '"myid::myapp.contentkey"'
```

Returns a `vdxfid` (i-address) you'll use as the outer key in `contentmultimap`. The same key can hold multiple entries (it's a multimap).

---

## Step 2: Write the entry with `updateidentity`

Pass `{data: {<input>}}` inside `contentmultimap`. The daemon recognizes this shorthand and handles encryption, wrapping, and key generation:

```
updateidentity '{
  "name": "myid",
  "parent": "iJhCez...",
  "contentmultimap": {
    "<your-vdxfid>": [
      { "data": { "message": "content to publish" } }
    ]
  }
}'
```

The `data: {...}` object accepts the same input keys as [`signdata`](../../reference/data/signdata.md): `message`, `messagehex`, `filename`, `vdxfdata`, `datahash`, plus `encrypttoaddress` for private mode.

> **Always include existing `contentmultimap` entries** when updating — omitting `contentmultimap` clears all visible content. Read with `getidentity` first if you need to preserve prior entries.


---

## Step 3: Inspect what was stored

Read the identity to see the on-chain shape:

```
getidentity "myid@"
```

### Default mode (no `encrypttoaddress`)

```json
{
  "<your-vdxfid>": [
    {
      "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
        "version": 1,
        "flags": 13,
        "objectdata": "<ciphertext hex>",
        "epk": "<ephemeral public key>",
        "ivk": "<incoming viewing key, published>"
      }
    }
  ]
}
```

`flags: 13` = `HAS_OBJECTDATA | ENCRYPTED | HAS_IVK`. The published `ivk` is what makes the entry decryptable by any reader.

### With `encrypttoaddress`

```json
{
  "<your-vdxfid>": [
    {
      "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
        "version": 1,
        "flags": 5,
        "objectdata": "<ciphertext hex>",
        "epk": "<ephemeral public key>"
      }
    }
  ]
}
```

`flags: 5` = `HAS_OBJECTDATA | ENCRYPTED`. No `ivk` — the recipient must hold the spending key, EVK, or IVK out-of-band.

> **Reading superseded entries.** `getidentity` returns only the current `contentmultimap`. To retrieve an entry that was overwritten by a later `updateidentity`, use [`getidentitycontent`](../../reference/identity/getidentitycontent.md) — it aggregates content across all revisions in the given height range. `contentmultimap` is append-only on-chain, so prior revisions remain decryptable with their original IVK.

---

## Step 4: Decrypt the entry

Decryption follows the indirect-reference pattern: the on-chain DataDescriptor stores a reference; the actual ciphertext sits in the same `updateidentity` transaction. Pass the descriptor + the txid + `retrieve: true`.

### Default mode — anyone can decrypt with the on-chain IVK

```
decryptdata '{
  "datadescriptor": {
    "version": 1,
    "flags": 13,
    "objectdata": "<ciphertext>",
    "epk": "<epk>",
    "ivk": "<ivk from on-chain entry>"
  },
  "txid": "<updateidentity txid>",
  "retrieve": true
}'
```

The `ivk` is part of the on-chain descriptor — pull it from `getidentity` and pass it back in. No EVK export, no out-of-band key sharing required.

### Private mode — recipient decrypts via wallet or explicit key

If the wallet holds the recipient z-address's spending key:

```
decryptdata '{
  "datadescriptor": {
    "version": 1,
    "flags": 5,
    "objectdata": "<ciphertext>",
    "epk": "<epk>"
  },
  "txid": "<updateidentity txid>",
  "retrieve": true
}'
```

The wallet auto-decrypts. Without spending authority, pass an `evk` (from `z_exportviewingkey`) or `ivk` instead.

### Decrypted output

Both modes return the same shape:

```json
[
  {
    "version": 1,
    "flags": 2,
    "objectdata": "<plaintext as hex>",
    "salt": "<auto-generated salt>"
  }
]
```

`flags: 2` indicates decrypted output. Decode `objectdata` from hex to recover the original content (text via `xxd -r -p`, binary via `bytes.fromhex()` or equivalent).

---

## Storing an MMR

The envelope accepts `createmmr` and `mmrdata` for batched leaf storage:

```
updateidentity '{
  "name": "myid",
  "parent": "iJhCez...",
  "contentmultimap": {
    "<your-vdxfid>": [
      {
        "data": {
          "createmmr": true,
          "mmrdata": [
            { "message": "leaf 1" },
            { "message": "leaf 2" },
            { "message": "leaf 3" }
          ]
        }
      }
    ]
  }
}'
```

The on-chain entry is a single flags:13 wrapper. Decrypting with the wrapper's IVK + the txid + `retrieve: true` returns the full MMR structure: `mmrroot`, `mmrhashes`, and a `datadescriptors` array of per-leaf flags:5 entries. Every per-leaf entry decrypts with the same outer IVK — the envelope path is all-or-nothing for the MMR.

> **No SSK is surfaced.** `updateidentity` returns only a txid, so the per-object SSK that `signdata` would otherwise produce is never returned to the caller. If you need per-leaf or per-call SSK selective disclosure on top of an MMR, use the manual [`signdata`](../../reference/data/signdata.md) → store path described in [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md).

> Confirmed on vrsctest, 2026-05-06: 3-leaf MMR encrypted via the envelope; all three leaves decrypted with the published outer IVK.

---

## Storing files

The `filename` input mode reads a local file from disk and encrypts it as a single payload:

```
updateidentity '{
  "name": "myid",
  "parent": "iJhCez...",
  "contentmultimap": {
    "<your-vdxfid>": [
      { "data": { "filename": "/path/to/image.png" } }
    ]
  }
}'
```

**Requires** the daemon flag `-enablefileencryption` (startup or config). Decryption returns the raw bytes as hex `objectdata` — write them back to a file with `xxd -r -p > out.png` to reconstruct.

> Confirmed byte-perfect round-trip with a 1.3 KB PNG on vrsctest, 2026-05-06: SHA-256 matched the original after publish + retrieve + decrypt.

**Size limits:** the daemon caps payloads at 1,000,000 bytes (1 MB) before encryption. Larger files are rejected. Fees scale with transaction size — a small text payload costs ~0.0035 VRSCTEST; a ~900 KB file costs ~9.5 VRSCTEST.

---

## Privacy in private mode

In private mode, the IVK is not published — but the z-address you specified in `encrypttoaddress` is now part of the recipient's privacy budget. If that address ever holds other shielded value or receives other notes, anyone with its IVK or EVK gains read access to everything there, including this entry.

**Practical rule:** for private mode, encrypt to a z-address dedicated to a single recipient context. Don't reuse a wallet's main `id@:private` for one-off `encrypttoaddress` outputs if that address handles other shielded value.

Default mode does not have this concern — the daemon uses a disposable ephemeral z-address per call. See [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) for the underlying model.

---

## When to use which mode

| Goal | Mode | Why |
|---|---|---|
| Typical application data on a VerusID — profiles, attestations, application state | **Default `{data: {...}}`** (recommended) | Recommended default for app data. Every consumer reads through the same `decryptdata` entry path against an on-chain DataDescriptor, regardless of which application wrote the entry. |
| Data that must be readable by generic chain consumers without `decryptdata` — cross-chain readers, block explorers, third-party indexers consuming raw `cmm` | Plaintext (manual cmm, no `data:` envelope) | Opt-out for composability priority — the raw `cmm` value is directly consumable by readers that don't (or can't) call `decryptdata`. |
| Content meant for a specific recipient only | `{data: {..., "encrypttoaddress": "zs1..."}}` | Recipient-targeted, no IVK published |
| Selective per-object disclosure to multiple parties | `signdata --encrypttoaddress` then manual cmm store | Returns a per-object SSK alongside the encrypted descriptor — share different SSKs with different parties. See [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md). |

---

## See also

- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — the broader encryption model and the three storage tiers
- [How to Publish Encrypted Data on an Identity from a Wallet App](publish-encrypted-data-from-wallet.md) — the same on-chain shape via `IdentityUpdateRequest` and Verus Mobile (single-object envelope, `messagehex`)
- [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md) — the manual `signdata` → `updateidentity` path for SSK-based selective disclosure
- [How to Store and Retrieve Private Data](store-and-retrieve-private-data.md) — the z-address path via `sendcurrency:data`
- [How to Grant Read Access to Encrypted Data](grant-read-access.md) — sharing EVKs and SSKs
- [`updateidentity`](../../reference/identity/updateidentity.md) — the `{data: {...}}` envelope shape
- [`decryptdata`](../../reference/data/decryptdata.md) — decryption with `retrieve: true` + txid
- [`signdata`](../../reference/data/signdata.md) — input vocabulary shared with the `data` envelope
