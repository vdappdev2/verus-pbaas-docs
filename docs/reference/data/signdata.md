# signdata

> Sign, hash, encrypt, or build MMRs over arbitrary data — without storing anything on-chain.

**Category:** Identity
**Daemon help:** `verus help signdata`

---

## Summary

Generate a hash and signature of data using a VerusID or transparent address. Supports multiple input formats, four hash algorithms, encryption to z-addresses, VDXF key binding, and Merkle Mountain Range construction. Nothing is stored on the blockchain — the signed output is returned for the caller to use.

The data object format is shared with `sendcurrency`'s `data` parameter. Use `signdata` when you want to sign without storing, encrypt data before storing on an identity via [`updateidentity`](../identity/updateidentity.md), or build MMR proofs over multiple data items.

---

## Syntax

```
signdata '{"address":"id@", "message":"text", ...}'
```

The input is a single JSON object with the fields below.

---

## Parameters

### Required

| Field | Type | Description |
|-------|------|-------------|
| `address` | string | VerusID name or transparent R-address to sign with. The identity's primary address must be in the wallet. A t-address produces a simple signature with no identity metadata. |

### Input modes (use one)

| Field | Type | Description | Status |
|-------|------|-------------|--------|
| `message` | string | Plain text message | Confirmed |
| `messagehex` | string | Hex-encoded data | Confirmed |
| `filename` | string | Path to local file. Requires `-enablefileencryption` daemon flag. | Confirmed |
| `datahash` | string | Pre-computed 256-bit hex hash. Signs the hash directly without hashing again — produces the same signature as signing the original data. | Confirmed |
| `messagebase64` | string | Base64-encoded data | Fails in daemon v2000753 ("Invalid parameter for data output") |
| `vdxfdata` | object | VDXF-encoded structured data | Not tested |

> Input mode status confirmed on vrsctest, 2026-03-24 with `test1.mcp3@`.

### Hash algorithm

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `hashtype` | string | `"sha256"` | Hash algorithm: `"sha256"`, `"sha256D"`, `"blake2b"`, `"keccak256"` |

Each algorithm produces a different hash and a different `hashtype` integer in the `signaturedata` output:

| String | Integer | Description |
|--------|---------|-------------|
| `sha256` | 5 | SHA-256 |
| `sha256D` | 4 | Double SHA-256 |
| `blake2b` | 1 | BLAKE2b |
| `keccak256` | 3 | Keccak-256 |

> All four algorithms confirmed with sign + verify round-trips on vrsctest, 2026-03-24.

### Encryption

| Field | Type | Description |
|-------|------|-------------|
| `encrypttoaddress` | string | Sapling z-address to encrypt to. Returns both encrypted and plaintext versions, plus a per-object SSK for selective disclosure. |

### VDXF binding

| Field | Type | Description |
|-------|------|-------------|
| `vdxfkeys` | array | VDXF key i-addresses to bind into the signature hash |
| `vdxfkeynames` | array | VDXF key names or friendly IDs (resolved by the daemon — no i-addresses) |
| `boundhashes` | array | 256-bit hex hashes to bind into the signature hash |
| `prefixstring` | string | Extra string hashed during signing — must be supplied for verification |

Binding mixes the keys/hashes into the signed hash, producing a different signature than signing the same data without binding. The same binding parameters must be passed to [`verifysignature`](verifysignature.md).

> `vdxfkeynames` and `boundhashes` binding confirmed with sign + verify round-trips on vrsctest, 2026-03-24. Note: `boundhashes` appear byte-reversed in `signaturedata.boundhashes` (little-endian) but in original order in the top-level output.

### MMR (Merkle Mountain Range)

| Field | Type | Description |
|-------|------|-------------|
| `createmmr` | boolean | If true (or if `mmrdata` has multiple items), build an MMR and sign the root |
| `mmrdata` | array | Array of data objects: `[{"message": "..."}, {"filename": "..."}, ...]` |
| `mmrsalt` | array | Array of 64-character hex strings (256-bit salts), one per leaf. Replaces auto-generated salts. |
| `mmrhashtype` | string | Hash algorithm for the MMR tree structure. Default: `"blake2b"`. Options: same as `hashtype`. |
| `priormmr` | array | Prior MMR hashes for extending an existing MMR. **UNIMPLEMENTED** per daemon help text. |

> Basic MMR creation and explicit hex salting confirmed on vrsctest, 2026-03-24. **Warning:** passing non-hex strings in `mmrsalt` crashes the daemon (assertion failure in `uint256.cpp`). Always use 64-character hex values.

### Multi-sig accumulation

| Field | Type | Description |
|-------|------|-------------|
| `signature` | string | Existing base64 signature to accumulate for multi-sig identities |

---

## Return value

### Standard signing

