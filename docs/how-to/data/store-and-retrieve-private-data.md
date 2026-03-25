# How to Store and Retrieve Private Data

Store arbitrary data on-chain encrypted to a Sapling z-address, then retrieve and decrypt it. The data is private — only holders of the decryption key can read it.

**Prerequisites:**
- A funded address in the wallet (transparent address or VerusID)
- A Sapling z-address in the wallet (generate one with `z_getnewaddress`)
- Familiarity with the async operation workflow (`z_getoperationstatus`)

---

## Step 1: Identify a destination z-address

Data must be sent to a Sapling z-address. Use an existing one or generate a new one:

```
z_getnewaddress
```

Returns a `zs1...` address.

Alternatively, if the data should be readable by a specific VerusID, use `"ID@:private"` — this resolves to the ID's assigned Sapling address (set via the `privateaddress` field on the identity).

---

## Step 2: Send data to the z-address

Use `sendcurrency` with the `data` parameter. `sendcurrency` handles encoding and encryption internally — you do **not** need to call `signdata` first. The data is automatically encrypted to the destination z-address. The `fromaddress` can be any funded address — it does not need to be a z-address.

### Store a text message

```
sendcurrency "myid@" '[{
  "address": "zs1...",
  "amount": 0,
  "currency": "vrsctest",
  "data": {
    "message": "hello from MCP data test 2026-03-22"
  }
}]'
```

Returns an operation ID:

```
"opid-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### Store hex or base64 data

Replace `message` with `hex` or `base64`:

```json
{
  "data": {
    "hex": "48656c6c6f"
  }
}
```

```json
{
  "data": {
    "base64": "SGVsbG8="
  }
}
```

### Store a file

> [!NOTE]
> File storage requires the `-enablefileencryption` daemon flag — pass it at startup for occasional use, or add it to the config file (e.g., `VRSCTEST.conf`) for persistent access. String, hex, and base64 data work without this flag.

```json
{
  "data": {
    "filename": "/home/user/document.pdf"
  }
}
```

> [!TIP]
> You can send multiple data payloads in one transaction by adding multiple entries to the `outputs` array, each targeting a **different** z-address. Two outputs to the same z-address in one call are rejected — use separate transactions instead.

---

## Step 3: Confirm the transaction

Poll the operation status until it succeeds:

```
z_getoperationstatus '["opid-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"]'
```

On success, the result includes the transaction ID:

```json
[{
  "id": "opid-...",
  "status": "success",
  "result": {
    "txid": "f1113235b529c73645c4cab66965204abc20b14eed98e0541984fed31f49a562"
  }
}]
```

Wait for at least 1 confirmation before retrieving.

---

## Step 4: Retrieve the data descriptor

List data received at the z-address:

```
z_listreceivedbyaddress "zs1..."
```

Data transactions appear with `amount: 0` and a `memo` field containing the **data descriptor** — a structured object that tells `decryptdata` how to find and decrypt the payload:

```json
{
  "txid": "f1113235...",
  "amount": 0.00000000,
  "memo": [
    {
      "i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv": {
        "version": 1,
        "flags": 0,
        "objectdata": {
          "iP3euVSzNcXUrLNHnQnR9G6q8jeYuGSxgw": {
            "type": 0,
            "version": 1,
            "flags": 1,
            "output": {
              "txid": "0000000000000000000000000000000000000000000000000000000000000000",
              "voutnum": 0
            },
            "objectnum": 0,
            "subobject": 0
          }
        }
      }
    }
  ]
}
```

The outer key `i4GC1YGEVD21afWudGoFJVdnfjJ5XWnCQv` is the VDXF key for `vrsc::data.type.object.datadescriptor`. The inner key `iP3euVSzNcXUrLNHnQnR9G6q8jeYuGSxgw` is `vrsc::data.type.object.crosschaindataref` — it points to the transaction containing the encrypted data. An all-zero txid with `voutnum: 0` means the data is in the same transaction as the descriptor.

Save the entire object under the `i4GC1YGEVD21...` key — you will pass it to `decryptdata` in the next step.

---

## Step 5: Export the viewing key

Export the extended viewing key for the destination z-address. The EVK is **required** for decrypting z-address data — without it, `decryptdata` returns the data still encrypted (`flags: 5`) even if the wallet holds the spending key.

```
z_exportviewingkey "zs1..."
```

Returns:

```
"zxviews1q..."
```

The EVK grants read access to **all** data encrypted to this z-address without granting spending authority. Share it carefully.

---

## Step 6: Decrypt the data

Pass the data descriptor, transaction ID, and (if not using wallet keys) the EVK to `decryptdata`:

```
decryptdata '{
  "datadescriptor": {
    "version": 1,
    "flags": 0,
    "objectdata": {
      "iP3euVSzNcXUrLNHnQnR9G6q8jeYuGSxgw": {
        "type": 0,
        "version": 1,
        "flags": 1,
        "output": {
          "txid": "0000000000000000000000000000000000000000000000000000000000000000",
          "voutnum": 0
        },
        "objectnum": 0,
        "subobject": 0
      }
    }
  },
  "txid": "f1113235b529c73645c4cab66965204abc20b14eed98e0541984fed31f49a562",
  "retrieve": true,
  "evk": "zxviews1q..."
}'
```

- `datadescriptor` — the descriptor object from Step 4
- `txid` — the transaction containing the data
- `retrieve: true` — instructs the daemon to fetch and decrypt
- `evk` — the extended viewing key (required for z-address data — do not omit)

Returns the decrypted payload:

```json
[
  {
    "version": 1,
    "flags": 2,
    "objectdata": "68656c6c6f2066726f6d204d43502064617461207465737420323032362d30332d3232",
    "salt": "ce9a1c150ec71ded2aad94809ff972426a7892dfcade6b13985d4066b253fc04"
  }
]
```

- `flags: 2` indicates decrypted data (the original descriptor had `flags: 0`)
- `objectdata` is the content as hex
- `salt` was auto-generated by the daemon during encryption

---

## Step 7: Decode the hex output

The `objectdata` value is hex-encoded. Decode it to recover the original content:

```
echo "68656c6c6f2066726f6d204d43502064617461207465737420323032362d30332d3232" | xxd -r -p
```

Output:

```
hello from MCP data test 2026-03-22
```

For binary files (images, PDFs, etc.), write the hex to a file instead:

```python
binary = bytes.fromhex(objectdata_hex)
with open("recovered-image.png", "wb") as f:
    f.write(binary)
