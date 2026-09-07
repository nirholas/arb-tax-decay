# ArbTaxDecay

**Prices the staleness of a pool: the longer a pool goes untraded, the more the next swap pays.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://arb-tax-decay.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/ArbTaxDecayHook.sol`](src/hooks/ArbTaxDecayHook.sol)
- **Licence:** MIT

## How it works

Loss-versus-rebalancing is the dominant cost of providing liquidity to a constant-function AMM. It is paid when an arbitrageur brings a stale pool price back to the market price, and the size of that arbitrage grows with how long the pool sat unpriced. Ordinary flow does not have this property: a swap that lands one second after another swap is almost certainly not an arbitrage, because there was no time for the reference price to drift.

This hook turns that observation into a fee. It measures the time since the pool last traded and adds a surcharge that grows with it along a saturating curve, capped at `maxSurcharge`: surcharge(elapsed) = maxSurcharge * elapsed / (elapsed + halfLife) At `elapsed == halfLife` the arbitrageur pays half the cap; a swap in the same second as the previous one pays only `baseFee`. The surcharge is an LP fee, so the value it captures is paid to in-range liquidity providers.

The hook never custodies funds and holds no privileged role. Two properties make this cheap to reason about. It needs no oracle, so there is nothing to manipulate and no liveness dependency.

And it is monotone in a quantity the arbitrageur cannot control: waiting longer to arbitrage a pool only raises the toll, so the strategy that minimizes the tax is to trade the pool more often, which is exactly the behaviour that keeps the price fresh for everyone else. Prior art: dynamic-fee hooks keyed on realized volatility or on price movement are common, and the LVR literature (Milionis, Moallemi, Roughgarden, Zhang) motivates charging arbitrageurs more. Keying the fee on time-since-last- trade rather than on a price signal is the part that is new here, and it is what removes the oracle.

Limitation, stated plainly: on a pool that trades continuously the surcharge is near zero, so this hook does nothing for a busy major pair. It is aimed at the long tail, where pools are quiet for minutes or hours at a time and the arbitrage on the first trade back is the whole of the LP's loss.

## Prior art

Dynamic-fee hooks keyed on realized volatility or on price movement are common, and the loss-versus-rebalancing literature (Milionis, Moallemi, Roughgarden, Zhang) motivates charging arbitrageurs more. Keying the fee on time since the last trade rather than on a price signal is what is new here, and it is what removes the oracle.

## Where it does not help

On a pool that trades continuously the surcharge is near zero, so this does nothing for a busy major pair. It is aimed at the long tail, where pools sit quiet for minutes or hours and the arbitrage on the first trade back is the whole of the provider loss.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    ArbTaxDecayHook.Config({
        baseFee: /* uint24 */ 0,
        maxSurcharge: /* uint24 */ 0,
        halfLife: /* uint32 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `baseFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `maxSurcharge` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `halfLife` | `uint32` | seconds |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidHalfLife()` | `halfLife` was zero, which would make every swap pay the full surcharge. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SurchargeTooLarge()` | `baseFee + maxSurcharge` must leave room under the 100% protocol maximum. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # ArbTaxDecay
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # mev, lvr, dynamic-fee, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/arb-tax-decay
cd arb-tax-decay
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
