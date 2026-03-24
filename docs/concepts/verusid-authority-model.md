# VerusID Authority Model

Every VerusID has three independent authorities that control different aspects of the identity. Understanding how they interact — and how they can brick an identity if misconfigured — is essential before creating or modifying any VerusID.

---

## The three authorities

### Primary authority

The `primaryaddresses` and `minimumsignatures` fields define who can **spend funds** and **sign transactions** on behalf of the identity. This is the day-to-day authority — it controls the identity's wallet, authorizes sends, updates content, and signs data.

- Up to 25 R-addresses for multi-sig (up to 13 signatures required)
- Only transparent R-addresses — not i-addresses or z-addresses
- Changed via [`updateidentity`](../reference/identity/updateidentity.md) (by the primary authority itself) or [`recoveridentity`](../reference/identity/recoveridentity.md) (by the recovery authority)

### Revocation authority

The `revocationauthority` is a VerusID that can **revoke** this identity, freezing it completely. A revoked identity cannot spend, sign, or authorize anything. This is the emergency brake — use it when keys are compromised.

- Must be a VerusID (friendly name or i-address), not an R-address
- Defaults to self if not specified
- Only needs to be in the wallet for signing — a separate address can pay the transaction fee
- Invoked via [`revokeidentity`](../reference/identity/revokeidentity.md)

### Recovery authority

The `recoveryauthority` is a VerusID that can **restore** a revoked identity with new state — typically new primary addresses. This is the only way to bring a revoked identity back to life.

- Must be a VerusID (friendly name or i-address), not an R-address
- Defaults to self if not specified
- Only needs to be in the wallet for signing — a separate address can pay the fee
- Invoked via [`recoveridentity`](../reference/identity/recoveridentity.md)

### Token authority

Some identities have an associated **control token** — a currency with the same name and a total supply of exactly 1 satoshi (0.00000001). Whoever holds the control token can revoke and recover the identity via `tokenrevoke` and `tokenrecover` parameters, independent of the named revocation/recovery authorities.

Check for a control token: `flags: 5` in [`getidentity`](../reference/identity/getidentity.md) output, or call `getcurrency` with the identity's name or i-address — if a currency definition exists with a supply of 0.00000001, it's a control token.

---

## How authorities interact

The state machine is simple:

```
Active ──[revoke]──> Revoked ──[recover]──> Active
```

- **Only the revocation authority** (or token revocation) can move to the revoked state
- **Only the recovery authority** (or token recovery) can move back to active
- **Primary authority** cannot revoke or recover — it is powerless once the identity is revoked

This separation means a compromised primary key can be neutralized (revoke) and replaced (recover) by authorities that are stored separately, potentially on different machines, in cold storage, or held by trusted third parties.

---

## The bricking scenario

The most dangerous misconfiguration is setting `revocationauthority` to an external identity while leaving `recoveryauthority` as self:

```json
{
  "revocationauthority": "external-guardian@",
  "recoveryauthority": "myid@"
}
```

If `external-guardian@` revokes the identity:
1. The identity is now revoked
2. Only `recoveryauthority` (self = `myid@`) can recover it
3. But `myid@` is revoked — it cannot authorize anything, including its own recovery
4. The identity is **permanently bricked**

**The daemon does not prevent this configuration.** It is accepted without warning.

### Real-world example

`vault.bitcoins@` on Verus mainnet is permanently bricked in exactly this way. Its history (74 revisions) shows an authority battle: the owner repeatedly recovered with self/self authorities, but an external revocation authority's transaction landed in the same block as a recovery, revoking the identity after it was set back to self/self. The result is a state that no single transaction could create — revoked with self as both authorities.

### Self-revocation is impossible

An identity whose `revocationauthority` is itself cannot revoke itself. The daemon returns "mandatory-script-verify-flag-failed". This is logically consistent: the identity would need to authorize its own revocation, but after revocation it can't authorize anything. This applies to both root IDs and sub-IDs.

---

## Safe configurations

### Self-sovereign (default)

```json
{
  "revocationauthority": "myid@",
  "recoveryauthority": "myid@"
}
```

The identity controls itself. It cannot be revoked by anyone (self-revocation is impossible), and recovery is not needed. Simplest, but no emergency brake if keys are compromised.

### External guardian

```json
{
  "revocationauthority": "guardian@",
  "recoveryauthority": "backup@"
}
```

A trusted guardian can revoke if keys are compromised. A separate backup identity can recover with new keys. **Both must be external** — never set one external and one to self.

### Mutual guardianship

Two identities guard each other:

- `alice@`: revoke=`bob@`, recover=`bob@`
- `bob@`: revoke=`alice@`, recover=`alice@`

Either can rescue the other. Both must be vigilant.

---

## Authority and fees

The authority identity only needs to be **in the wallet for signing**. It does not need to pay the transaction fee. Use `sourceoffunds` to specify a different address for fee payment. This matters when the authority is on a hardware wallet or cold storage device — it signs the transaction, but fees come from a hot wallet.

---

## See also

- [`updateidentity`](../reference/identity/updateidentity.md) — change authorities
- [`revokeidentity`](../reference/identity/revokeidentity.md) — invoke revocation authority
- [`recoveridentity`](../reference/identity/recoveridentity.md) — invoke recovery authority
- [VerusID Lifecycle](verusid-lifecycle.md) — registration, updates, revocation, recovery
- [How to Set Up Identity Security](../how-to/identity/set-up-security.md) — practical guide
