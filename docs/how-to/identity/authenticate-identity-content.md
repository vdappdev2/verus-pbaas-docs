# How to Authenticate Identity Content

Verify that a `contentmultimap` entry was authored or endorsed by a specific namespace authority — not just by whoever owns the identity it sits on. Use this pattern when consumers must reject entries that lack a valid signature from the expected signer.

Builds on three primitives: [`signdata`](../../reference/data/signdata.md) (with VDXF key binding), [`updateidentity`](../../reference/identity/updateidentity.md), and [`verifysignature`](../../reference/data/verifysignature.md).

**Prerequisites:**
- A namespace authority identity (its primary key in the wallet) — the entity whose endorsement consumers will check
- A target identity where the content will be published — its primary key needed for writing, not for later verification
- Familiarity with VDXF keys and [authority over vdxfkey writes](../../concepts/vdxf-and-identity-content.md#authority-over-vdxfkey-writes)

---

## Pattern overview

1. The namespace authority signs content with `signdata`, binding the signature to **both** the namespace vdxfkey and the target identity's i-address.
2. The writer publishes `{content, proof}` together on the target identity under the namespace-controlled vdxfkey.
3. The consumer reads the entry, calls `verifysignature` with the embedded proof, and accepts only on `signaturestatus: "verified"` AND a target-binding match.

---

## Step 1: Sign with VDXF + target-identity binding

The namespace authority signs the content, binding the signature to the namespace vdxfkey and to the target identity's i-address:

```
signdata '{
  "address": "mcp3@",
  "message": "endorsed: test1.mcp3@ achievement v1",
  "vdxfkeys": [
    "iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W",
    "iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT"
  ]
}'
```

- `address` — the namespace authority. Its primary key must be in the wallet.
- First vdxfkey — the namespace URI's i-address (`mcp3.vrsctest::test1.endorsement` → `iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W` via [`getvdxfid`](../../reference/identity/getvdxfid.md)).
- Second vdxfkey — the target identity's i-address (`test1.mcp3@` → `iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT`). This is the **replay defense** — see [below](#replay-defense-why-the-target-identity-is-bound).

Relevant output fields:

```json
{
  "signature": "AgWfJxAAAUEf2R4JuQ0GqtlssoPxz7J1B6XDhIHESz/0IuZtVbRAjrwdE/h898Sxy2aUhFutC7lz5bFcjQhC+BXGveJ/msnDXA==",
  "signatureheight": 1058719,
  "vdxfkeys": [
    "iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W",
    "iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT"
  ]
}
```

Capture `signature` and `vdxfkeys`. `signdata` does not touch chain or wallet state.

> Verus uses deterministic ECDSA: the same key + same message + same bindings + same `signatureheight` produces byte-identical signatures. Different `signatureheight` values produce different signature bytes for the same logical signing — all valid signatures by the same key. Confirmed on vrsctest, 2026-05-12.

---

## Step 2: Publish content + proof together

Bundle the content and proof into a single JSON object and store it on the target identity's `contentmultimap` under the namespace vdxfkey. Use `mimetype: application/json` so `objectdata` is rendered as opaque hex rather than free text (see [Display shape depends on mimetype](../../concepts/vdxf-and-identity-content.md#display-shape-depends-on-mimetype)).

```
updateidentity '{
  "name": "test1",
  "parent": "i77n5FCqSBkXAK3UWHpdrPpdtXRc8sqjoz",
  "contentmultimap": {
    "iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W": [
      {
        "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
          "version": 1,
          "mimetype": "application/json",
          "objectdata": {
            "message": "{\"content\":\"endorsed: test1.mcp3@ achievement v1\",\"proof\":{\"signer\":\"mcp3@\",\"vdxfkeys\":[\"iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W\",\"iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT\"],\"signature\":\"AgWfJxAAAUEf2R4JuQ0GqtlssoPxz7J1B6XDhIHESz/0IuZtVbRAjrwdE/h898Sxy2aUhFutC7lz5bFcjQhC+BXGveJ/msnDXA==\"}}"
          },
          "label": "iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W"
        }
      }
    ]
  }
}'
```

Notes on the write:

- Pass the bundle as `objectdata: {"message": "<minified JSON string>"}`. The daemon canonicalizes it to a hex string on storage.
- The writer needs authority over the target identity. The namespace authority's primary key is not needed for the write — only for the prior signing in Step 1.
- Always merge with existing `contentmultimap` entries when present — omitting the field clears all content.

After one confirmation, `getidentity "test1.mcp3@"` shows:

```json
"objectdata": "7b22636f6e74656e74223a22656e646f727365643a..."
```

> Round-trip confirmed on vrsctest, 2026-05-12 (txid `6649979fa3de0414da69927e8d2c0d012441a8fc2c1ae401a7b91636fb123776`): `{"message": "<json string>"}` write → `objectdata: "<hex>"` read, hex decodes back to the exact original JSON bundle.

---

## Step 3: Consumer reads and verifies

The consumer fetches the entry, decodes the hex, extracts content and proof, and runs two checks: cryptographic verification and target-binding match.

```python
import json, subprocess

ident = json.loads(subprocess.check_output([
    "verus", "-chain=vrsctest", "getidentity", "test1.mcp3@"
]))
entries = ident["identity"]["contentmultimap"]["iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W"]
payload = json.loads(bytes.fromhex(
    entries[0]["i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv"]["objectdata"]
).decode())

result = json.loads(subprocess.check_output([
    "verus", "-chain=vrsctest", "verifysignature", json.dumps({
        "address": payload["proof"]["signer"],
        "message": payload["content"],
        "vdxfkeys": payload["proof"]["vdxfkeys"],
        "signature": payload["proof"]["signature"]
    })
]))

# Check 1: signature is cryptographically valid
if result["signaturestatus"] != "verified":
    raise ValueError("entry not authenticated by expected signer")

# Check 2: signature is bound to THIS identity (replay defense)
if payload["proof"]["vdxfkeys"][1] != ident["identity"]["identityaddress"]:
    raise ValueError("signature is for a different target identity")

print(payload["content"])  # safe to use
```

Both checks together close the replay window: the cryptographic statement is sound, and it was made for this specific identity.

> Verified on vrsctest, 2026-05-12: `signaturestatus: "verified"`, second `vdxfkeys` element (`iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT`) matches `test1.mcp3@`'s `identityaddress`.

---

## Replay defense: why the target identity is bound

A signature is just bytes — it is not located anywhere. Without binding to the target identity, a valid `(content, signature)` pair signed for placement on identity A can be copied into identity B's cmm under the same vdxfkey, and `verifysignature` still returns `verified` — because the cryptographic claim (`signer signed this content under this vdxfkey at this height`) is unchanged.

Adding the target identity's i-address as an extra `vdxfkeys` element makes the signature only valid when verified with that same i-address binding. A consumer compares the second `vdxfkeys` element to the identity it is reading and rejects on mismatch.

> Confirmed on vrsctest, 2026-05-12: a signature bound to `[iMapKxgVLT3geSE9pDBMaK8ZaupRKNob7W, iNkRqYXKFaBYg9pNHVpgS7EXHFtAzMMwvT]` returned `signaturestatus: "invalid"` when verified with the second vdxfkey replaced by an unrelated i-address; same signer, same message, same first binding.

---

## Common pitfalls

**Forgetting the target binding.** A signature bound only to the namespace vdxfkey is portable — any identity owner can copy and replant the `(content, signature)` pair. Always include the target identity's i-address as one of the `vdxfkeys` (or a `boundhash`) at signing time, and check it at read time.

**Trusting only `signaturestatus`.** `verifysignature` returns `verified` whenever the signature matches its bound inputs. It does **not** confirm that those inputs are the ones the consumer expects. The consumer must independently confirm that the returned `vdxfkeys` are the correct namespace key and the correct target identity.

**Signing the wrong layer.** `signdata` signs the `message` (or hash) you pass in — not the on-chain DataDescriptor wrapper. The daemon may mutate storage flags (e.g., `flags: 5 → 37` in manual encryption); your signature must cover content that is recoverable independent of any wrapper mutation.

**Re-signing produces different bytes.** Each `signdata` call re-hashes with the current `signatureheight`, so signature bytes change between calls for identical inputs. This is expected — all are valid signatures by the same key. Pin the `signatureheight` from the original output if you need byte-stability (e.g., for test fixtures).

---

## See also

- [`signdata`](../../reference/data/signdata.md) — full parameter reference including VDXF key binding
- [`verifysignature`](../../reference/data/verifysignature.md) — verification semantics
- [VDXF and Identity Content — Authority over vdxfkey writes](../../concepts/vdxf-and-identity-content.md#authority-over-vdxfkey-writes) — why the chain doesn't enforce namespace ownership
- [How to Sign and Verify Data](../data/sign-and-verify-data.md) — base `signdata` → `verifysignature` round-trip without storage
- [How to Store and Read Identity Content](store-and-read-content.md) — base patterns for writing cmm entries
