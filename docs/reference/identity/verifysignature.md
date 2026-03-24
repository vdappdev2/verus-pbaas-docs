# verifysignature

> Verify a signature produced by signdata.

**Category:** Identity
**Daemon help:** `verus help verifysignature`

---

## Summary

Verify a data signature against the signing identity's keys. By default, verifies against the identity's keys at the signing height. Use `checklatest: true` to verify against the identity's current keys.

---

## Syntax

```
verifysignature '{"address":"id@", "message":"text", "signature":"...", ...}'
```

---

## Parameters

The input is a single JSON object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `address` | string | Yes | Identity or address to verify against |
| `message` | string | Yes* | The original message that was signed (*or use `messagehex`, `filename`, `hash`, etc.) |
| `messagehex` | string | Yes* | Hex-encoded message (*alternative to `message`) |
| `signature` | string | Yes | The signature from [`signdata`](signdata.md) output |
| `checklatest` | boolean | No | Verify against current keys instead of keys at signing height |

---

## Return value

```json
{
  "signaturestatus": "verified"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `signaturestatus` | string | `"verified"` if valid, error details if not |

---

## Important behaviors

- **Hash mismatch warning:** The daemon hashes `message` differently in `verifysignature` than in `signdata`, which can produce different hashes for the same string. If verification fails with a direct `message` parameter, try converting to `messagehex` first.
- **Height-aware verification:** By default, the signature is verified against the identity's keys at the block height when it was signed. This means a signature remains valid even after the identity's keys are changed — it proves who controlled the identity at that time.

---

## Example

```
verifysignature '{"address":"alice@","message":"hello world","signature":"AgXMCg..."}'
```

---

## See also

- [`signdata`](signdata.md) — create signatures to verify
