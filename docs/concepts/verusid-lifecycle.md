# VerusID Lifecycle

A VerusID moves through a defined set of states from creation to ongoing use. Each transition is an on-chain transaction with specific rules about who can invoke it and what changes.

---

## States

| State | Description | Can spend | Can sign | Can receive |
|-------|-------------|-----------|----------|-------------|
| **Active** | Normal operation | Yes | Yes | Yes |
| **Revoked** | Frozen by revocation authority | No | No | Yes (but funds invisible until recovery) |

There is no "deleted" state. A VerusID exists permanently once registered.

---

## Registration (two steps)

Identity registration uses a **commit-reveal** pattern to prevent miners from front-running desirable names.

### Step 1: Commit

[`registernamecommitment`](../reference/identity/registernamecommitment.md) creates a transaction that commits to a name using a salted hash. The name itself is not revealed on-chain — only the hash. This prevents miners from seeing a desirable name in the mempool and registering it themselves.

### Step 2: Reveal and register

After the commitment has at least 1 confirmation, [`registeridentity`](../reference/identity/registeridentity.md) reveals the name and registers the identity with its initial configuration: primary addresses, authorities, and optionally content and a private address.

### Why two steps?

Without the commitment, a miner who sees `registeridentity "coolname"` in the mempool could create their own registration for "coolname" and include it in their block first. The commitment makes this impossible — the miner cannot determine which name corresponds to which commitment hash.

The `salt` in the commitment is critical. Without it, an attacker could precompute hashes of desirable names and match them against commitments.

---

## Updates

[`updateidentity`](../reference/identity/updateidentity.md) modifies any mutable property: primary addresses, authorities, private address, and content. Only the primary authority can update (or the token authority via `tokenupdate`).

Key behaviors:
- Most fields carry over when omitted — they are preserved from the current state
- `contentmultimap` is the exception: omitting it clears content to `{}`
- Name casing can be changed; the i-address is unaffected
- Timelock cannot be removed via update — only via revoke+recover

Each update creates a new revision on-chain. Previous state is immutable and accessible via [`getidentityhistory`](../reference/identity/getidentityhistory.md).

---

## Revocation

[`revokeidentity`](../reference/identity/revokeidentity.md) freezes the identity. Only the revocation authority can do this (or token revocation).

Effects of revocation:
- `status` changes to `"revoked"`, `flags` includes `32768`
- Cannot spend funds or sign transactions
- **Can still receive** funds — they are stored but invisible and unspendable until recovery
- Any active timelock countdown is destroyed (reset to 0)

Self-revocation is impossible — an identity cannot revoke itself.

---

## Recovery

[`recoveridentity`](../reference/identity/recoveridentity.md) restores a revoked identity. Only the recovery authority can do this (or token recovery). The identity must be revoked first.

Recovery typically includes new primary addresses (since the old ones may be compromised). Key differences from update:
- `timelock` does NOT carry over — omitting resets to 0
- `privateaddress` carries over if omitted (same as update)
- Funds sent while revoked reappear after recovery

---

## Identity as namespace

When a currency is defined with the same name as a VerusID, that ID becomes the **root ID** of a namespace. Sub-IDs can be registered under it (e.g., `alice.brand@`). The identity continues to function normally — it just also serves as the controller and namespace for the currency.

Each VerusID can define at most one currency, and only once — even if the currency fails to launch.

---

## Cross-chain identity

A VerusID can be exported to other chains via [`sendcurrency`](../reference/multichain/sendcurrency.md) with `exportid: true`. The exported ID becomes an **independent copy** — it has its own update history, authorities, and content from that point forward. Changes on one chain do not affect the other.

---

## See also

- [VerusID Authority Model](verusid-authority-model.md) — how the three authorities work
- [Timelocking Identities](timelocking-identities.md) — adding time-based security
- [How to Register a VerusID](../how-to/identity/register-identity.md) — step-by-step registration