```

Or via shell:

```
echo "<hex>" | xxd -r -p > recovered-image.png
```

> Full binary round-trip confirmed on vrsctest, 2026-03-24: a 898 KB PNG stored via `sendcurrency:data` with `filename`, retrieved via `z_listreceivedbyaddress` + `decryptdata`, and decoded from hex — byte-for-byte identical to the original (MD5 match).

---

## Cost

Data storage fees scale with transaction size. Observed on vrsctest:

| Payload | Transaction size | Fee |
|---|---|---|
| Short text message | 1587 bytes | 0.00354 VRSCTEST |

Larger payloads produce larger transactions with proportionally higher fees. The maximum data payload is **1,000,000 bytes (1 MB)**. A 898 KB PNG image cost 9.48 VRSCTEST.

---

## Common pitfalls

**Do not call `signdata` first.** `sendcurrency:data` and `signdata` are independent pipelines that share the same input keys (`message`, `messagehex`, etc.). Passing `signdata`'s output as the `data` parameter fails. `sendcurrency` handles encryption automatically — do not pass `encrypttoaddress` either. For the identity content path (where `signdata` *is* needed), see [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md).

**Destination must be a z-address.** Sending data to a transparent address or VerusID (without `:private`) fails with "Cannot use data parameter unless sending to a private address."

**Always pass the EVK for z-address data.** Without the EVK, `decryptdata` returns data still encrypted (`flags: 5`) even if the wallet holds the spending key. This is specific to z-address data retrieved via `datadescriptor` + `txid` + `retrieve: true`. Export the EVK with `z_exportviewingkey` and always include it. (Confirmed 2026-03-24.)

**Hex output requires decoding.** `decryptdata` returns content as hex in `objectdata`, not as plaintext. The caller must decode it — text via `xxd -r -p`, binary files via hex-to-bytes conversion.

**Older data remains visible.** `z_listreceivedbyaddress` returns all data ever sent to the z-address, not just the most recent. Filter by `txid` or block height as needed.

---

## See also

- [On-Chain Data Storage and Encryption](../../concepts/on-chain-data-storage-and-encryption.md) — how the two storage paths and encryption model work
- [How to Encrypt Data on a Public Identity](encrypt-data-on-public-identity.md) — the alternative path for public-but-encrypted data
- [`sendcurrency`](../../reference/multichain/sendcurrency.md) — the `data` parameter
- [`signdata`](../../reference/data/signdata.md) — the data object format
