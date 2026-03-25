# How to Sign and Verify Data

Sign arbitrary data with a VerusID and verify the signature. This workflow does not store anything on-chain — it produces a portable signature object that any party can verify independently.

**Prerequisites:**
- A VerusID you control (its primary address must be in the wallet)

---

## Step 1: Sign data with `signdata`

Call `signdata` with the data to sign and the identity to sign with:

```
signdata '{
  "address": "myid@",
  "message": "I attest that this document is authentic"
}'
```

The `address` field specifies which VerusID signs the data. The identity's primary address must be in the local wallet.

### Input modes

`signdata` accepts multiple input formats. Use whichever matches your data:

| Field | Use |
|---|---|
| `message` | Plain text string |
| `hex` | Raw hex-encoded data |
| `base64` | Base64-encoded data |
| `datahash` | Hash-only (signs the hash without including the data) |

> [!NOTE]
> `filename` and `vdxfdata` input modes are documented in `verus help signdata` but have not been tested.

### Output

`signdata` returns a structured object containing the signature, hash, and identity information:

```json
{
  "hash": "a1b2c3d4...",
  "signaturedata": {
    ...
  },
  "identity": "myid@",
  "canonicalname": "myid.vrsctest@",
  "address": "iABC123...",
  "signatureheight": 12345
}
```

Key fields:

- `signaturedata` — the structured signature object. Pass this to `verifysignature` along with the original data.
- `signatureheight` — the block height at which the signature was created. This binds the signature to a specific point in time and a specific state of the identity.
- `hash` — the hash of the signed data (default algorithm: sha256).

> `signdata` does not modify chain or wallet state. It is a read-only operation. Confirmed on vrsctest, 2026-03-22.

---

## Step 2: Verify the signature with `verifysignature`

Pass the original data and the `signaturedata` from Step 1 to `verifysignature`:

```
verifysignature '{
  "address": "myid@",
  "message": "I attest that this document is authentic",
  "signaturedata": { ... }
}'
```

- `address` — the identity that allegedly signed the data
- `message` — the original data (must match exactly what was signed)
- `signaturedata` — the signature object from `signdata` output

### Verified result

```json
{
  "signaturestatus": "verified"
}
```

If the data, identity, or signature do not match, the status will indicate the failure.

> `signdata` → `verifysignature` round-trip confirmed on vrsctest, 2026-03-22. The `message` parameter worked directly for both sign and verify.

---

## Hash algorithms

By default, `signdata` uses `sha256`. Specify a different algorithm with the `hashtype` field:

```
signdata '{
  "address": "myid@",
  "message": "data to sign",
  "hashtype": "blake2b"
}'
```

Supported algorithms:

| Value | Algorithm |
|---|---|
| `sha256` | SHA-256 (default) |
| `sha256D` | Double SHA-256 |
| `blake2b` | BLAKE2b |
| `keccak256` | Keccak-256 |

> [!NOTE]
> All four hash algorithms are documented in `verus help signdata`. Individual testing of each algorithm is pending.

---

## Use cases

**Attestations.** Sign a statement (hash, document reference, or assertion) with a VerusID. Share the signature with relying parties who can verify it against the identity on-chain.

**Data integrity.** Sign a file hash before distribution. Recipients verify the hash against the signature to confirm authenticity and that the file has not been modified since signing.

**Timestamped signatures.** The `signatureheight` in the output binds the signature to a specific block height. This proves the signature was created no earlier than that block and that the signing identity was active and controlled by the signer at that height.

**Pre-encryption.** Combine signing with `encrypttoaddress` to produce encrypted, signed data for storage on a public identity. See [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md).

---

## See also

- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — the broader data framework including signing
- [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md) — combining signing with encryption
- [How to Build an MMR Proof](build-mmr-proof.md) — batched attestations with Merkle Mountain Ranges
- [How to Grant Read Access to Encrypted Data](grant-read-access.md) — sharing EVKs and SSKs
- [`signdata`](../../reference/data/signdata.md) — full parameter reference
- [`verifysignature`](../../reference/data/verifysignature.md) — full parameter reference
