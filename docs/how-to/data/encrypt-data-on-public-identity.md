# How to Encrypt Data on a Public Identity

Store encrypted data in a VerusID's `contentmultimap` so that the ciphertext is publicly visible on-chain but only authorized parties can decrypt it. This enables selective disclosure — share decryption keys with specific parties while the data remains publicly anchored.

> **This how-to is for the identity content path**, where encryption is manual. For private data storage to z-addresses, use `sendcurrency:data` directly — it encrypts automatically and does not require `signdata`. See [How to Store and Retrieve Private Data](store-and-retrieve-private-data.md).

**Prerequisites:**
- A VerusID you control
- A Sapling z-address in the wallet (the encryption target — generate one with `z_getnewaddress`)
- Familiarity with [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md)

---

## Step 1: Encrypt the data with `signdata`

Call `signdata` with your data and `encrypttoaddress` set to a Sapling z-address. The z-address determines who can decrypt — anyone with its spending key, viewing key, or the per-object SSK.

```
signdata '{
  "address": "myid@",
  "message": "confidential content here",
  "encrypttoaddress": "zs1..."
}'
```

The output contains both encrypted and plaintext versions:

**Encrypted DataDescriptor** — at `mmrdescriptor_encrypted.datadescriptors[0]`:

```json
{
  "version": 1,
  "flags": 5,
  "objectdata": "a3f7c9e1...",
  "epk": "02b4d8f3..."
}
```

- `flags: 5` — indicates encrypted data
- `objectdata` — the ciphertext (not human-readable)
- `epk` — the encryption public key (required for decryption)

**Specific symmetric key (SSK)** — at `signaturedata_ssk`:

```
"d4e2f8a1..."
```

Save this key. It decrypts only this specific object — share it to grant per-object read access.

**Plaintext version** — at `mmrdescriptor.datadescriptors[0]`:

```json
{
  "version": 1,
  "flags": 2,
  "objectdata": "636f6e666964656e7469616c20636f6e74656e742068657265",
  "salt": "a7b3c9d2..."
}
```

> **Save the encrypted DataDescriptor and the SSK.** You will need the encrypted DataDescriptor (with `flags: 5`, `objectdata`, and `epk`) for decryption later. Do not discard it after storing on-chain — the on-chain version has modified flags that break decryption.

---

## Step 2: Store the encrypted DataDescriptor on the identity

Write the encrypted DataDescriptor into `contentmultimap` via `updateidentity`. Wrap it in the DataDescriptor VDXF key (`i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv`) under an outer VDXF key of your choice.

First, read the current identity to preserve existing content:

```
getidentity "myid@"
```

Then update with the encrypted content merged in:

```
updateidentity '{
  "name": "myid",
  "parent": "iJhCez...",
  "contentmultimap": {
    "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c": [
      {
        "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
          "version": 1,
          "flags": 5,
          "objectdata": "a3f7c9e1...",
          "epk": "02b4d8f3...",
          "label": "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c"
        }
      }
    ]
  }
}'
```

The outer key (`iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c` = `vrsc::data.type.string`) is the application category key. Use any VDXF key appropriate for your application — resolve custom names with `getvdxfid`.

The `label` field identifies the semantic type of the entry. Best practice: use the i-address of the relevant VDXF key.

> **Important:** Include any existing `contentmultimap` entries in the update. Omitting `contentmultimap` clears all visible content to `{}`.

---

## Step 3: Verify the on-chain state

After 1 confirmation, inspect the identity:

```
getidentity "myid@"
```

The `contentmultimap` shows the ciphertext. The daemon modifies `flags` during storage:

```json
{
  "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
    "version": 1,
    "flags": 37,
    "objectdata": "a3f7c9e1...",
    "epk": "02b4d8f3..."
  }
}
```

Note `flags: 37` — the daemon adjusted from the original `5`. The ciphertext and `epk` are preserved. Anyone can see that encrypted data exists, but cannot read the content without the decryption key.

---

## Step 4: Decrypt the data

Use the **original** encrypted DataDescriptor from Step 1 (with `flags: 5`), not the on-chain version (which has `flags: 37`).

### With wallet keys (auto-decrypt)

If the wallet holds the z-address spending key:

```
decryptdata '{
  "datadescriptor": {
    "version": 1,
    "flags": 5,
    "objectdata": "a3f7c9e1...",
    "epk": "02b4d8f3..."
  }
}'
```

### With the SSK

If you do not hold the z-address but have the SSK from Step 1:

```
decryptdata '{
  "datadescriptor": {
    "version": 1,
    "flags": 5,
    "objectdata": "a3f7c9e1...",
    "epk": "02b4d8f3...",
    "ssk": "d4e2f8a1..."
  }
}'
```

Both return the decrypted content:

```json
[
  {
    "version": 1,
    "flags": 2,
    "objectdata": "636f6e666964656e7469616c20636f6e74656e742068657265",
    "salt": "a7b3c9d2..."
  }
]
```

Decode the hex `objectdata` to recover the original message.

> Both decryption methods (wallet keys and SSK) confirmed on vrsctest, 2026-03-23.

---

## Sharing access

| To grant | Share | Scope |
|---|---|---|
| Read access to all encrypted data at the z-address | Extended viewing key (`z_exportviewingkey "zs1..."`) | All objects encrypted to this z-address |
| Read access to one specific object | The SSK from `signdata` output | Only the specific object |
| No access | Nothing | Ciphertext visible on-chain but unreadable |

SSK-based selective disclosure is the primary pattern for encrypted identity content. Encrypt multiple objects to the same z-address, then share different SSKs with different parties.

---

## Common pitfalls

**Do not put `encrypttoaddress` in `contentmultimap`.** It is a processing instruction for `signdata`, not a DataDescriptor field. Placing it in a contentmultimap entry via `updateidentity` is silently ignored — the content is stored as plaintext with no error or warning. Always encrypt with `signdata` first, then store the output. (Confirmed on vrsctest, 2026-03-23.)

**Cache the original encrypted DataDescriptor.** The daemon modifies `flags` when storing DataDescriptors in `contentmultimap` (5 becomes 37). The `decryptdata` shortcut that queries identity content directly (`iddata` parameter with `identityid` + `vdxfkey` + `getlast`) fails for encrypted content stored via this flow — it returns "Invalid data descriptor or cannot decrypt." Applications must store the original descriptor from `signdata` output for reliable decryption. (Confirmed on vrsctest, 2026-03-23.)

**The full `mmrdescriptor` does not work with `decryptdata`.** Passing the complete `mmrdescriptor_encrypted` object (rather than just `datadescriptors[0]`) returns empty output with no error. Use the individual DataDescriptor. (Observed on vrsctest, 2026-03-23.)

**Omitting `contentmultimap` clears it.** When updating an identity, always include existing `contentmultimap` entries alongside the new encrypted content. Read the identity first with `getidentity` to capture existing content.

---

## See also

- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — how the two storage paths and encryption model work
- [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md) — contentmultimap format and VDXF keys
- [How to Store and Retrieve Private Data](store-and-retrieve-private-data.md) — the z-address path for fully private data
- [How to Sign and Verify Data](sign-and-verify-data.md) — `signdata` without encryption or storage
- [How to Grant Read Access to Encrypted Data](grant-read-access.md) — sharing EVKs and SSKs for selective disclosure
- [`signdata`](../../reference/data/signdata.md) — sign and encrypt data objects
- [`updateidentity`](../../reference/identity/updateidentity.md) — write content to an identity
