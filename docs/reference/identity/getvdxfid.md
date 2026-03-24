# getvdxfid

> Resolve a VDXF URI to its i-address.

**Category:** Vdxf
**Daemon help:** `verus help getvdxfid`

---

## Summary

Convert a VDXF URI string (e.g., `"vrsc::data.type.string"`) to its deterministic i-address. VDXF keys are the addressing system for structured on-chain data — they tell applications how to interpret data stored in identity content, signed data, and other VDXF-aware contexts.

---

## Syntax

```
getvdxfid "vdxfuri" ("vdxfkey") ("uint256") (indexnum)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `vdxfuri` | string | Yes | — | VDXF URI (e.g., `"vrsc::data.type.string"`, `"vtimestamp.vrsc::proof.basic"`) |
| 2 | `vdxfkey` | string | No | — | Combine with another VDXF key |
| 3 | `uint256` | string | No | — | Combine with a 256-bit hex hash |
| 4 | `indexnum` | number | No | — | Combine with an integer index |

---

## Return value

```json
{
  "vdxfid": "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c",
  "hash160result": "hexstring",
  "qualifiedname": {
    "namespace": "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV",
    "name": "vrsc::data.type.string"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `vdxfid` | string | The i-address for this VDXF key — use this in `contentmultimap` and filters |
| `indexid` | string | Index-form address |
| `hash160result` | string | Raw hash |
| `qualifiedname` | object | Namespace and name components |

---

## Common VDXF keys

### Primitive types (`vrsc::data.type.*`)

| URI | i-address |
|-----|-----------|
| `vrsc::data.type.string` | `iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c` |
| `vrsc::data.type.byte` | `iBXUHbh4iacbeZnzDRxishvBSrYk2S2k7t` |
| `vrsc::data.type.int32` | `iHpLPprRDv3H5H3ZMaJ9nyHFzkG9xJWZDb` |
| `vrsc::data.type.uint256` | `i8k7g7z6grtGYrNZmZr5TQ872aHssXuuua` |
| `vrsc::data.type.bytevector` | `iKMhRLX1JHQihVZx2t2pAWW2uzmK6AzwW3` |

### Object types (`vrsc::data.type.object.*`)

| URI | i-address |
|-----|-----------|
| `vrsc::data.type.object.datadescriptor` | `i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv` |
| `vrsc::data.type.object.crosschaindataref` | `iP3euVSzNcXUrLNHnQnR9G6q8jeYuGSxgw` |
| `vrsc::data.type.object.url` | `iJ7xdhJTJAvJubNnSJFXyA3jujzqGxjLuZ` |

### Identity/content keys

| URI | i-address |
|-----|-----------|
| `vrsc::identity.multimapremove` | `i5Zkx5Z7tEfh42xtKfwbJ5LgEWE9rEgpFY` |
| `vrsc::identity.profile.media` | `iEYsp2njSt1M4EVYi9uuAPBU2wpKmThkkr` |

### Custom keys

Any namespace can define keys: `vtimestamp.vrsc::proof.basic`, `mcp3.vrsctest::test.vdxfformat`, etc.

---

## Examples

### System-defined key

```
getvdxfid "vrsc::data.type.string"
```

```json
{
  "vdxfid": "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c",
  "indexid": "xPwgY6oPdusaAgNK3u5yHCQG5NsHEcBpi5",
  "hash160result": "e5c061641228a399169211e666de18448b7b8bab",
  "qualifiedname": {
    "namespace": "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV",
    "name": "vrsc::data.type.string"
  }
}
```

The `namespace` is the VRSC system namespace. All `vrsc::*` keys resolve under this namespace.

### Custom namespace key

Any currency namespace can define its own keys. Here, the `mcp3` currency on vrsctest defines a key for custom content:

```
getvdxfid "mcp3.vrsctest::test.vdxfformat"
```

```json
{
  "vdxfid": "iJPwxSRHCtD4mMp5gW7rZwuuV5zSdjyGss",
  "indexid": "xPE4RErN4CRjPXh7YBn1YLSSWk1TUV21zb",
  "hash160result": "bc7a291e12c130fcc6d6ffd128f578333347aca3",
  "qualifiedname": {
    "namespace": "i77n5FCqSBkXAK3UWHpdrPpdtXRc8sqjoz",
    "name": "mcp3.vrsctest::test.vdxfformat"
  }
}
```

The `namespace` here is the mcp3 currency's i-address. Use the returned `vdxfid` as the outer key in `contentmultimap` or as the `vdxfkey` filter in [`getidentitycontent`](getidentitycontent.md).

---

## See also

- [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md) — how VDXF keys work
- [`getidentitycontent`](getidentitycontent.md) — filter content by VDXF key
- [`updateidentity`](updateidentity.md) — write content keyed by VDXF addresses
