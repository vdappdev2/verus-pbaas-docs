# setidentitytimelock

> Safely set or manage a timelock on a VerusID.

**Category:** Identity
**Daemon help:** `verus help setidentitytimelock`

---

## Summary

Set or modify a timelock without the dangers of the raw `timelock` field in [`updateidentity`](updateidentity.md). Provides two operations: setting a delay lock and triggering an unlock countdown. This is the recommended way to manage timelocks.

Only affects the identity on the current chain — not on other chains where the identity may have been exported.

---

## Syntax

```
setidentitytimelock "identity" '{"setunlockdelay":n}' (returntx) (feeoffer) (sourceoffunds)
setidentitytimelock "identity" '{"unlockatblock":n}' (returntx) (feeoffer) (sourceoffunds)
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `identity` | string | Yes | — | VerusID name or i-address |
| 2 | `locks` | object | Yes | — | One of `setunlockdelay` or `unlockatblock` (mutually exclusive) |
| 3 | `returntx` | boolean | No | `false` | Return hex instead of broadcasting |
| 4 | `feeoffer` | number | No | standard fee | Non-standard fee |
| 5 | `sourceoffunds` | string | No | `"*"` | Funding source. Supports wildcards (`"*"`, `"R*"`, `"i*"`), specific addresses, and VerusID names. |

### Lock operations

| Field | Type | Description |
|-------|------|-------------|
| `setunlockdelay` | number | Set a delay lock of N blocks. Sets identity `flags: 2`, `timelock: N`. |
| `unlockatblock` | number | Trigger unlock countdown. Pass `0` on a delay-locked identity to start the countdown. **Rejected on absolute locks** (`flags: 0`, `timelock > 0`). |

---

## Return value

- **Default:** Transaction ID string
- **With `returntx: true`:** Hex string

---

## Timelock workflow

The safe timelock pattern is a three-step process:

### 1. Set the delay

```
setidentitytimelock "alice@" '{"setunlockdelay": 1440}'
```

Identity is now locked with a 1440-block (~1 day) delay. State: `flags: 2`, `timelock: 1440`.

### 2. Start the countdown

When you want to unlock:

```
setidentitytimelock "alice@" '{"unlockatblock": 0}'
```

The daemon calculates the unlock block: approximately `current_block + delay + buffer`. State: `flags: 0`, `timelock: calculated_block`.

### 3. Wait

Once the block height passes the timelock value, the identity can spend again.

### Cancel the countdown

If you detect unauthorized activity during the countdown, revoke:

```
revokeidentity "alice@"
```

Revocation **destroys the countdown entirely** — timelock resets to 0. Then recover to restore with no timelock.

---

## Behavior reference

| Action | Result |
|--------|--------|
| `setunlockdelay: N` | `flags: 2`, `timelock: N` (delay lock) |
| `unlockatblock: 0` (on delay lock, flags=2) | Triggers countdown: `flags: 0`, `timelock: ~current+N` |
| `unlockatblock: 0` (on absolute lock, flags=0) | **Rejected** — script-verify-flag-failed |
| `unlockatblock: N` (on delay lock) | `flags: 0`, `timelock: adjusted_height` (daemon overrides) |
| `updateidentity` with `timelock: 0` | **Rejected** — cannot remove timelock via update |
| `recoveridentity` without `timelock` | Clears timelock to 0 |
| Revoke during countdown | Countdown destroyed, timelock resets to 0 |

---

## See also

- [Timelocking Identities](../../concepts/timelocking-identities.md) — how timelocks work, delay vs absolute, dangers
- [`revokeidentity`](revokeidentity.md) — cancel a countdown via revocation
- [`recoveridentity`](recoveridentity.md) — clear a timelock via recovery
- [How to Set Up Identity Security](../../how-to/identity/set-up-security.md) — timelocks as part of security configuration
