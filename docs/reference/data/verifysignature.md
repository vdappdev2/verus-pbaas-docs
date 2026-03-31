# verifysignature

> Verify a signature produced by `signdata`.

**Category:** Identity
**Daemon help:** `verus help verifysignature`

---

## Summary

Verify that a data signature is valid for a given identity or address. By default, validates against the identity's keys at the block height stored in the signature. Use `checklatest: true` to verify against the identity's current keys instead.

---

## Syntax

```
verifysignature '{"address":"id@", "message":"text", "signature":"base64sig", ...}'
```

The input is a single JSON object with the fields below.

---

## Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `address` | string | Yes | VerusID name or R-address to verify against |
| `signature` | string | Yes | Base64-encoded signature from [`signdata`](signdata.md) output |
| `checklatest` | boolean | No | If true, verify against the identity's current keys. Default: false (verify against keys at the signing height stored in the signature). |

### Data input (use the same mode used for signing)

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | Plain text message that was signed |
| `messagehex` | string | Hex-encoded data that was signed |
| `filename` | string | File path that was signed. Requires `-enablefileencryption`. |
| `datahash` | string | Pre-computed 256-bit hex hash that was signed |
| `messagebase64` | string | Base64-encoded data that was signed (may not work — see `signdata`) |

### Hash and binding (must match what was used during signing)

| Field | Type | Description |
|-------|------|-------------|
| `hashtype` | string | Hash algorithm used during signing: `"sha256"` (default), `"sha256D"`, `"blake2b"`, `"keccak256"`. Must match the algorithm passed to `signdata`. |
| `vdxfkeys` | array | VDXF key i-addresses bound during signing |
| `vdxfkeynames` | array | VDXF key names bound during signing |
| `boundhashes` | array | Hex hashes bound during signing |
| `prefixstring` | string | Prefix string used during signing |

> All binding and hash parameters must match exactly. If `signdata` was called with `vdxfkeynames`, `verifysignature` must include the same `vdxfkeynames`. Omitting them produces a different hash and verification fails.

---

## Return value

```json
{
  "signaturestatus": "verified",
  "system": "VRSCTEST",
  "systemid": "iJhCezBExJHvtyH3fGhNnt2NhU4Ztkf2yq",
  "identity": "test1.mcp3@",
  "canonicalname": "test1.mcp3@",
  "address": "iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT",
  "hashtype": "sha256",
  "hash": "5d4948bdc1a6882c5af8ec144815c03708b18acd2e3d281f0f6a6e54d90fa2e2",
  "height": 990651,
  "signatureheight": 990651,
  "signature": "AgW7HQ8..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `signaturestatus` | string | `"verified"` if valid |
| `hash` | string | Hash of the data as computed by the verifier |
| `height` | number | Current block height at verification time |
| `signatureheight` | number | Block height when the signature was created |
| `identity` | string | Identity that signed the data |
| `hashtype` | string | Hash algorithm used |
| `signature` | string | The signature that was verified |

When binding parameters were used, `vdxfkeynames` and/or `boundhashes` are echoed back.

> Output structure confirmed on vrsctest, 2026-03-24.

---

## Important behaviors

- **Height-aware verification.** By default, the signature is verified against the identity's keys at the block height stored in the signature. A signature remains valid even after the identity's keys change — it proves who controlled the identity at that time.
- **`checklatest: true`** verifies against the identity's current keys instead. Use this when you need to confirm the signer still controls the identity now, not just that they did when they signed.
- **`hashtype` is not auto-detected.** The verifier must be told which algorithm was used. If you signed with `blake2b`, you must pass `"hashtype": "blake2b"` to verify. The default is `sha256`.
- **All binding parameters must match.** `vdxfkeynames`, `vdxfkeys`, `boundhashes`, and `prefixstring` are mixed into the hash. Omitting any that were present during signing produces a hash mismatch.

---

## Examples

### Verify a message signature

```
verifysignature '{"address":"test1.mcp3@", "message":"hash algorithm test", "signature":"AgW7HQ8..."}'
```

### Verify with a non-default hash algorithm

```
verifysignature '{"address":"test1.mcp3@", "message":"hash algorithm test", "hashtype":"blake2b", "signature":"AgG7HQ8..."}'
```

### Verify a file signature

```
verifysignature '{"address":"test1.mcp3@", "filename":"/tmp/verus-test-file.txt", "signature":"AgW7HQ8..."}'
```

### Verify with VDXF key binding

```
verifysignature '{"address":"test1.mcp3@", "message":"attested data", "vdxfkeynames":["vrsc::data.type.string"], "signature":"AgW+HQ8..."}'
```

### Verify with bound hashes

```
verifysignature '{"address":"test1.mcp3@", "message":"bound hash test", "boundhashes":["5d4948bd..."], "signature":"AgW+HQ8..."}'
```

---

## See also

- [`signdata`](signdata.md) — create signatures to verify
- [How to Sign and Verify Data](../../how-to/data/sign-and-verify-data.md) — step-by-step guide
