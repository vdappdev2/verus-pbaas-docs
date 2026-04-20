# VerusID Multisig

Every VerusID has a primary authority defined by two fields:

- `primaryaddresses` — transparent R-addresses that can sign on behalf of the identity
- `minimumsignatures` — how many of those addresses must sign (the threshold `M`)

When `minimumsignatures > 1`, the identity is a multisig. Spending, sending, updating content, and signing data all require `M` signatures from the addresses in `primaryaddresses`.

---

## Hard caps

| Parameter | Limit |
|-----------|-------|
| `primaryaddresses` length (`N`) | 1 ≤ N ≤ 25 |
| `minimumsignatures` (`M`) | 1 ≤ M ≤ 13 |
| Relationship | M ≤ N |

The caps are enforced by the daemon at both [`registeridentity`](../reference/identity/registeridentity.md) and [`updateidentity`](../reference/identity/updateidentity.md). Invalid configurations are rejected with `"Invalid identity"`. The caps are identical for root IDs and sub-IDs.

**The M cap is a hard ceiling, not a majority rule.** 13-of-25 is permitted (where 13 = ceil(25/2)), but so is 13-of-24 or 13-of-13. What's rejected is any M > 13, regardless of N.

### Verified on vrsctest

| `M of N` | Result |
|---|---|
| 13 of 25 | accepted |
| 14 of 25 | rejected |
| 1 of 26 | rejected |
| 13 of 24 | accepted |
| 13 of 13 | accepted |
| 1 of 25 | accepted |

---

## Common patterns

| Pattern | Typical use |
|---------|-------------|
| 1 of 1 | Single-signer identity — the default |
| 1 of N | Any one of several devices or people can sign — convenience over control |
| 2 of 3 | Classic resilient multisig — lose one key without losing the identity |
| 3 of 5 | Enterprise or committee control |
| 13 of 25 | Maximum allowed multisig — deep redundancy or large voting body |

---

## Address rules

- Every entry in `primaryaddresses` must be a **transparent R-address**. i-addresses and z-addresses are rejected.
- Addresses may be reused across identities — an R-address is not consumed by being listed as a primary.
- The `controladdress` passed to [`registernamecommitment`](../reference/identity/registernamecommitment.md) is independent from the final identity's `primaryaddresses`. The commitment and the identity definition each have their own key setup.

---

## Multisig and the other authorities

The [VerusID authority model](verusid-authority-model.md) has three independent authorities:

- **Primary** — this is what `primaryaddresses`/`minimumsignatures` defines; controls spending and day-to-day signing.
- **Revocation** — a VerusID (not an R-address) that can revoke.
- **Recovery** — a VerusID that can restore a revoked identity.

Revocation and recovery authorities point to **other VerusIDs**, and those VerusIDs can themselves be multisig. This lets you compose structures like:

- Primary: 2-of-3 with daily-use keys (fast)
- Recovery authority: a separate 3-of-5 multisig VerusID held by different stakeholders (slow and deliberate)

The caps (M ≤ 13, N ≤ 25) apply to each identity individually. They do not compose across authority delegation.

---

## Signing flow

### Same-wallet signers (simple case)

When all of a multisig ID's primary R-addresses are in a single wallet, spending is automatic. `sendcurrency`, `updateidentity`, etc. construct, sign, and broadcast the transaction in one step. The wallet signs with **every primary key it holds** — even beyond the threshold. Additional signatures above `M` are valid and simply included in the input script.

This is the common case when an operator runs a node holding all signing keys.

### Split-wallet signers (co-signing)

When primary keys live on separate wallets, the transaction must be co-signed:

1. One wallet builds the transaction (typically by calling the write RPC with a flag such as `returntx: true`) and signs with its share of the keys. The result is a partially-signed hex.
2. The hex is handed to each additional signer.
3. Each signer runs [`signrawtransaction`](#) on the hex, adding their signature. The command returns the updated hex.
4. Once the hex contains at least `M` valid signatures, any party can broadcast it with `sendrawtransaction`.

The network only accepts the final transaction if the input script contains `M` or more signatures from the current `primaryaddresses` set. Additional signatures beyond the threshold are permitted but not required.

---

## Enforcement at update time

The caps apply to **every state of the identity**, not just registration. `updateidentity` rejects the same invalid configurations (`M > 13`, `N > 25`, `M > N`) that `registeridentity` rejects. You cannot create a valid identity and later mutate it into an over-cap configuration.

---

## See also

- [VerusID Authority Model](verusid-authority-model.md) — how primary, revocation, and recovery authorities interact
- [`registeridentity`](../reference/identity/registeridentity.md) — register a new identity with any valid `M of N`
- [`updateidentity`](../reference/identity/updateidentity.md) — change the primary signing set of an existing identity
- [How to Register a Multisig VerusID](../how-to/identity/register-multisig-identity.md) — practical 2-of-3 walkthrough
