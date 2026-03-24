# Timelocking Identities

Timelocks add a time-delay to identity spending, creating a window during which unauthorized transactions can be detected and stopped via revocation. This is one of the strongest security features available for VerusIDs.

---

## Why timelock?

If an attacker compromises a VerusID's primary keys, they can immediately spend all funds. A timelock changes this: even with valid keys, spending is delayed by a configurable number of blocks. During the delay, the legitimate owner (or their revocation authority) can revoke the identity, destroying the countdown and preventing the theft.

This is analogous to a bank hold on large withdrawals — the delay gives time for human intervention.

---

## Two types of locks

### Delay lock (recommended)

Set via [`setidentitytimelock`](../reference/identity/setidentitytimelock.md) with `setunlockdelay`. The identity stores a delay duration (in blocks) and enters a locked state (`flags: 2`).

To spend, the owner must first trigger a countdown (`unlockatblock: 0`), then wait for the delay to elapse. During the countdown, anyone monitoring the identity can see the unlock in progress and revoke if unauthorized.

**This is the safe, recommended approach.**

### Absolute lock (dangerous)

Set by including `timelock: N` in the identity JSON during [`registeridentity`](../reference/identity/registeridentity.md) or [`recoveridentity`](../reference/identity/recoveridentity.md). The identity is locked until block height N.

**Absolute locks cannot be removed by `updateidentity`** — only by revoke+recover. If set incorrectly (e.g., a block height far in the future), the identity is effectively bricked until that height is reached or a revoke+recover cycle is performed.

**Avoid absolute locks.** Use `setidentitytimelock` with delay locks instead.

---

## Delay lock workflow

### 1. Set the delay

```
setidentitytimelock "alice@" '{"setunlockdelay": 1440}'
```

The identity is now locked with a 1440-block (~1 day) delay.
State: `flags: 2`, `timelock: 1440`

### 2. Request unlock

When you want to spend:

```
setidentitytimelock "alice@" '{"unlockatblock": 0}'
```

The daemon calculates the target block: approximately `current_block + 1440`.
State: `flags: 0`, `timelock: target_block`

### 3. Wait

The identity cannot spend until the block height passes the timelock value. During this window, monitor for unauthorized unlock attempts.

### 4. Spend

After the target block is reached, the identity can spend normally.

---

## Cancelling an unlock

If you see an unauthorized unlock countdown in progress:

```
revokeidentity "alice@"
```

Revocation **destroys the countdown entirely** — timelock resets to 0. The attacker's unlock is cancelled even if it was about to complete.

Then recover with new keys:

```
recoveridentity '{"name":"alice","parent":"...","primaryaddresses":["RNewSafeAddr..."],"minimumsignatures":1}'
```

Recovery resets timelock to 0. Set a new delay lock after recovery if desired.

---

## Behavior summary

| Action | Result |
|--------|--------|
| `setunlockdelay: N` | Lock with N-block delay (`flags: 2`, `timelock: N`) |
| `unlockatblock: 0` on delay lock | Start countdown (`flags: 0`, `timelock: ~current+N`) |
| `unlockatblock: 0` on absolute lock | **Rejected** |
| `updateidentity` with `timelock: 0` | **Rejected** — cannot remove any timelock via update |
| `recoveridentity` without `timelock` | Resets timelock to 0 (timelock does NOT carry over) |
| `recoveridentity` with `timelock: N` | Sets absolute lock (`flags: 0`), not delay lock |
| Revoke during countdown | Countdown destroyed, timelock resets to 0 |

---

## Timelock is per-chain

Timelocks only apply on the chain where they are set. If a VerusID is exported to another chain, the exported copy does not inherit the timelock from the source chain.

---

## See also

- [`setidentitytimelock`](../reference/identity/setidentitytimelock.md) — RPC reference
- [VerusID Authority Model](verusid-authority-model.md) — revocation as the cancel mechanism
- [How to Set Up Identity Security](../how-to/identity/set-up-security.md) — practical security guide
