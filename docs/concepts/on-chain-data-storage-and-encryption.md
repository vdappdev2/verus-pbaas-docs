# On-Chain Data Storage and Encryption

Verus provides two distinct paths for storing arbitrary data on-chain. Each path has different visibility, encryption, and retrieval semantics. Choosing the right path depends on whether the data should be private or public, and how consumers will access it.

---

## Two storage paths

| | Z-address data | Identity content |
|---|---|---|
| **Store via** | `sendcurrency` with `data` parameter | `updateidentity` with `contentmultimap` |
| **Retrieve via** | `z_listreceivedbyaddress` then `decryptdata` | `getidentity`, `getidentitycontent`, `getidentityhistory` |
| **Visibility** | Private — encrypted to the destination z-address | Public — readable by anyone on-chain |
| **Encryption** | Automatic — `sendcurrency` encrypts to the destination z-address (explicit `encrypttoaddress` rejected) | Two options: daemon-managed via the `{data: {...}}` envelope in `contentmultimap`, or manual via `signdata` + `encrypttoaddress` followed by `updateidentity` |
| **Data format** | Same input keys as `signdata` (`message`, `messagehex`, etc.) — but independent pipeline | VDXF-keyed entries (simple typed values or DataDescriptors) |
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
- File storage requires the `-enablefileencryption` daemon flag (startup flag or config file). String, hex, and base64 data work without this flag.

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

Content in `contentmultimap` is public by default. There are three encryption tiers for identity content, each producing a different on-chain DataDescriptor shape:

| Tier | How to write | On-chain | Who decrypts |
|---|---|---|---|
| **Plaintext** | Write the value directly under your VDXF key | `flags: 0` / `96` (no encryption) | Anyone, no decryption needed |
| **Public-encrypted (envelope)** | `{data: {<input>}}` in `contentmultimap` (no `encrypttoaddress`) | `flags: 13` with on-chain `ivk` | Anyone — IVK is published; content is encrypted-at-rest for opt-in viewing |
| **Private-encrypted (envelope or manual)** | Envelope: `{data: {<input>, "encrypttoaddress": "zs1..."}}`. Manual: `signdata --encrypttoaddress` then store the resulting flags:5 descriptor. | `flags: 5` with `epk` only (envelope), or `flags: 37` after manual storage flag mutation | Recipient z-address holder, or anyone with the EVK / IVK / SSK |

#### Public-encrypted via the `{data:{}}` envelope

Wrap any [`signdata`](../reference/data/signdata.md)-style input inside `data: {...}` directly in `contentmultimap`. The daemon:

1. Encrypts the payload to a **freshly generated ephemeral z-address**
2. **Discards the spending key** for that address
3. Publishes the IVK on-chain alongside the ciphertext

Because the recipient z-address is ephemeral and disposable, publishing the IVK is privacy-safe — no other notes will ever be sent to that address, and the address has no spending key alive. The IVK only ever decrypts the one published note.

This tier is suited to content that should be **encrypted at rest but readable on demand** — e.g., user-generated content where wallets shouldn't render text/images unconditionally, but any reader who chooses to can decrypt.

> Confirmed on vrsctest, 2026-05-06: two consecutive default-mode entries on the same identity produced different IVKs (proving fresh ephemeral recipients), and a 1.3 KB PNG round-tripped byte-perfect via the `filename` input.

#### Private-encrypted

Two construction paths produce recipient-targeted encrypted content:

