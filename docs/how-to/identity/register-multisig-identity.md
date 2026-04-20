# How to Register a Multisig VerusID

This guide walks through registering a **2-of-3 multisig** sub-identity and then spending from it. A 2-of-3 is the simplest useful multisig: losing one key doesn't lock you out, but no single key can unilaterally spend.

For the underlying rules and caps (M ≤ 13, N ≤ 25, etc.), see [VerusID Multisig](../../concepts/verusid-multisig.md).

**Prerequisites:**
- A wallet with funds for the registration fee (in the parent currency)
- 3 R-addresses you control. These can all be in one wallet (simpler, recommended for the first pass) or split across wallets.
- An R-address in the local wallet to use as `controladdress` for the commitment

---

## Step 1: Prepare the three R-addresses

Generate or identify three transparent R-addresses. If you want them all in your current wallet, use `getnewaddress` three times.

```
getnewaddress
getnewaddress
getnewaddress
```

Note the three addresses — they will be the identity's `primaryaddresses`.

---

## Step 2: Commit the name

Sub-IDs register under a parent currency. In this example the parent is `broom` and the child will be `demo2of3.broom@`.

```
registernamecommitment "demo2of3" "<controladdress>" "" "broom" "broom@"
```

Parameters (positional): `name`, `controladdress`, `referralidentity` (empty here), `parentnameorid`, `sourceoffunds`.

The response looks like:

```json
{
  "txid": "b0af540568be873819a8cd621b57064db17ef9dbadd79629c7baa5a1ffe0f95f",
  "namereservation": {
    "version": 1,
    "name": "demo2of3",
    "parent": "iHYt61ZrCopWmDEZkb6Ta9Z2z5jMJZvfwA",
    "salt": "953567dfff210e2c830d7c97ff103f125941f019af59dc0de9eb8d24868b89a8",
    "referral": "",
    "nameid": "iK6UKtgZDYfzHXhnTjtqyLSDAD6Mvrjbop"
  }
}
```

Keep the entire response — you need `txid` and `namereservation` for the next step.

---

## Step 3: Wait one confirmation

The commitment must be mined before `registeridentity` will accept it.

```
getblockcount
```

If `registeridentity` is called too early, it returns `"Invalid or unconfirmed commitment transaction id"`. Wait a block and retry.

---

## Step 4: Register with 3 primary addresses, threshold 2

Pass the commitment plus an identity definition with `primaryaddresses` set to the three R-addresses and `minimumsignatures: 2`.

```
registeridentity '{
  "txid": "b0af540568be873819a8cd621b57064db17ef9dbadd79629c7baa5a1ffe0f95f",
  "namereservation": {
    "version": 1,
    "name": "demo2of3",
    "parent": "iHYt61ZrCopWmDEZkb6Ta9Z2z5jMJZvfwA",
    "salt": "953567dfff210e2c830d7c97ff103f125941f019af59dc0de9eb8d24868b89a8",
    "referral": ""
  },
  "identity": {
    "name": "demo2of3",
    "parent": "iHYt61ZrCopWmDEZkb6Ta9Z2z5jMJZvfwA",
    "primaryaddresses": [
      "RVey9PeLHsz3mK1DD98zVG5qTMUHksLuwi",
      "RNeTBRk2jkcyMh9S52YNzvsSJpmuZtmZqY",
      "RTrP479vGgweC6mYWYqmfVur6q22AGaJde"
    ],
    "minimumsignatures": 2
  }
}' false 0 "broom@"
```

Returned value: the registration transaction id.

---

## Step 5: Verify

```
getidentity "demo2of3.broom@"
```

Check:
- `status: "active"`
- `primaryaddresses` lists all 3 R-addresses in the order you supplied
- `minimumsignatures: 2`
- `canspendfor: true`, `cansignfor: true` — the wallet holds enough keys to act on the identity's behalf

---

## Step 6: Spend from the multisig ID

To prove the threshold is working, send some value from the identity.

### Fund the identity first

A freshly registered identity holds no coin. Send it a small amount from any wallet:

```
sendcurrency "broom@" '[{"address":"demo2of3.broom@","amount":0.2,"currency":"VRSCTEST"}]'
```

This returns an operation id; poll it:

```
z_getoperationstatus '["opid-..."]'
```

Confirm the balance landed:

```
getcurrencybalance "demo2of3.broom@"
```

### Send from the multisig ID

Now send back out from the multisig. Because all three primary keys are in the same wallet, the daemon signs with **every key it holds** and broadcasts the transaction in one call — no manual cosigning needed:

```
sendcurrency "demo2of3.broom@" '[{"address":"broom@","amount":0.05,"currency":"VRSCTEST"}]'
```

Poll the opid and note the resulting txid.

### Observe the signatures

Inspect the raw hex of the outgoing transaction. For a 2-of-3 ID whose wallet holds all 3 keys, the input script will contain **3 (pubkey, signature) pairs** — one per primary address — even though only 2 were required. Additional signatures above the threshold are valid and simply included.

In a 2-of-3 where one signer is absent, the input script would contain only 2 pairs, and the transaction would still be accepted on-chain.

---

## Split-wallet cosigning

If the three primary R-addresses live in **different wallets**, spending requires a partial-signature round-trip:

1. Wallet A builds the transaction and signs with its key(s) — typically by invoking the RPC with a `returntx` flag, which yields partially-signed hex.
2. Wallet A hands the hex to Wallet B.
3. Wallet B runs `signrawtransaction` on the hex. The command re-signs with any keys the wallet holds and returns updated hex.
4. Repeat until the hex carries at least `minimumsignatures` valid signatures.
5. Any party broadcasts the final hex with `sendrawtransaction`.

If the hex reaches threshold, `signrawtransaction` reports `"complete": true`, signaling it is ready to broadcast.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| `"Invalid identity"` | `minimumsignatures > 13` or `primaryaddresses.length > 25` or `minimumsignatures > primaryaddresses.length`. See [VerusID Multisig](../../concepts/verusid-multisig.md). |
| `"Invalid or unconfirmed commitment transaction id"` | Commitment has not been mined yet. Wait for a block. |
| `"Insufficient funds for identity registration"` | The `sourceoffunds` does not hold enough of the parent currency to cover `idregistrationfees`. Check with `getcurrency` on the parent and `getcurrencybalance` on the source. |
| `"Invalid parent currency"` | Pass the parent as a bare name (e.g., `"broom"`) or as an i-address. Do not append `@`. |

---

## See also

- [VerusID Multisig](../../concepts/verusid-multisig.md) — caps and signing model
- [VerusID Authority Model](../../concepts/verusid-authority-model.md) — interaction with revocation and recovery
- [`registernamecommitment`](../../reference/identity/registernamecommitment.md)
- [`registeridentity`](../../reference/identity/registeridentity.md)
- [`updateidentity`](../../reference/identity/updateidentity.md) — change the primary signing set later
- [How to Register a VerusID](register-identity.md) — single-sig baseline
