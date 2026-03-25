# On-Chain Data Storage and Encryption

Verus provides two distinct paths for storing arbitrary data on-chain. Each path has different visibility, encryption, and retrieval semantics. Choosing the right path depends on whether the data should be private or public, and how consumers will access it.

---

## Two storage paths

| | Z-address data | Identity content |
|---|---|---|
| **Store via** | `sendcurrency` with `data` parameter | `updateidentity` with `contentmultimap` |
| **Retrieve via** | `z_listreceivedbyaddress` then `decryptdata` | `getidentity`, `getidentitycontent`, `getidentityhistory` |
| **Visibility** | Private — encrypted to the destination z-address | Public — readable by anyone on-chain |
| **Encryption** | Automatic (destination z-address encrypts) | Manual (encrypt with `signdata` first, then store ciphertext) |
| **Data format** | `signdata` object format | VDXF-keyed entries (simple typed values or DataDescriptors) |
| **Key requirement** | Destination must be a z-address or `"ID@:private"` | Identity must be controlled by the caller |

Both paths store data immutably on-chain. Once written, data cannot be deleted — only superseded by newer content (in the identity path) or left in place (in the z-address path).

---

## Path 1: Private data via z-addresses

Data sent via `sendcurrency` with the `data` parameter is encrypted to the destination Sapling z-address. Only holders of the appropriate decryption key can read it.

### How it works

1. The caller constructs a data object (message, hex, base64, or file) and sends it to a z-address with `amount: 0`
2. The daemon encrypts the data to the destination z-address and broadcasts the transaction
3. The recipient lists received data with `z_listreceivedbyaddress`, which returns a memo containing a **data descriptor** — a metadata wrapper that references the encrypted payload
4. The recipient decrypts with `decryptdata`, passing the data descriptor and a decryption key

### Requirements

