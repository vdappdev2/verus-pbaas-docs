# How to Register a VerusID

This guide walks through registering a new VerusID, including fee discovery for sub-IDs under different currency namespaces.

**Prerequisites:**
- A wallet with sufficient funds for the registration fee
- An R-address in the wallet to use as `controladdress`

---

## Step 1: Choose the namespace

- **Root ID** (e.g., `alice@` on VRSC): no parent needed. Fee is the chain's `idregistrationfees` (100 VRSC on mainnet, reduced with referral).
- **Sub-ID** (e.g., `alice.brand@`): registered under a currency namespace. Fee varies — see [Step 2](#step-2-discover-the-fee).

---

## Step 2: Discover the fee

For root IDs, the fee is well-known (100 VRSC on mainnet, 80 with referral).

For sub-IDs, use the **fee shortcut**: attempt registration with `feeoffer: 0.00000001`. The daemon rejects and tells you the exact minimum:

```
registeridentity '{"txid":"anything","namereservation":{"name":"test","salt":"0","referral":"","parent":"...","nameid":"..."},"identity":{"name":"test","parent":"...","primaryaddresses":["RMyAddr..."],"minimumsignatures":1}}' false 0.00000001
```

Response:
```
"Fee offer must be at least 50.00000000"
```

This check runs **before commitment validation** — no cost, no on-chain effect. Works even without a valid commitment.

---

## Step 3: Commit the name

```
registernamecommitment "alice" "myid@"
```

For a sub-ID with referral:

```
registernamecommitment "newsub" "myid@" "existingsub.brand@" "brand"
```

**Save the entire output** — you need the `txid` and `namereservation` (including the `salt`) for step 4.

```json
{
  "txid": "abc123...",
  "namereservation": {
    "name": "alice",
    "salt": "def456...",
    "referral": "",
    "parent": "iJhCez...",
    "nameid": "i1234..."
  }
}
```

---

## Step 4: Wait 1 block

The commitment must have at least 1 confirmation before registration. Check with:

```
getblockcount
```

---

## Step 5: Register

Pass the commitment data and identity definition:

```
registeridentity '{
  "txid": "abc123...",
  "namereservation": {
    "name": "alice",
    "salt": "def456...",
    "referral": "",
    "parent": "iJhCez...",
    "nameid": "i1234..."
  },
  "identity": {
    "name": "alice",
    "parent": "iJhCez...",
    "primaryaddresses": ["RMyAddr..."],
    "minimumsignatures": 1
  }
}'
```

To delegate authorities (recommended for valuable identities):

```json
{
  "identity": {
    "name": "alice",
    "parent": "iJhCez...",
    "primaryaddresses": ["RMyAddr..."],
    "minimumsignatures": 1,
    "revocationauthority": "guardian@",
    "recoveryauthority": "backup@"
  }
}
```

> **Safety:** If you set `revocationauthority` to an external identity, always also set `recoveryauthority` to a different external identity. Never leave one external and one as self. See [VerusID Authority Model](../../concepts/verusid-authority-model.md).

---

## Step 6: Verify

After 1 confirmation:

```
getidentity "alice@"
```

Check:
- `status: "active"`
- `primaryaddresses` matches what you set
- `revocationauthority` and `recoveryauthority` are correct

---

## Optional: Set content at registration

You can include `contentmultimap` in the identity definition to store data from the start:

```json
{
  "identity": {
    "name": "alice",
    "parent": "iJhCez...",
    "primaryaddresses": ["RMyAddr..."],
    "minimumsignatures": 1,
    "contentmultimap": {
      "iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c": [
        {"iK7a5JNJnbeuYWVHCDRpJosj3irGJ5Qa8c": "My first VerusID"}
      ]
    }
  }
}
```

---

## Optional: Assign a private address

Include `privateaddress` to enable receiving at a shielded z-address via `"alice@:private"`:

```json
{
  "identity": {
    "name": "alice",
    "parent": "iJhCez...",
    "primaryaddresses": ["RMyAddr..."],
    "minimumsignatures": 1,
    "privateaddress": "zs1..."
  }
}
```

---

## See also

- [`registernamecommitment`](../../reference/identity/registernamecommitment.md) — step 1 reference
- [`registeridentity`](../../reference/identity/registeridentity.md) — step 2 reference
- [VerusID Lifecycle](../../concepts/verusid-lifecycle.md) — why registration is two steps
- [VerusID Authority Model](../../concepts/verusid-authority-model.md) — safe authority setup
