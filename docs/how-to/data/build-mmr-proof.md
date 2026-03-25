# How to Build an MMR Proof

Build a Merkle Mountain Range (MMR) over multiple data items, sign the root with a VerusID, and share individual leaves with their proofs. This enables batched attestations — sign once, prove any item individually.

**Prerequisites:**
- A VerusID you control
- Familiarity with [`signdata`](../../reference/data/signdata.md)

---

## Step 1: Prepare the data items

Collect the data items that should be committed together. Each item becomes a leaf in the MMR. Items can use any input mode: `message`, `messagehex`, `filename`, etc.

```json
[
  {"message": "credential: Alice passed exam A"},
  {"message": "credential: Bob passed exam B"},
  {"message": "credential: Carol passed exam C"}
]
```

---

## Step 2: Build and sign the MMR

Call `signdata` with `createmmr: true` and the items in `mmrdata`:

```
signdata '{
  "address": "myid@",
  "createmmr": true,
  "mmrdata": [
    {"message": "credential: Alice passed exam A"},
    {"message": "credential: Bob passed exam B"},
    {"message": "credential: Carol passed exam C"}
  ]
}'
```

The daemon constructs the MMR, salts each leaf, and signs the root hash. The output includes:

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
        "objectdata": "63726564656e7469616c3a20416c696365207061737365642065...",
        "salt": "86fd3b8f..."
      },
      {
        "version": 1,
        "flags": 2,
        "objectdata": "63726564656e7469616c3a20426f62207061737365642065...",
        "salt": "a2c4e7f1..."
      },
      {
        "version": 1,
        "flags": 2,
        "objectdata": "63726564656e7469616c3a204361726f6c207061737365642065...",
        "salt": "d3b5f8a9..."
      }
    ]
  },
  "signaturedata": { "...": "..." },
  "signature": "AgW7HQ8..."
}
```

Key fields:
- **`mmrroot.objectdata`** — the root hash (the single value the signature covers)
- **`mmrhashes.objectdata`** — the full hash tree (needed for proof construction)
- **`datadescriptors`** — per-leaf content and salt (the leaves themselves)
- **`signature`** — signs the root hash with the VerusID

---

## Step 3: Save the outputs

Save the full `signdata` output. To share or verify individual leaves, you need:

| What | Where | Purpose |
|---|---|---|
| Root hash | `mmrdescriptor.mmrroot.objectdata` | The commitment — what the signature covers |
| Hash tree | `mmrdescriptor.mmrhashes.objectdata` | For constructing proof paths from leaf to root |
| Leaf content | `mmrdescriptor.datadescriptors[N].objectdata` | The actual data (hex-encoded) |
| Leaf salt | `mmrdescriptor.datadescriptors[N].salt` | Mixed into the leaf hash |
| Signature | `signaturedata` / `signature` | Proves the root was signed by the identity |

---

## Step 4: Share a specific leaf

To prove that "Alice passed exam A" is in the signed set:

1. Share the leaf's `objectdata` and `salt` (from `datadescriptors[0]`)
2. Share the proof path — the sibling hashes from the leaf to the root (derived from `mmrhashes`)
3. Share the `mmrroot` and `signaturedata`

The verifier can:
1. Hash the leaf content with its salt
2. Walk the proof path to recompute the root
3. Verify the root matches the signed root
4. Verify the signature against the signing identity with `verifysignature`

> [!NOTE]
> The proof-path extraction from `mmrhashes` and single-leaf verification via `verifysignature` have not been independently tested on vrsctest. The MMR construction and root signing are confirmed. If your application needs leaf-level proofs, test the verification path on testnet first.

---

## Optional: Encrypt the MMR

Add `encrypttoaddress` to encrypt each leaf:

```
signdata '{
  "address": "myid@",
  "createmmr": true,
  "encrypttoaddress": "zs1...",
  "mmrdata": [
    {"message": "credential: Alice passed exam A"},
    {"message": "credential: Bob passed exam B"},
    {"message": "credential: Carol passed exam C"}
  ]
}'
```

This adds `mmrdescriptor_encrypted` (per-leaf ciphertext) and `signaturedata_ssk` (symmetric decryption key) to the output.

The SSK is per-call, not per-leaf — all leaves share it. For per-leaf selective disclosure, make separate `signdata` calls. See [How to Grant Read Access to Encrypted Data](grant-read-access.md).

---

## Optional: Use explicit salts

For deterministic leaf hashes, provide salts via `mmrsalt`:

```
signdata '{
  "address": "myid@",
  "createmmr": true,
  "mmrdata": [{"message": "leaf 1"}, {"message": "leaf 2"}],
  "mmrsalt": [
    "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
    "f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3b2a1f6e5"
  ]
}'
```

> [!WARNING]
> Each salt must be a 64-character hex string. Non-hex values crash the daemon.

---

## See also

- [Merkle Mountain Ranges on Verus](../../concepts/merkle-mountain-ranges.md) — how MMRs work and their properties
- [`signdata`](../../reference/data/signdata.md) — full parameter reference for `createmmr`, `mmrdata`, `mmrsalt`
- [How to Grant Read Access to Encrypted Data](grant-read-access.md) — sharing EVKs and SSKs for encrypted MMR leaves
- [How to Sign and Verify Data](sign-and-verify-data.md) — basic signing without MMRs