- **Envelope with `encrypttoaddress`:** `{data: {<input>, "encrypttoaddress": "zs1..."}}` in `contentmultimap`. Daemon encrypts to your specified z-address and **withholds the IVK** (only the recipient's keys decrypt).
- **Manual `signdata` → `updateidentity`:** Call `signdata` with `encrypttoaddress`, capture the encrypted DataDescriptor (`flags: 5`) AND the per-object SSK, then store the descriptor via `updateidentity`. The daemon mutates `flags: 5 → 37` on storage. SSK enables per-object selective disclosure to multiple parties.

Use the manual path when you need the SSK for selective disclosure. Use the envelope when you need a single recipient and want one-call simplicity.

> **Caveat (manual path only):** the daemon's flag mutation (5 → 37) breaks the `decryptdata` shortcut via `iddata`. Cache the original encrypted DataDescriptor from `signdata` output for reliable decryption. The envelope path does not exhibit this mutation — flags:13 is preserved on-chain.

> **Gotcha:** `encrypttoaddress` placed directly inside a DataDescriptor (i.e., not inside a `data:` envelope) is silently ignored. The daemon only honors `encrypttoaddress` as a processing instruction within `signdata`, `sendcurrency:data`, or the `{data:{}}` envelope.

> Caveats confirmed on vrsctest by reading the on-chain `flags` field via `getidentity` after writing: manual-path entry showed `flags: 37` (mutated from the original `5`) on 2026-03-23; envelope-path entry showed `flags: 13` unchanged on 2026-05-06.

### Why publishing the IVK is privacy-safe in default envelope mode

The default envelope mode encrypts to a **freshly generated, single-use ephemeral Sapling z-address** for each call. The daemon discards the spending key immediately after encryption. Two consequences make publishing the IVK harmless:

- The address has no spending key alive — it can never participate in any other transaction
- The daemon does not reuse it — no other notes will ever be sent there

The published IVK therefore decrypts only the one note it accompanies. There is no broader privacy budget to compromise.

This is a different security shape from the **private-encrypted** mode (`encrypttoaddress` set). When you specify a recipient z-address, the daemon withholds the IVK precisely because that address may be reused for other shielded value. Reusing a wallet's main `id@:private` for one-off `encrypttoaddress` outputs is therefore risky if its IVK or EVK is later shared — every note on that address becomes readable.

> Verified on vrsctest, 2026-05-06: two consecutive default-mode entries on the same identity produced different on-chain IVKs, confirming a fresh ephemeral recipient per call.

---

## VDXF keys: structural use vs binding use

VDXF keys appear in two distinct roles. They are not mutually exclusive — many real workflows use both, layered.

### Role 1: structural keys in `contentmultimap`

The outer key under which an entry sits, and the optional `label` field on each entry, are both VDXF keys. This is how `cmm` is organized.

Storing under a VDXF key produces a chain-anchored fact: the controlling identity authorized an `updateidentity` at a given block height with content filed under a known semantic key. Anyone calling `getidentity` or `getidentitycontent` (optionally with a `vdxfkey` filter) can read it.

This role is always direct — `signdata` is not involved. Both plaintext entries and the `{data: {...}}` envelope use it.

### Role 2: binding parameters in a signed proof

`signdata` accepts `vdxfkeys`, `vdxfkeynames`, `boundhashes`, and `prefixstring`. These mix context into the signature hash so the resulting `signaturedata` is **bound to that context** — different binding produces a different signature for the same input.

The artifact is a portable signed proof: a verifier can check it offline (or in a different chain context) without trusting any particular chain query, as long as they know the same binding parameters.

### These compose

The two roles operate at different layers and stack cleanly: run `signdata` with whatever binding you want, then store the resulting encrypted DataDescriptor in `cmm` via the manual `updateidentity` path. The result is a chain-anchored entry under a known VDXF key AND a portable signed proof. See [How to Encrypt Data on a Public Identity](../how-to/data/encrypt-data-on-public-identity.md) for the manual workflow.

### What is actually constrained

The single hard constraint we have verified: **`signdata` binding parameters cannot be used inside the `{data: {...}}` envelope shorthand.** The envelope's encryption is signed with the funding wallet's pubkey rather than a VerusID, and the daemon rejects binding fields with: *"When signing with public key and not identity, cannot include vdxf keys, vdxf key names, bound hashes, or multisig."* `hashtype` and `prefixstring` are accepted by the envelope but silently ignored — they have no observable effect on the encrypted output.

So the choice is not "cmm or signdata." It is "envelope shorthand or manual `signdata` path." If a workflow needs binding, an SSK, or a VerusID-bound MMR root, run `signdata` first and store its output via the manual cmm path.

> Envelope rejection of `vdxfkeys` / `vdxfkeynames` / `boundhashes` / multisig confirmed on vrsctest, 2026-05-06. Silent ignoring of `hashtype` and `prefixstring` confirmed the same day by comparing decrypted plaintext across baseline / `hashtype: "blake2b"` / `prefixstring: "myprefix"` — all identical.

---

## Chain-anchored fact vs portable signed proof

These two artifacts answer different verification questions. Picking the right model is independent of whether VDXF keys are involved.

| Artifact | Produced by | What it proves | Verifier needs |
|---|---|---|---|
| **Chain-anchored fact** | `updateidentity` writing to `cmm` (envelope or plaintext) | A specific identity authorized this content at this block height under this VDXF key | Chain access — call `getidentity` / `getidentitycontent` |
| **Portable signed proof** | `signdata` with optional binding | A specific identity signed this content with this binding context, at this height | The signature blob plus the matching binding parameters; no chain query required |

Common patterns:

- **Chain-anchored fact only.** Public application data, identity profile fields, time-anchored proof of existence (storing a hash under a VDXF key at a height), per-application data namespaces. The chain itself is the authority — `signdata` adds overhead without adding value.
- **Portable signed proof only.** Off-chain attestations passed between systems that cannot or do not query the chain. Signed messages embedded in other artifacts. `signdata` produces the proof; storage is optional.
- **Both, layered.** Run `signdata` (with binding if needed), then store the encrypted DataDescriptor in `cmm`. Anyone can find the entry by querying the identity, AND any holder of the signed blob can verify it offline. This is the typical pattern for portable attestations that also need a chain-discoverable record.

Reach for `signdata` specifically when a workflow needs:

- A `signaturedata` blob verifiable independently of chain access
- Domain-separated signatures (same input must produce different signatures in different contexts)
- External-content binding (`boundhashes`)
- Per-object SSK selective disclosure
- Multi-sig signature accumulation
- A VerusID-signed MMR root

For everything else, direct `cmm` storage is sufficient.

---

## The data input format

Both `sendcurrency:data` and `signdata` accept the same input keys for specifying data content:

| Field | Type | Description |
|---|---|---|
| `message` | string | Plain text message |
| `filename` | string | Local file path (requires `-enablefileencryption` daemon flag) |
| `messagehex` | string | Raw hex data |
| `messagebase64` | string | Base64-encoded data (broken in daemon v2000753 — use `messagehex`) |
| `datahash` | string | Hash-only reference (stores the hash, not the data) |
| `vdxfdata` | string or object | VDXF-encoded structured data. Object form performs VDXF binary serialization; string form is equivalent to `message`. |

Only one input mode should be used per data object.

### Two independent pipelines

Although `sendcurrency:data` and `signdata` share the same input vocabulary, **they are independent pipelines** — the output of `signdata` is not passed to `sendcurrency`.

| | `sendcurrency:data` | `signdata` |
|---|---|---|
| **Purpose** | Store data on-chain at a z-address | Sign, encrypt, or build MMRs off-chain |
| **Encryption** | Automatic — always encrypts to the destination z-address | Manual — use `encrypttoaddress` |
| **`encrypttoaddress`** | **Rejected** — encryption is implicit | Accepted — specifies which z-address to encrypt to |
| **On-chain effect** | Broadcasts a transaction | None — returns result for caller to use |
| **Typical next step** | Retrieve with `z_listreceivedbyaddress` + `decryptdata` | Pass encrypted output to `updateidentity`, or verify with `verifysignature` |

> Confirmed on vrsctest, 2026-03-24. Passing `signdata` output as `sendcurrency:data` returns "Must include one and only one of filename, message, messagehex, messagebase64, and datahash." Passing `encrypttoaddress` inside `sendcurrency:data` returns "Data output may only be sent to a z-address, is always encrypted, and may not have an explicit encrypttoaddress option."

### signdata-only fields

These fields are accepted by `signdata` but not by `sendcurrency:data`:

| Field | Type | Description |
|---|---|---|
| `address` | string | Identity or R-address to sign with |
| `hashtype` | string | Hash algorithm: `"sha256"` (default), `"sha256D"`, `"blake2b"`, `"keccak256"` |
| `encrypttoaddress` | string | Sapling z-address to encrypt to |
| `createmmr` | boolean | Build an MMR over the data objects |
| `mmrdata` | array | Array of data objects to include in the MMR |
| `mmrsalt` | array | Array of 64-char hex salts, one per MMR leaf |
| `priormmr` | string | Reference to a prior MMR to extend |

> [!NOTE]
> `priormmr` is listed in the daemon help text but is unimplemented. `createmmr`, `mmrdata`, and `mmrsalt` (with 64-char hex values) are confirmed working. See [`signdata`](../reference/data/signdata.md) for details.

---

## Encryption and access control

Verus provides three levels of decryption access, enabling fine-grained control over who can read encrypted data:

| Access level | Key type | Scope | How to obtain |
|---|---|---|---|
| **Full control** | Wallet spending key | All data at the z-address | Wallet holds the z-address |
| **Read-only** | Extended viewing key (EVK) | All data at the z-address | `z_exportviewingkey "zs1..."` |
| **Per-object** | Specific symmetric key (SSK) | One encrypted object only | Returned by `signdata` as `signaturedata_ssk` |

### Spending key (auto-decrypt)

If the wallet holds the z-address spending key, `decryptdata` can auto-decrypt for **encrypted identity content** (DataDescriptors with `flags: 5` and `epk`).

> **Caveat — two different behaviors:**
> - **On-chain data** (via `datadescriptor` + `txid` + `retrieve: true`): EVK **must** be passed explicitly — without it, the daemon returns the data still encrypted (`flags: 5`), even on the owning node.
> - **Direct `signdata` encrypted output** (never stored on-chain): the wallet spending key auto-decrypts — EVK/SSK not strictly required on the owning node.
>
> Always pass the EVK for on-chain z-address data. For `signdata` output shared with third parties, provide the EVK or SSK.

> Auto-decrypt confirmed for identity content on vrsctest, 2026-03-23. EVK requirement for on-chain z-address data confirmed 2026-03-24. Wallet auto-decrypt for direct signdata output confirmed 2026-03-24.

### Extended viewing key (EVK)

The EVK grants read access to all data encrypted to a z-address without granting spending authority. Export it with `z_exportviewingkey` and pass it as the `evk` parameter to `decryptdata`. Share it to grant read access to a third party.

> Confirmed on vrsctest, 2026-03-22.

### Specific symmetric key (SSK)

The SSK decrypts only the specific object it was generated for. `signdata` returns it as `signaturedata_ssk` in its output. This enables **selective disclosure** — share different SSKs with different parties to grant access to specific objects without exposing everything at the z-address.

> [!IMPORTANT]
> SSK granularity is **per `signdata` call**, not per MMR leaf. If you sign three items in one `signdata` call with `createmmr: true` and `encrypttoaddress`, all three leaves share the same SSK. For per-item selective disclosure, make separate `signdata` calls for each item.

> Confirmed on vrsctest, 2026-03-23. MMR SSK scope confirmed 2026-03-24.

### Incoming viewing key (IVK)

A hex-format viewing key passed as the `ivk` parameter to `decryptdata`. The IVK reveals all incoming notes to every diversified address derived from the same Sapling account — same scope as the EVK for incoming reads, minus outgoing visibility and child-key derivation.

The IVK is the key the daemon **publishes on-chain** for envelope-mode entries (flags:13). Because the recipient z-address in that path is freshly generated and immediately abandoned, the published IVK only decrypts the one note it accompanies — no broader privacy is at stake. Pull the descriptor (with embedded `ivk`) from `getidentity` and pass it directly to `decryptdata` with the storing transaction's txid + `retrieve: true`.

For manually-encrypted content (flags:5 path), the IVK is **not** published. It must be obtained out-of-band — e.g., derived locally from the EVK, or shared by the recipient.

> Embedded-IVK decryption confirmed on vrsctest, 2026-05-06, with both `message` and `filename` inputs.

---

## Signing without storage

`signdata` signs data off-chain. It accepts the same input keys as `sendcurrency:data` but does not store anything on the blockchain. The signed output includes a structured `signaturedata` object that can be passed to `verifysignature` for verification.

Use cases:
- **Attestations:** Sign a statement with a VerusID, share the signature for others to verify
- **Pre-encryption for identity content:** Encrypt data with `encrypttoaddress` before storing via `updateidentity` (the only way to get encrypted data on a public identity — `sendcurrency:data` encrypts to z-addresses automatically and does not use `signdata`)
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

- **File storage requires `-enablefileencryption`.** The `filename` input mode reads files from the local filesystem, which is disabled by default. Pass `-enablefileencryption` at daemon startup for occasional use, or add it to the daemon config file for persistent access. String, hex, and base64 modes work without this flag. Full round-trip confirmed on vrsctest, 2026-03-24.

- **MMR salting crash bug.** `mmrsalt` values must be 64-character hex strings (256-bit hashes). Passing arbitrary strings (e.g., `"customsalt1"`) crashes the daemon with an assertion failure in `uint256.cpp`. The daemon should validate input and return an error. (Daemon v2000753.)

- **`priormmr` is unimplemented.** Listed in `verus help signdata` but not functional. MMR creation and explicit salting are confirmed working.

- **Max data size is 1,000,000 bytes (1 MB) per transaction.** The daemon enforces this limit on the raw data before encryption. Larger payloads are rejected with an explicit error. (Confirmed 2026-03-24: 898 KB PNG succeeded, 1,037,191-byte PNG rejected.)

- **One output per z-address per transaction.** `sendcurrency` rejects any transaction that includes two or more outputs to the same z-address ("Cannot duplicate private address source or destination"). This applies to all z-address sends — data, value, or both. Multiple data payloads can be sent in one transaction if each targets a different z-address. (Confirmed 2026-03-24.)

- **Contentmultimap DataDescriptor wrapping is strict.** Encrypted DataDescriptors in `contentmultimap` (for both `registeridentity` and `updateidentity`) must use `i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv` as the inner VDXF key. Using a different inner key silently strips the content to an empty string — the transaction succeeds but the data is lost. Full round-trip confirmed: `signdata` → encrypted DataDescriptor → `registeridentity` → `decryptdata` with EVK → original plaintext. (Confirmed 2026-03-25.)

---

## See also

- [VDXF and Identity Content](vdxf-and-identity-content.md) — the VDXF key system and contentmultimap format
- [How to Store and Retrieve Private Data](../how-to/data/store-and-retrieve-private-data.md) — step-by-step guide for z-address data
- [How to Publish Encrypted Data on an Identity](../how-to/data/publish-encrypted-data-on-identity.md) — the daemon-managed `{data:{}}` envelope path
- [How to Encrypt Data on a Public Identity](../how-to/data/encrypt-data-on-public-identity.md) — manual signdata-based path with SSK selective disclosure
- [How to Sign and Verify Data](../how-to/data/sign-and-verify-data.md) — signing workflow without storage
- [How to Grant Read Access to Encrypted Data](../how-to/data/grant-read-access.md) — sharing EVKs and SSKs
- [Merkle Mountain Ranges on Verus](merkle-mountain-ranges.md) — MMR construction and proofs via `signdata`
- [`sendcurrency`](../reference/multichain/sendcurrency.md) — the `data` parameter for z-address storage
- [`signdata`](../reference/data/signdata.md) — sign, encrypt, and structure data objects
- [`decryptdata`](../reference/data/decryptdata.md) — decrypt on-chain data