```json
{
  "signaturedata": {
    "version": 1,
    "systemid": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
    "hashtype": 5,
    "signaturehash": "5d4948bd...",
    "identityid": "iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT",
    "signaturetype": 1,
    "signature": "AgW7HQ8..."
  },
  "system": "VRSCTEST",
  "systemid": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
  "hashtype": "sha256",
  "hash": "5d4948bd...",
  "identity": "test1.mcp3@",
  "canonicalname": "test1.mcp3@",
  "address": "iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT",
  "signatureheight": 990651,
  "signatureversion": 2,
  "signature": "AgW7HQ8..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `signaturedata` | object | Structured signature object — pass to [`verifysignature`](verifysignature.md) |
| `hash` | string | Hash of the signed data |
| `hashtype` | string | Algorithm used |
| `identity` | string | Signing identity friendly name |
| `canonicalname` | string | Fully qualified name |
| `address` | string | Signing identity i-address |
| `signatureheight` | number | Block height at signing time — binds the signature to a point in time |
| `signatureversion` | number | Signature format version |
| `signature` | string | Base64-encoded signature (same as `signaturedata.signature`) |

When `vdxfkeynames` or `boundhashes` are used, those arrays are echoed back in the output.

### With `encrypttoaddress`

Additionally returns:

| Field | Type | Description |
|-------|------|-------------|
| `mmrdescriptor_encrypted.datadescriptors[0]` | object | Encrypted DataDescriptor: `flags: 5`, ciphertext in `objectdata`, `epk` |
| `signaturedata_ssk` | string | Per-object symmetric key for selective disclosure |
| `mmrdescriptor.datadescriptors[0]` | object | Plaintext DataDescriptor: `flags: 2`, original data in `objectdata`, `salt` |

> Encryption output confirmed on vrsctest, 2026-03-23.

### With `createmmr` / `mmrdata`

```json
{
  "mmrdescriptor": {
    "version": 1,
    "objecthashtype": 5,
    "mmrhashtype": 1,
    "mmrroot": {
      "version": 1,
      "flags": 0,
      "objectdata": "9a6b6a50..."
    },
    "mmrhashes": {
      "version": 1,
      "flags": 0,
      "objectdata": "41d826d3..."
    },
    "datadescriptors": [
      {
        "version": 1,
        "flags": 2,
        "objectdata": "6d6d72206c6561662031",
        "salt": "86fd3b8f..."
      }
    ]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `mmrdescriptor.mmrroot` | object | DataDescriptor containing the MMR root hash |
| `mmrdescriptor.mmrhashes` | object | Full hash tree as a single hex blob |
| `mmrdescriptor.datadescriptors` | array | Per-leaf DataDescriptors with content and salt |
| `mmrdescriptor.objecthashtype` | number | Hash algorithm for leaf data (5 = sha256) |
| `mmrdescriptor.mmrhashtype` | number | Hash algorithm for the tree (1 = blake2b) |

The signature signs the MMR root, not individual leaves. Each leaf is auto-salted unless explicit salts are provided via `mmrsalt`.

> MMR output confirmed on vrsctest, 2026-03-24 with 3 leaves. Explicit hex salting also confirmed.

---

## Important behaviors

- **Does not modify blockchain or wallet state.** Purely local operation.
- **`encrypttoaddress` is a processing instruction** — it is NOT a DataDescriptor field. Placing it directly in a `contentmultimap` DataDescriptor via `updateidentity` does nothing (silently ignored, content stored as plaintext). Confirmed on vrsctest, 2026-03-23.
- **`datahash` signs the raw hash.** It does not hash the input again. Passing the sha256 of "hello" produces the same signature as signing "hello" with `message`. This is useful for signing pre-computed hashes of large files without transmitting the file.
- **`mmrsalt` requires 64-character hex values.** Arbitrary strings cause a daemon crash (assertion failure in `uint256.cpp`, daemon v2000753). This is a daemon bug — the daemon should validate and return an error.
- **`messagebase64` does not work** in daemon v2000753. Use `messagehex` instead.

---

## Examples

### Sign a message

```
signdata '{"address":"test1.mcp3@", "message":"hash algorithm test"}'
```

### Sign with a specific hash algorithm

```
signdata '{"address":"test1.mcp3@", "message":"hash algorithm test", "hashtype":"blake2b"}'
```

### Sign a file

Requires `-enablefileencryption` in daemon config.

```
signdata '{"address":"test1.mcp3@", "filename":"/tmp/verus-test-file.txt"}'
```

### Sign and encrypt

```
signdata '{"address":"test1.mcp3@", "message":"secret data", "encrypttoaddress":"zs1..."}'
```

### Sign with VDXF key binding

```
signdata '{"address":"test1.mcp3@", "message":"attested data", "vdxfkeynames":["vrsc::data.type.string"]}'
```

### Build an MMR

```
signdata '{"address":"test1.mcp3@", "createmmr":true, "mmrdata":[{"message":"leaf 1"},{"message":"leaf 2"},{"message":"leaf 3"}]}'
```

---

## See also

- [`verifysignature`](verifysignature.md) — verify a signature from `signdata`
- [`sendcurrency`](../multichain/sendcurrency.md) — the `data` parameter uses the same object format for on-chain storage
- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — the two storage paths and encryption model
- [How to Sign and Verify Data](../../how-to/data/sign-and-verify-data.md) — step-by-step signing workflow
- [How to Encrypt Data on a Public Identity](../../how-to/data/encrypt-data-on-public-identity.md) — signdata → updateidentity → decryptdata flow
