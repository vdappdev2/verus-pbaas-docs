# Merkle Mountain Ranges on Verus

A Merkle Mountain Range (MMR) is an append-only hash tree that lets you prove specific data items belong to a signed set without revealing the other items. Verus exposes MMR construction through `signdata`, enabling off-chain proofs, batched attestations, and selective disclosure of data leaves.

---

## What an MMR is

An MMR is a collection of perfect binary trees (peaks) that together cover all leaves. Each leaf is hashed, and pairs of hashes are combined into parent nodes up to the peak. The MMR root is a hash over all peaks, providing a single commitment to the entire dataset.

Key properties:

- **Append-only:** New leaves are added without recomputing the entire tree.
- **Compact proofs:** Proving a specific leaf is in the tree requires only a logarithmic number of hashes (the path from leaf to root), not the full dataset.
- **Selective disclosure:** Share a single leaf and its proof path — the verifier can confirm inclusion without seeing any other leaves.

---

## How Verus builds MMRs

Call `signdata` with `createmmr: true` and an array of data items in `mmrdata`. The daemon constructs the MMR, signs the root, and returns the full structure.

```
signdata '{
  "address": "myid@",
  "createmmr": true,
  "mmrdata": [
    {"message": "leaf 1"},
    {"message": "leaf 2"},
    {"message": "leaf 3"}
  ]
}'
```

### Output structure

| Field | Description |
|---|---|
| `mmrdescriptor.mmrroot` | DataDescriptor containing the MMR root hash — the single commitment to all leaves |
| `mmrdescriptor.mmrhashes` | Full hash tree as a single hex blob |
| `mmrdescriptor.datadescriptors` | Array of per-leaf DataDescriptors, each with content (`objectdata`) and salt |
| `mmrdescriptor.objecthashtype` | Hash algorithm for leaf data (default: `5` = sha256) |
| `mmrdescriptor.mmrhashtype` | Hash algorithm for the tree structure (default: `1` = blake2b) |

The signature signs the MMR root, not individual leaves. This means one signature covers all leaves — and individual leaves can be proven against the root without additional signatures.

---

## Salting

Each leaf is automatically salted with a random value. The salt is included in the leaf's DataDescriptor (`salt` field) and mixed into the leaf hash before tree construction.

Salting prevents brute-force discovery of leaf contents from their hashes. Without salting, an attacker who knows a leaf's content could compute its hash and confirm its presence in the tree.

### Explicit salts

Override auto-generated salts with the `mmrsalt` parameter — an array of 64-character hex strings (256-bit values), one per leaf:

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

Explicit salts are useful when you need deterministic leaf hashes (e.g., for cross-system verification).

> [!WARNING]
> `mmrsalt` values **must** be 64-character hex strings. Passing arbitrary strings (e.g., `"customsalt1"`) crashes the daemon (assertion failure in `uint256.cpp`, daemon v2000753). This is a daemon bug — the daemon should validate and return an error.

---

## Encryption with MMRs

Combine `createmmr: true` with `encrypttoaddress` to build an encrypted MMR. The daemon encrypts each leaf and returns both encrypted and plaintext versions, plus an SSK for decryption.

```
signdata '{
  "address": "myid@",
  "createmmr": true,
  "encrypttoaddress": "zs1...",
  "mmrdata": [
    {"message": "item 1"},
    {"message": "item 2"},
    {"message": "item 3"}
  ]
}'
```

The output includes:
- `mmrdescriptor_encrypted.datadescriptors` — per-leaf encrypted DataDescriptors (`flags: 5`, ciphertext, `epk`)
- `mmrdescriptor.datadescriptors` — per-leaf plaintext DataDescriptors (`flags: 2`, original content, `salt`)
- `signaturedata_ssk` — a symmetric key for decryption

> [!IMPORTANT]
> The SSK is **per `signdata` call**, not per MMR leaf. All leaves in one call share the same SSK. For per-leaf selective disclosure, make separate `signdata` calls — each produces its own SSK.

When decrypting, pass individual `datadescriptors` entries to `decryptdata` — not the full `mmrdescriptor_encrypted` object (which returns empty output).

---

## Verification

The signature covers the MMR root hash. This means:

- **Collective proof:** The root signature proves that all leaves, in their specific positions, existed when the signer signed.
- **Individual leaf proof:** A verifier with the root hash, a specific leaf, and the proof path (sibling hashes from leaf to root) can confirm that the leaf is part of the signed set.

> [!NOTE]
> Leaf-level verification (proving a single leaf against the root via `verifysignature`) has not been independently tested. The signature and MMR construction are confirmed working — the verification path against individual leaves is inferred from the MMR structure but not yet exercised on vrsctest.

---

## Known limitations

- **`priormmr` is unimplemented.** Listed in `verus help signdata` but not functional. You cannot extend an existing MMR with new leaves — each `signdata` call builds a standalone MMR.

- **`mmrsalt` crash bug.** Non-hex strings in `mmrsalt` crash the daemon. Always use 64-character hex values. (Daemon v2000753.)

- **SSK granularity.** One SSK per `signdata` call, not per leaf. Use separate calls for per-item selective disclosure.

- **Full encrypted MMR descriptor not decryptable as a unit.** `decryptdata` must receive individual `datadescriptors` entries, not the whole `mmrdescriptor_encrypted`.

---

## See also

- [How to Build an MMR Proof](../how-to/data/build-mmr-proof.md) — step-by-step guide
- [`signdata`](../reference/data/signdata.md) — the `createmmr`, `mmrdata`, and `mmrsalt` parameters
- [On-Chain Data Storage and Encryption](on-chain-data-storage-and-encryption.md) — the encryption model and access control levels
- [How to Grant Read Access to Encrypted Data](../how-to/data/grant-read-access.md) — sharing EVKs and SSKs
