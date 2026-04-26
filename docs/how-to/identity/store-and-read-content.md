# How to Store and Read Identity Content

This guide covers storing structured data on a VerusID using `contentmultimap` and reading it back with VDXF key filtering.

**Prerequisites:**
- An active VerusID you control
- Familiarity with VDXF keys (see [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md))

---

## Step 1: Choose or create a VDXF key

Every piece of content is stored under an outer VDXF key that identifies the application or data category. Resolve a key name to its i-address:

```
getvdxfid "vrsc::data.type.string"
```

```json
{
  "vdxfid": "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c"
}
```

For application-specific data, use a custom namespace key:

```
getvdxfid "myapp.vrsc::user.profile"
```

Use the returned `vdxfid` as the outer key in `contentmultimap`.

---

## Step 2: Read existing content

Before writing, read the current `contentmultimap` — omitting it in an update **clears all content**:

```
getidentity "alice@"
```

Check the `contentmultimap` field. If it has data you want to keep, include it in your update.

---

## Step 3: Write content

### Simple typed value

Store a string under the string type key:

```
updateidentity '{"name":"alice","parent":"iJhCez...","contentmultimap":{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c":[{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c":"Hello from alice"}]}}'
```

### DataDescriptor with metadata

Store richer content with mimetype and label:

```
updateidentity '{"name":"alice","parent":"iJhCez...","contentmultimap":{"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c":[{"i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv":{"version":1,"objectdata":{"message":"Hello from alice"},"mimetype":"text/plain","label":"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c"}}]}}'
```

The daemon auto-sets `flags: 96` when `mimetype` and `label` are present.

### Choosing the objectdata shape by mimetype

The shape you write determines how the daemon stores and returns the value. Use the shapes below for round-trip stability.

| mimetype | Write `objectdata` as | Reads back as |
|---|---|---|
| `text/plain` | `{"message": "<utf8 string>"}` | `{"message": "<utf8 string>"}` |
| `application/json` | `"<utf8ToHex(jsonString)>"` (raw hex string at top level) | `"<hex>"` (raw hex string at top level) |

For `text/plain`, `{"serializedhex": "<hex>"}` is also accepted on write but the daemon decodes it back to `{"message": "<utf8 string>"}` on read. Stick to `{"message": ...}` for symmetry.

For `application/json`, `{"message": "<jsonString>"}` and `{"serializedhex": "<hex>"}` are also accepted on write and round-trip to the same canonical `"<hex>"` form. Writing the raw hex directly avoids the daemon's rewrite.

**Critical:** Do not use `{"serializedbase64": "<b64>"}` under `application/json`. The daemon accepts the call (returns a txid, mines a transaction) but silently drops the data — the on-chain entry has `objectdata: null`.

A reader for `application/json` content must hex-decode `objectdata` as a string and then JSON-parse the result. The `text/plain` `{message}` shape and the `application/json` raw-hex shape are the two distinct patterns a complete reader needs to cover.

### Preserving existing content

If the identity already has content you want to keep, merge it:

```
getidentity "alice@"
```

Copy the existing `contentmultimap`, add your new entries, and pass the combined object to `updateidentity`.

---

## Step 4: Read content back

### Current state only

```
getidentity "alice@"
```

Shows only the most recent update's `contentmultimap`.

### Cumulative content

```
getidentitycontent "alice@"
```

Shows all content across all updates — including entries from updates that were later overwritten.

### Filtered by VDXF key

```
getidentitycontent "alice@" 0 -1 false "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c"
```

Returns only entries under the specified outer key, across all updates. Use this to retrieve your application's data without parsing unrelated content.

### Per-revision history

```
getidentityhistory "alice@"
```

Shows what each individual update contained — useful for auditing what changed when.

---

## Common pitfalls

**Omitting `contentmultimap` clears it.** This is the most common mistake. If you update any other field (e.g., `primaryaddresses`) without including `contentmultimap`, all visible content is cleared. Always read first, then include existing content in the update.

**`{message: "text"}` is not valid.** The `signdata`/`sendcurrency:data` message format does not work in contentmultimap entries. Use `{vdxfTypeKey: value}` or `{dataDescriptorKey: {version, objectdata, ...}}`.

**Bare strings are silently ignored.** `["text"]` in the entry array is accepted but stores nothing. Always wrap values in a typed object.

**`{serializedbase64}` under `application/json` silently drops data.** The daemon accepts the call and mines the transaction, but the on-chain `objectdata` is `null`. Use `objectdata: "<utf8ToHex(jsonString)>"` (raw hex string at top level) instead.

---

## See also

- [VDXF and Identity Content](../../concepts/vdxf-and-identity-content.md) — full content format and VDXF key system
- [`getvdxfid`](../../reference/identity/getvdxfid.md) — resolve VDXF key names
- [`updateidentity`](../../reference/identity/updateidentity.md) — write content
- [`getidentitycontent`](../../reference/identity/getidentitycontent.md) — read with filtering
- [`getidentityhistory`](../../reference/identity/getidentityhistory.md) — per-revision audit
