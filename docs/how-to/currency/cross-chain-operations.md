# How to Send Currency and Export IDs Cross-Chain

This guide covers sending currency between chains, exporting currency definitions, and exporting VerusIDs. Examples use the VRSC ↔ vDEX pair.

**Prerequisites:**
- A funded wallet on the source chain
- A valid destination address on the destination chain (VerusID, i-address, or R-address)

---

## Understand what the destination chain knows

Each chain is independent. A chain can only receive currency it knows about and can only route to addresses it recognizes.

- **Bridge reserve currencies are already known.** If a currency is part of a chain's bridge reserves (e.g., VRSC is a reserve of Bridge.vDEX), it does not need to be exported — the chain already has the definition.
- **Other currencies must be exported first.** Before sending a non-bridge currency cross-chain, its definition must be exported to the destination chain using `exportcurrency`.
- **IDs require their namespace.** Before a VerusID can be exported to a chain, the chain must know the ID's namespace currency. If `kaiju` hasn't been exported to vDEX, then `alice.kaiju@` can't be exported there either.

---

## Send currency cross-chain

Send VRSC from the VRSC chain to an address on vDEX:

```
sendcurrency "myid@" '[{"currency":"VRSC","amount":10,"exportto":"vDEX","address":"recipient@"}]'
```

- `exportto` — the destination chain
- `address` — must be valid on the destination chain (the recipient needs their private key imported there)
- VRSC is a bridge reserve currency on vDEX, so no currency export is needed

Track the operation:

```
z_getoperationstatus
```

Transfer time depends on block times, notarization frequency, and bridge traffic.

---

## Export a currency definition

Before sending a non-bridge currency to another chain, export its definition. Anyone can do this — it is not restricted to the currency's defining ID.

```
sendcurrency "myid@" '[{"currency":"kaiju","amount":0,"exportto":"vDEX","exportcurrency":true,"address":"myid@"}]'
```

- `amount: 0` — no value is transferred, only the definition
- `exportcurrency: true` — triggers the definition export
- This is a one-time operation per currency per destination chain

### Fee

The fee is the `currencyimportfee` from the destination chain's currency definition, denominated in the destination chain's native coin:

| Destination | `currencyimportfee` | Denomination |
|-------------|---------------------|--------------|
| vDEX | 5 | vDEX |
| VRSC | 100 | VRSC |

Check the fee for any chain:

```
getcurrency "vDEX"
```

Look for the `currencyimportfee` field.

After the export confirms, the currency can be sent cross-chain and IDs in that namespace can be exported.

---

## Export a VerusID

Export a VerusID to another chain. The ID becomes an independent entity on the destination chain — updated separately from the source.

### Prerequisite

The ID's namespace currency must already be known on the destination chain. For root-level IDs (e.g., `myid@`), the chain's native currency is the namespace — always known. For sub-IDs (e.g., `alice.kaiju@`), the parent currency (`kaiju`) must have been exported first.

### Export the ID

```
sendcurrency "myid@" '[{"currency":"VRSC","amount":0,"exportto":"vDEX","exportid":true,"address":"myid@"}]'
```

- `fromaddress` (first parameter) — must be the ID being exported
- `exportid: true` — triggers the ID export
- `amount: 0` — no value transferred
- The fee is the `idimportfees` from the destination chain's currency definition (0.002 vDEX for vDEX, 0.02 VRSC for VRSC)

### After export

- Import the private key on the destination chain to control the exported ID there
- The exported ID starts as a copy of the source — same addresses, authorities, and content
- Changes to the ID on either chain are independent from that point on

> [!NOTE]
> If the ID is exported before a control token is created for it, the exported ID cannot later be controlled by that token. Create the control token first if needed.

---

## Typical workflow

### Send a known currency cross-chain

If the currency is already known on the destination (bridge reserve or previously exported):

```
sendcurrency "myid@" '[{"currency":"VRSC","amount":50,"exportto":"vDEX","address":"recipient@"}]'
```

### Send a new currency cross-chain

1. Export the currency definition:

```
sendcurrency "myid@" '[{"currency":"kaiju","amount":0,"exportto":"vDEX","exportcurrency":true,"address":"myid@"}]'
```

2. Wait for confirmation, then send:

```
sendcurrency "myid@" '[{"currency":"kaiju","amount":100,"exportto":"vDEX","address":"recipient@"}]'
```

### Export an ID and send it funds

1. Export the namespace currency if needed (skip for root-level IDs):

```
sendcurrency "myid@" '[{"currency":"kaiju","amount":0,"exportto":"vDEX","exportcurrency":true,"address":"myid@"}]'
```

2. Export the ID:

```
sendcurrency "alice.kaiju@" '[{"currency":"VRSC","amount":0,"exportto":"vDEX","exportid":true,"address":"alice.kaiju@"}]'
```

3. Send funds to the exported ID on the destination chain:

```
sendcurrency "myid@" '[{"currency":"VRSC","amount":25,"exportto":"vDEX","address":"alice.kaiju@"}]'
```

---

## See also

- [`sendcurrency`](../../reference/multichain/sendcurrency.md) — full parameter reference for cross-chain modes
- [`getcurrency`](../../reference/multichain/getcurrency.md) — check `currencyimportfee` and `idimportfees` on destination chains
- [`z_getoperationstatus`](../../reference/multichain/z_getoperationstatus.md) — track cross-chain operation progress
