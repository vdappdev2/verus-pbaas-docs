# Currency Launch Lifecycle

Every currency on Verus goes through a lifecycle: definition, an optional preconversion window, launch, and ongoing operation. Centralized currencies can additionally transition to decentralized control at a predetermined block height.

---

## Phases

### 1. Definition

The currency controller calls [`definecurrency`](../reference/multichain/definecurrency.md) with the currency's parameters — type, reserves, supply, fees, and launch terms. This returns a signed transaction hex, which the controller broadcasts via `sendrawtransaction`.

The controller must hold a VerusID with the same name as the currency. Each VerusID can define at most one currency, and only once — even if the launch fails.

### 2. Preconversion window (optional)

If the currency defines `currencies` (for baskets) or `currencies` + `conversions` (for simple tokens with a funding mechanism), a preconversion window opens between broadcast and `startblock`. The minimum window is 20 blocks.

During this window, anyone can participate by calling [`sendcurrency`](../reference/multichain/sendcurrency.md) with `preconvert: true`.

**Simple tokens with `conversions`** use a fixed preconversion rate. Each source currency has a price-per-token: `[0.1]` means 1 source unit buys 10 tokens. Preconversion proceeds go to the rootID as a funding mechanism.

**Basket currencies** (`options: 33`) pool all preconversions and distribute `initialsupply` proportionally. Everyone gets the same fair rate based on total contributions and configured weights. There is no price advantage for early or late participation within the window.

**Thresholds:**
- `minpreconversion` — if not met, launch fails and all preconversions are automatically refunded (no user action required)
- `maxpreconversion` — excess beyond the cap is automatically refunded after launch

A 0.025% fee is deducted from preconversions.

**Simple tokens without preconversion:** A simple token defined with only `preallocations` (no `currencies` or `conversions`) does not have a preconversion window. Supply is distributed entirely through preallocations at launch.

### 3. Launch

The currency activates at `startblock`. Verify with `getcurrency "name"`.

**For basket currencies:**
- Reserves are established from preconversions and `initialcontributions`
- `initialsupply` is distributed to preconverters proportionally
- `preallocations` are minted (lowering reserve ratio)
- `prelaunchcarveout` redirects a fraction of reserves to rootID
- On-chain conversion becomes available

**For simple tokens:**
- `preallocations` are minted
- Preconversion proceeds (if any) are delivered to rootID
- The token is active for transfers

### 4. Active operation

After launch, the currency operates according to its `proofprotocol`:

**Decentralized (proofprotocol: 1, default):** No one can mint. Sub-ID registration fees are burned — for baskets this is deflationary (less supply, same reserves).

**Centralized (proofprotocol: 2):** The rootID can mint new supply (`mintnew: true`) and burn tokens (`burn: true`) using [`sendcurrency`](../reference/multichain/sendcurrency.md). Sub-ID registration fees go to the rootID.

**Ethereum-mapped (proofprotocol: 3):** The currency is mapped 1:1 to an Ethereum token contract via a decentralized, trustless bridge. Tokens can be sent back and forth between Verus and Ethereum with provable cross-chain verification — no centralized custodian or multisig controls the bridge.

---

## The endblock transition

Centralized currencies can specify an `endblock` in their definition — the block height at which centralized control ends **permanently and irreversibly**.

After this block:
- Minting is disabled ("minting is disallowed, even by currency ID after the currency endblock")
- Burning by the rootID is disabled
- Sub-ID fee routing to rootID continues — only minting/burning ends
- The `proofprotocol` still reads 2 in the definition, but centralized operations are blocked

This enables a pattern where a currency launches with centralized control for initial management, then transitions to fully decentralized operation at a known point. The minimum gap between `startblock` and `endblock` is 480 blocks, enforced by the daemon.

---

## Launch failure

If `minpreconversion` thresholds are not met by `startblock`, the currency does not activate. All preconversions are automatically refunded to the `refundto` address (or `fromaddress` if not specified). Transaction and conversion fees are not refunded.

The currency name is permanently consumed — the VerusID cannot be used to define another currency.

---

## Costs

| Item | Cost on Verus mainnet |
|------|-----------------------|
| VerusID registration | 20–100 VRSC |
| Currency launch | 200 VRSC |
| Transaction fees | ~0.0002 VRSC |

Plus any `initialcontributions` amounts, which go to reserves (baskets) or rootID (simple tokens).

---

## See also

- [`definecurrency`](../reference/multichain/definecurrency.md) — creating currencies
- [`sendcurrency`](../reference/multichain/sendcurrency.md) — preconverting, minting, burning
- [Fractional Basket Conversions](fractional-basket-conversions.md) — how basket pricing works
- [MEV-Resistant DeFi](mev-resistant-defi.md) — why all conversions are fair
- [VerusID Lifecycle](verusid-lifecycle.md) — the identity that becomes a currency namespace
