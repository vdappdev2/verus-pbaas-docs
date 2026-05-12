# How to Publish Encrypted Data on an Identity from a Wallet App

Store encrypted data in a VerusID's `contentmultimap` from a wallet-integrated app — one that builds an `IdentityUpdateRequest` and delegates signing to Verus Mobile via deeplink. The on-chain result is the same flags:13 public-encrypted entry produced by the daemon's [`{data:{}}` envelope](publish-encrypted-data-on-identity.md), but the SDK's cmm shape and submission flow are different.

The SDK detects the `data` envelope in cmm and routes it through `PartialSignData` instead of the standard cmm encoder. After wallet signing and submission, the daemon stores a flags:13 DataDescriptor with `epk` + `ivk` published — readable by anyone via [`decryptdata`](../../reference/data/decryptdata.md).

**Prerequisites:**
- A VerusID you can authorize signing for via Verus Mobile
- A wallet-integration scaffold that builds `IdentityUpdateRequest` and sends it as a `GenericRequest` deeplink (out of scope for this how-to)
- Verus Mobile v1.1.0-1 or later, with the experimental deeplinks toggle enabled

---

## Step 1: Build the cmm envelope

The cmm value must be a **single object** containing a `data` key — not an array. The `data` object accepts any [`PartialSignData` input field](../../reference/data/signdata.md#input-modes-use-one); for app-defined JSON payloads use `messagehex`:

```ts
const payload = { sha256: "...", title: "...", description: "..." };
const messagehex = Buffer.from(JSON.stringify(payload), "utf-8").toString("hex");

const contentmultimap = {
  "myid.vrsc::myapp.contentkey": {           // SINGLE OBJECT (not array)
    data: { messagehex }
  }
};
```

The outer key uses the **FQN form** (`<id>.<chain>::<vdxf-name>`) for wallet writes — the wallet's identity-update validator rejects custom i-address outer keys.

> **Type-cast caveat.** The `ContentMultiMap` type in `verus-typescript-primitives` declares cmm values as `DataDescriptorWrapper[]`. The single-object envelope needs a cast (`as unknown as DataDescriptorWrapper[]`) or a locally-widened type.

---

## Step 2: Build and send the request

Pass the cmm into `IdentityUpdateRequestDetails.fromCLIJson` alongside the identity name and parent. The SDK shortcut at `IdentityUpdateRequestDetails.ts:354-372` lifts the `data` envelope out of cmm and into the request's `signDataMap`:

```ts
import { IdentityUpdateRequestDetails } from "verus-typescript-primitives";

const details = IdentityUpdateRequestDetails.fromCLIJson(
  { name: "myid", parent: "iJhCez...", contentmultimap },
  { requestid: "<your-request-id>" }
);
```

Sign the details as a `GenericRequest` and send to the wallet via deeplink. The user approves in Verus Mobile; the wallet submits to the daemon.

---

## Step 3: Inspect what was stored

After the tx confirms:

```
getidentity "myid@"
```

```json
{
  "<outer-key-iaddr>": [
    {
      "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
        "version": 1,
        "flags": 13,
        "objectdata": "<ciphertext hex>",
        "epk": "<ephemeral public key>",
        "ivk": "<incoming viewing key, published>"
      }
    }
  ]
}
```

`flags: 13` with both `epk` and `ivk` confirms the entry took the public-encrypted path. The daemon normalizes the outer key to its i-address form on storage.

> Confirmed on vrsctest (`dust.broom@`) and VRSC mainnet (`next.bitcoins@`), 2026-05-11.

---

## Step 4: Decrypt

Decryption is identical to the daemon-RPC path: pass the on-chain DataDescriptor (with its embedded `ivk`) + the storing transaction's txid + `retrieve: true` to `decryptdata`. See [How to Publish Encrypted Data on an Identity → Step 4](publish-encrypted-data-on-identity.md#step-4-decrypt-the-entry) for the full call shape.

`decryptdata` returns `objectdata` as hex utf-8 bytes of whatever string was originally written. For a `messagehex`-of-JSON payload like Step 1, the round-trip is `Buffer.from(result[0].objectdata, "hex").toString("utf8")` → `JSON.parse`.

> **Public RPC requirement.** `decryptdata` must be whitelisted on the RPC endpoint. Public VRSC: `rpc.vrsc.syncproof.net` has it; `api.verus.services` does not. There is no public VRSCTEST endpoint with `decryptdata` whitelisted — testnet reads require a local vrsctest daemon.

---

## Common pitfalls

**`Unknown vdxfkey: [object Object]`** at request-build time. You wrapped the envelope in an array. The shortcut at `IdentityUpdateRequestDetails.ts:358` tests `cmm[key]['data']` for truthiness — that test fails for arrays, so the value falls through to `VdxfUniValue.fromJson`, which throws because `data` is not a registered VDXF data-type key. Fix: pass `{ outer_key: { data: {...} } }`, not `{ outer_key: [{ data: {...} }] }`.

**Wallet rejects the request without a clear error.** Older Verus Mobile builds do not handle the deeplink envelope path. Confirm the user is on **v1.1.0-1 or later** with the **experimental deeplinks toggle enabled**.

**flags:13 without `encrypttoaddress` is expected here.** A literal read of [`signdata`](../../reference/data/signdata.md)'s reference might suggest that signdata-style processing without `encrypttoaddress` produces flags:2 signed plaintext. Empirically, the wallet/daemon promotes a `{ messagehex }` payload routed through this shortcut to **flags:13 with `epk` + `ivk` published** — the same shape as the daemon's `{data:{}}` envelope. The reader's standard public-encrypted branch (bit 1 + bit 8) handles it without changes.

**Always include existing `contentmultimap` entries.** Omitting `contentmultimap` from `IdentityUpdateRequestDetails.fromCLIJson` clears all visible content. Read with `getidentity` first if you need to preserve prior entries.

---

## See also

- [How to Publish Encrypted Data on an Identity](publish-encrypted-data-on-identity.md) — the same on-chain shape via the daemon's `{data:{}}` envelope (direct RPC, no wallet)
- [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md) — the manual `signdata` → `updateidentity` path for SSK-based selective disclosure
- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — the underlying model and why publishing the IVK is privacy-safe
- [`signdata`](../../reference/data/signdata.md) — input vocabulary mirrored by `PartialSignData`
- [`updateidentity`](../../reference/identity/updateidentity.md) — the underlying RPC the wallet ultimately calls
- [`decryptdata`](../../reference/data/decryptdata.md) — decryption with `retrieve: true` + txid