- The destination address **must** be a z-address (`zs1...`) or `"ID@:private"` (which resolves to the ID's assigned Sapling address). Transparent and identity addresses are rejected with "Cannot use data parameter unless sending to a private address."
- The source address (`fromaddress`) does **not** need to be a z-address — transparent addresses and VerusID names work as the funding source.
- No value transfer is required (`amount: 0` is typical).
- Fee scales with data size. Observed: 0.00354 VRSCTEST for a 1587-byte transaction containing a short message.
- File storage requires the `-enablefileencryption` flag in the daemon config. String, hex, and base64 data work without this flag.

> All requirements confirmed on vrsctest, 2026-03-22.

### The data descriptor

When data is retrieved via `z_listreceivedbyaddress`, the memo field contains a **data descriptor** — a structured object that tells `decryptdata` how to locate and decrypt the payload.

The descriptor uses two VDXF object types:

| VDXF key | i-address | Role |
|---|---|---|
| `vrsc::data.type.object.datadescriptor` | `i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv` | Outer wrapper with version and flags |
| `vrsc::data.type.object.crosschaindataref` | `iP3euVSzNcXUrLNHnQnR9G6q8jeYuGSxgw` | References the transaction containing the data |

A `crosschaindataref` with an all-zero txid and `voutnum: 0` means the data is in the same transaction as the descriptor itself. This is the typical pattern for data stored via `sendcurrency`.

### Decrypted output

`decryptdata` returns the payload as hex in the `objectdata` field with `flags: 2` (indicating decrypted data). The caller must decode the hex to recover the original content. A `salt` field is present — the daemon auto-generates it during encryption.

---

## Path 2: Public data on identities

Data stored in a VerusID's `contentmultimap` is publicly readable by anyone. This is the primary mechanism for structured application data on-chain — profiles, attestations, timestamps, and application state.

For the full content format and VDXF key system, see [VDXF and Identity Content](vdxf-and-identity-content.md).

### Encrypting identity content

Content in `contentmultimap` is public by default. To store encrypted data on a public identity, use a two-step process:

1. **Encrypt with `signdata`:** Call `signdata` with the data and `encrypttoaddress` (a Sapling z-address). This returns an encrypted DataDescriptor containing ciphertext, an encryption public key (`epk`), and a specific symmetric key (`ssk`) for this object.

2. **Store via `updateidentity`:** Write the encrypted DataDescriptor into `contentmultimap`. The daemon accepts it and stores the ciphertext. On-chain, the data is visible but unreadable without the decryption key.

3. **Decrypt with `decryptdata`:** Pass the **original** encrypted DataDescriptor from step 1 to `decryptdata`. The wallet auto-decrypts if it holds the z-address keys.

> **Caveat:** The daemon modifies the DataDescriptor's `flags` field when storing it on an identity (observed: 5 becomes 37). This modification breaks the `decryptdata` shortcut that queries identity content directly (`iddata` parameter). Applications must cache the original encrypted DataDescriptor from step 1 for reliable decryption, or reconstruct the original descriptor from the on-chain version.

> **Gotcha:** `encrypttoaddress` is a **processing instruction** for `signdata` and `sendcurrency:data`. Placing it directly in a contentmultimap DataDescriptor via `updateidentity` does nothing — it is silently ignored and content is stored as plaintext.

> Both caveats confirmed on vrsctest, 2026-03-23.

---

## The data object format

Both storage paths use the same data object format, defined by `signdata`. The format supports multiple input modes:

| Field | Type | Description |
|---|---|---|
| `message` | string | Plain text message |
| `filename` | string | Local file path (requires `-enablefileencryption` daemon flag) |
| `hex` | string | Raw hex data |
| `base64` | string | Base64-encoded data |
| `datahash` | string | Hash-only reference (stores the hash, not the data) |
| `vdxfdata` | object | VDXF-encoded structured data |

Only one input mode should be used per data object.

### Signing fields

| Field | Type | Description |
|---|---|---|
| `address` | string | Identity or t-address to sign with |
| `hashtype` | string | Hash algorithm: `"sha256"` (default), `"sha256D"`, `"blake2b"`, `"keccak256"` |

### Encryption fields

| Field | Type | Description |
|---|---|---|
| `encrypttoaddress` | string | Sapling z-address to encrypt to |

### MMR fields (Merkle Mountain Range)

| Field | Type | Description |
|---|---|---|
| `createmmr` | boolean | Build an MMR over the data objects |
| `mmrdata` | array | Array of data objects to include in the MMR |
| `mmrsalt` | boolean | Salt leaf nodes for privacy |
| `priormmr` | string | Reference to a prior MMR to extend |

> [!NOTE]
> `priormmr` is listed in the daemon help text but marked as unimplemented. MMR fields have not been tested end-to-end.

---

## Encryption and access control

Verus provides three levels of decryption access, enabling fine-grained control over who can read encrypted data:

| Access level | Key type | Scope | How to obtain |
|---|---|---|---|
| **Full control** | Wallet spending key | All data at the z-address | Wallet holds the z-address |
| **Read-only** | Extended viewing key (EVK) | All data at the z-address | `z_exportviewingkey "zs1..."` |
| **Per-object** | Specific symmetric key (SSK) | One encrypted object only | Returned by `signdata` as `signaturedata_ssk` |

### Spending key (auto-decrypt)

If the wallet holds the z-address spending key, `decryptdata` auto-decrypts with no additional parameters. This is the simplest path — the wallet recognizes the ciphertext and decrypts transparently.

> Confirmed on vrsctest, 2026-03-23.

### Extended viewing key (EVK)

The EVK grants read access to all data encrypted to a z-address without granting spending authority. Export it with `z_exportviewingkey` and pass it as the `evk` parameter to `decryptdata`. Share it to grant read access to a third party.

> Confirmed on vrsctest, 2026-03-22.

### Specific symmetric key (SSK)

The SSK decrypts only the specific object it was generated for. `signdata` returns it as `signaturedata_ssk` in its output. This enables **selective disclosure** — share different SSKs with different parties to grant access to specific objects without exposing everything at the z-address.

> Confirmed on vrsctest, 2026-03-23.

### Incoming viewing key (IVK)

A hex-format viewing key passed as the `ivk` parameter to `decryptdata`. Functions similarly to the EVK.

> [!NOTE]
> IVK decryption has not been independently tested. Behavior inferred from `decryptdata` parameter documentation.

---

## Signing without storage

`signdata` signs data off-chain using the same object format as `sendcurrency:data`. It does not store anything on the blockchain. The signed output includes a structured `signaturedata` object that can be passed to `verifysignature` for verification.

Use cases:
- **Attestations:** Sign a statement with a VerusID, share the signature for others to verify
- **Pre-encryption:** Encrypt data with `encrypttoaddress` before storing via `updateidentity`
- **Verification workflows:** Prove that a specific identity signed specific data at a specific block height

> `signdata` standalone operation and `signdata` → `verifysignature` round-trip both confirmed on vrsctest, 2026-03-22.

---

## Comparison of storage paths

| Consideration | Z-address data | Identity content |
|---|---|---|
| **Privacy** | Private by default | Public by default (encryption optional) |
| **Discovery** | Only the z-address holder knows it exists | Anyone can query the identity |
| **Retrieval complexity** | Two-step: list + decrypt | One-step: `getidentity` or `getidentitycontent` |
| **Data organization** | Flat (by z-address) | Structured (by VDXF key) |
| **Content updates** | Append-only (new tx per write) | Revision-based (each `updateidentity` is a revision) |
| **Deletion** | Not possible | Logical delete via `vrsc::identity.multimapremove` |
| **Cost** | Fee scales with data size | Fee for the `updateidentity` transaction |
| **Read access sharing** | Share EVK or SSK | Data is public; for encrypted content, share SSK |

---

## Known limitations

- **`decryptdata` with `iddata` does not work for encrypted identity content.** The daemon's flags modification (5 → 37) when storing DataDescriptors in `contentmultimap` breaks the one-step query-and-decrypt path. Applications must cache the original encrypted DataDescriptor from `signdata` output. (Confirmed 2026-03-23.)

- **File storage requires `-enablefileencryption`.** The `filename` input mode in the data object reads files from the local filesystem, which is disabled by default. Add `-enablefileencryption` to the daemon config to enable it. String, hex, and base64 modes work without this flag. (File storage not yet tested end-to-end.)

- **MMR features are untested.** The `createmmr`, `mmrdata`, and `mmrsalt` fields are documented in `verus help signdata` but have not been verified with live testing. `priormmr` is explicitly marked as unimplemented in the help text.

- **Max data size is constrained by block size** (2-4 MB) but the practical limit per transaction has not been tested.

---

## See also

- [VDXF and Identity Content](vdxf-and-identity-content.md) — the VDXF key system and contentmultimap format
- [How to Store and Retrieve Private Data](../how-to/data/store-and-retrieve-private-data.md) — step-by-step guide for z-address data
- [How to Encrypt Data on a Public Identity](../how-to/data/encrypt-data-on-public-identity.md) — step-by-step guide for encrypted identity content
- [How to Sign and Verify Data](../how-to/data/sign-and-verify-data.md) — signing workflow without storage
- [`sendcurrency`](../reference/multichain/sendcurrency.md) — the `data` parameter for z-address storage
- [`signdata`](../reference/data/signdata.md) — sign, encrypt, and structure data objects
- [`decryptdata`](../reference/data/signdata.md) — decrypt on-chain data
