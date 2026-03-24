# How to Set Up Identity Security

This guide covers configuring a VerusID with external authorities and timelocks for robust security. These measures protect against key compromise by adding layers that an attacker must defeat.

**Prerequisites:**
- An active VerusID you want to secure
- At least one other VerusID to serve as guardian (for external authorities)

---

## Step 1: Set up external authorities

The strongest protection is delegating revocation and recovery to separate identities — ideally stored on different devices or held by trusted parties.

### Create guardian identities

If you don't already have them, register two additional VerusIDs:
- One for revocation (`guardian@`)
- One for recovery (`backup@`)

These can be on the same chain. Store their keys separately from your main identity — cold storage, hardware wallet, or a different machine.

### Delegate authorities

```
updateidentity '{"name":"alice","parent":"iJhCez...","revocationauthority":"guardian@","recoveryauthority":"backup@"}'
```

Verify:

```
getidentity "alice@"
```

Confirm `revocationauthority` and `recoveryauthority` show the guardian identities.

> **Never set revocation to external while leaving recovery as self.** If the external authority revokes, a self-recovery identity cannot recover itself — it is permanently bricked. See [VerusID Authority Model](../../concepts/verusid-authority-model.md).

---

## Step 2: Add a timelock

A timelock adds a time-delay before spending is possible, giving you a window to detect and stop unauthorized activity.

### Set a delay lock

```
setidentitytimelock "alice@" '{"setunlockdelay": 1440}'
```

This sets a ~1 day delay (1440 blocks). The identity is now locked — spending requires starting a countdown first.

Choose a delay based on how quickly you can respond:

| Response time | Delay (blocks) |
|---------------|----------------|
| A few hours | 360 |
| 1 day | 1,440 |
| 1 week | 10,080 |

### When you need to spend

Start the countdown:

```
setidentitytimelock "alice@" '{"unlockatblock": 0}'
```

Wait for the delay to elapse, then spend normally.

### If you see an unauthorized unlock

Revoke immediately:

```
revokeidentity "alice@"
```

This destroys the countdown. Then recover with new keys:

```
recoveridentity '{"name":"alice","parent":"iJhCez...","primaryaddresses":["RNewSafeAddr..."],"minimumsignatures":1,"revocationauthority":"guardian@","recoveryauthority":"backup@"}'
```

Set the timelock again after recovery if desired.

---

## Step 3: Verify the configuration

```
getidentity "alice@"
```

Check:
- `revocationauthority` — your guardian identity
- `recoveryauthority` — your backup identity
- `flags: 2` — delay lock is active
- `timelock: 1440` (or your chosen delay)

---

## Responding to a compromise

If you suspect your primary keys are compromised:

### 1. Revoke

From the guardian identity's wallet:

```
revokeidentity "alice@"
```

The guardian only needs to be in the wallet for signing. Use `sourceoffunds` if fees should come from a different address:

```
revokeidentity "alice@" false false 0.0001 "payer@"
```

### 2. Verify revocation

```
getidentity "alice@"
```

Confirm `status: "revoked"` and `flags` includes `32768`.

### 3. Recover

From the backup identity's wallet:

```
recoveridentity '{"name":"alice","parent":"iJhCez...","primaryaddresses":["RNewSafeAddr..."],"minimumsignatures":1,"revocationauthority":"guardian@","recoveryauthority":"backup@"}'
```

### 4. Re-enable timelock

```
setidentitytimelock "alice@" '{"setunlockdelay": 1440}'
```

### 5. Verify recovery

```
getidentity "alice@"
```

Confirm `status: "active"`, new `primaryaddresses`, and `flags: 2` (delay lock).

---

## See also

- [VerusID Authority Model](../../concepts/verusid-authority-model.md) — how authorities work and the bricking scenario
- [Timelocking Identities](../../concepts/timelocking-identities.md) — delay vs absolute locks
- [`setidentitytimelock`](../../reference/identity/setidentitytimelock.md) — timelock RPC reference
- [`revokeidentity`](../../reference/identity/revokeidentity.md) — revocation reference
- [`recoveridentity`](../../reference/identity/recoveridentity.md) — recovery reference
