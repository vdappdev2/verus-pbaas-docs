# How to Grant Read Access to Encrypted Data

Share decryption keys to allow specific parties to read encrypted data — either all data at a z-address, or a single encrypted object. Verus provides two key types for this, each with different scope.

**Prerequisites:**
- Encrypted data (stored via `sendcurrency:data` to a z-address, or via `signdata` + `updateidentity` on an identity)
- The z-address used for encryption

---

## Choose the access scope

| Scope | Key type | What it unlocks | How to get it |
|---|---|---|---|
| **All data** at a z-address | Extended viewing key (EVK) | Every object encrypted to that z-address — past and future | `z_exportviewingkey "zs1..."` |
| **One specific object** | Specific symmetric key (SSK) | Only the object it was generated for | Returned by `signdata` as `signaturedata_ssk` |

Use EVK when the recipient should see everything at the address. Use SSK when they should see only specific items.

---

## Option A: Grant full read access with an EVK

### Step 1: Export the viewing key

```
z_exportviewingkey "zs1..."
```

Returns:

```
"zxviews1q..."
```

The EVK grants read-only access — no spending authority. The recipient can decrypt all data encrypted to this z-address but cannot spend funds.

### Step 2: Share the EVK with the recipient

Transmit the EVK through a secure channel. Anyone who holds the EVK can read all data at the address.

### Step 3: Recipient decrypts

The recipient uses the EVK in two ways:

**Pass explicitly to `decryptdata`:**

```
decryptdata '{
  "datadescriptor": { ... },
  "txid": "abc123...",
  "retrieve": true,
  "evk": "zxviews1q..."
}'
```

**Or import for persistent access:**

```
z_importviewingkey "zxviews1q..." "whenkeyisnew" 990000
```

After importing, the recipient's node indexes data at that z-address. The `startHeight` parameter (e.g., `990000`) avoids scanning the entire chain — set it to a block height before the first data was stored.

> [!NOTE]
> Whether importing the EVK enables `decryptdata` without passing `evk` explicitly has not been tested (requires a second wallet that does not hold the spending key). Passing the EVK explicitly to `decryptdata` always works.

---

## Option B: Grant per-object access with an SSK

### Step 1: Obtain the SSK

When you encrypt data with `signdata` + `encrypttoaddress`, the output includes `signaturedata_ssk`:

```
signdata '{
  "address": "myid@",
  "message": "confidential data",
  "encrypttoaddress": "zs1..."
}'
```

Response (excerpt):

```json
{
  "signaturedata_ssk": "d4e2f8a1...",
  "mmrdescriptor_encrypted": {
    "datadescriptors": [{
      "version": 1,
      "flags": 5,
      "objectdata": "a3f7c9e1...",
      "epk": "02b4d8f3..."
    }]
  }
}
```

Save the SSK and the encrypted DataDescriptor.

### Step 2: Share the SSK and encrypted DataDescriptor

The recipient needs both:
- The encrypted DataDescriptor (`flags: 5`, `objectdata`, `epk`)
- The SSK

### Step 3: Recipient decrypts

The recipient includes the SSK inside the DataDescriptor:

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

Returns the decrypted content:

```json
[{
  "version": 1,
  "flags": 2,
  "objectdata": "636f6e666964656e7469616c2064617461",
  "salt": "a7b3c9d2..."
}]
```

Decode the hex `objectdata` to recover the original data.

---

## Selective disclosure pattern

Encrypt multiple objects to the same z-address using separate `signdata` calls. Each call produces its own SSK. Share different SSKs with different parties:

```
Party A  →  SSK₁  →  can read object 1 only
Party B  →  SSK₂  →  can read object 2 only
Party C  →  EVK   →  can read everything
```

> [!IMPORTANT]
> The SSK is scoped to the entire `signdata` call, not to individual MMR leaves. If you encrypt three items in one `signdata` call with `createmmr: true`, all three share the same SSK. For per-item selective disclosure, make separate `signdata` calls.

---

## Key types compared

| | EVK | SSK |
|---|---|---|
| **Scope** | All data at the z-address | One `signdata` call's output |
| **Access duration** | Permanent (past and future data) | One-time (specific object only) |
| **Revocability** | Cannot be revoked — data encrypted to that z-address is readable forever | Inherently limited — only decrypts one object |
| **Source** | `z_exportviewingkey` | `signdata` with `encrypttoaddress` |
| **Spending authority** | None — read-only | None — read-only |
| **Works for z-address data** | Yes — pass as `evk` to `decryptdata` with `retrieve: true` | Not applicable (z-address data has no SSK — it's encrypted automatically by `sendcurrency`) |
| **Works for identity content** | Yes (if encrypted to the same z-address) | Yes — the primary mechanism |

---

## Common pitfalls

**Always pass the EVK for z-address data.** When decrypting on-chain z-address data (via `datadescriptor` + `txid` + `retrieve: true`), the EVK must be passed explicitly. Without it, `decryptdata` returns still-encrypted data (`flags: 5`) even if the wallet holds the spending key.

**SSK does not exist for `sendcurrency:data`.** Data sent to z-addresses via `sendcurrency` is encrypted automatically — no `signdata` call occurs, so no SSK is generated. Use the EVK to share read access to z-address data.

**EVK cannot be revoked.** Once shared, the recipient can read all data encrypted to that z-address, including data stored after the EVK was shared. Use SSK-based selective disclosure when you need fine-grained control.

**Use the original encrypted DataDescriptor.** For identity content, the daemon modifies `flags` when storing (5 becomes 37). Always use the DataDescriptor from the original `signdata` output, not the on-chain version. See [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md).

---

## See also

- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — the encryption model and three access levels
- [How to Store and Retrieve Private Data](store-and-retrieve-private-data.md) — z-address storage and retrieval
- [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md) — the `signdata` → `updateidentity` flow
- [`signdata`](../../reference/data/signdata.md) — `encrypttoaddress` and SSK output
- [`decryptdata`](../../reference/data/decryptdata.md) — `evk`, `ivk`, and `ssk` parameters
- [`z_exportviewingkey`](../../reference/data/z_exportviewingkey.md) — export an EVK
- [`z_importviewingkey`](../../reference/data/z_importviewingkey.md) — import an EVK for persistent access
