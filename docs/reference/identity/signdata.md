# signdata

> Sign data with a VerusID or address, without storing it on-chain.

**Category:** Identity
**Daemon help:** `verus help signdata`

---

## Summary

Sign arbitrary data using a VerusID or transparent address. The data is signed locally and the result is returned — nothing is stored on the blockchain. Use [`verifysignature`](verifysignature.md) to verify the result.

The data object format is shared with `sendcurrency`'s `data` parameter. `signdata` is for cases where you want to sign without storing, or prepare data (including encryption) for later use.

---

## Syntax

```
signdata '{"address":"id@", "message":"text", ...}'
```

---

## Parameters

The input is a single JSON object with the following fields:

### Input modes (use one)

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | Plain text message |
| `filename` | string | Path to local file. **Requires `-enablefileencryption`** daemon flag. |
| `hex` | string | Raw hex data |
| `base64` | string | Base64-encoded data |
| `datahash` | string | Hash-only reference (signs the hash, not the data) |
| `vdxfdata` | object | VDXF-encoded structured data |

### Signing fields

| Field | Type | Description |
|-------|------|-------------|
| `address` | string | Identity or t-address to sign with |
| `hashtype` | string | Hash algorithm: `"sha256"` (default), `"sha256D"`, `"blake2b"`, `"keccak256"` |
| `signature` | string | Existing signature to accumulate (for multi-sig) |

### Encryption fields

| Field | Type | Description |
|-------|------|-------------|
| `encrypttoaddress` | string | Sapling z-address to encrypt to. Returns both plaintext and encrypted versions. The encrypted DataDescriptor can be stored in `contentmultimap` via [`updateidentity`](updateidentity.md). |

### MMR fields

> These fields are documented from the daemon interface but have not yet been verified on testnet.

| Field | Type | Description |
|-------|------|-------------|
| `createmmr` | boolean | Build a Merkle Mountain Range over the data objects |
| `mmrdata` | array | Array of data objects to include in the MMR |
| `mmrsalt` | boolean | Salt leaf nodes for privacy |

### VDXF binding fields

| Field | Type | Description |
|-------|------|-------------|
| `vdxfkeys` | array | VDXF keys to bind to the signed data |
| `vdxfkeynames` | array | VDXF key names (resolved via [`getvdxfid`](getvdxfid.md)) |
| `boundhashes` | array | Hashes to bind into the signature |

---

## Return value

Returns a signed data object containing:

| Field | Type | Description |
|-------|------|-------------|
| `signaturedata` | object | Structured signature for passing to [`verifysignature`](verifysignature.md) |
| `hash` | string | Hash of the signed data |
| `identity` | string | Signing identity i-address |
| `signatureheight` | number | Block height at signing time |

When `encrypttoaddress` is used, also returns:

| Field | Type | Description |
|-------|------|-------------|
| `mmrdescriptor_encrypted` | object | Encrypted DataDescriptor (`flags: 5`, ciphertext, `epk`) |
| `signaturedata_ssk` | string | Specific symmetric key for this object — enables selective disclosure |
| `mmrdescriptor` | object | Plaintext DataDescriptor (`flags: 2`, original data, `salt`) |

---

## Important behaviors

- **Does not store anything on-chain.** Purely local signing operation.
- **`encrypttoaddress` is a processing instruction** for `signdata` — it is NOT a DataDescriptor field. Putting it directly in a `contentmultimap` DataDescriptor via `updateidentity` does nothing (silently ignored).
- **Encrypted content workflow:** Call `signdata` with `encrypttoaddress` → store the encrypted DataDescriptor in `contentmultimap` → decrypt later with `decryptdata` using the original descriptor.
- **Default hash type:** sha256.

---

## Examples

### Sign a message

```
signdata '{"address":"alice@","message":"hello world"}'
```

### Sign and encrypt

```
signdata '{"address":"alice@","message":"secret data","encrypttoaddress":"zs1..."}'
```

---

## See also

- [`verifysignature`](verifysignature.md) — verify a signature from `signdata`
- [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md) — storing encrypted content on identities
