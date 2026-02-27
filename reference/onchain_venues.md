# On-chain Venues (Uniswap / PancakeSwap)

This page documents the v1 on-chain venue surface in Cracktrader.

Supported in v1:
- `uniswap` (v2 constant-product path)
- `pancakeswap` (v2 constant-product path)
- chains: Ethereum (1), BSC (56), Polygon (137) via connector config

## Factory and Session Usage

Use the same high-level factory/session style as other venues:

```python
import cracktrader as ct

session = ct.exchange("uniswap", mode="paper")
feed = session.feed(symbol="WETH/USDC", amount_in=10**18)
broker = session.broker(mode="paper")
```

PancakeSwap uses the same API:

```python
session = ct.exchange("pancakeswap", mode="live")
feed = session.feed(symbol="WETH/USDC", amount_in=10**18)
broker = session.broker(mode="live")
```

## End-to-end Example (Backtest Engine Callback Path)

```python
from cracktrader.engine.feed_port import InMemoryFeedCursor, InMemoryFeedPort
from cracktrader.engine_runtime import CracktraderEngine
from cracktrader.onchain.backtest import (
    BacktestReserveFrame,
    OnchainBacktestDataset,
    OnchainBacktestBrokerCallback,
)

dataset = OnchainBacktestDataset(
    data_name="WETH/USDC",
    token_in="WETH",
    token_out="USDC",
    chain_id=1,
    pool_address="0xpool",
    frames=[
        BacktestReserveFrame(
            timestamp=1700000000.0,
            block_number=19000000,
            reserve_in=1_000_000_000_000_000_000_000,
            reserve_out=2_000_000_000_000,
            gas_price_wei=20_000_000_000,
            gas_used=120_000,
            approval_gas_used=45_000,
            fee_bps=30,
            quote_per_native=3000.0,
        )
    ],
)

engine = CracktraderEngine()
feed_port = InMemoryFeedPort(
    [InMemoryFeedCursor(data_name=dataset.data_name, candles=dataset.to_candles())]
)
results = engine.run_native(
    feed_port=feed_port,
    strategy_callback=lambda _events: [],
    broker_callback=OnchainBacktestBrokerCallback(dataset=dataset),
    broker_callback_mode="intent_results",
    initial_cash=10_000.0,
)
```

## Configuration Notes

### Chain and endpoints
- Configure `chain_id` and router address on connector/store construction.
- RPC endpoint and relay settings should be environment-specific (paper/backtest/live).

### Token registry and symbols
- Symbol mapping follows the allowlist model in [On-chain Token Registry](onchain_token_registry.md).
- Use explicit `TOKEN_IN/TOKEN_OUT` symbols for on-chain feeds.

### Slippage and quote policy
- Quote path uses `QuoteEngine` and v2 reserve snapshots.
- `slippage_bps` controls `minAmountOut` bounds for swap tx build.
- Backtest simulator uses the same quote and min-out path for parity.

### Private relay policy
- Tx lifecycle and private-first fallback semantics are defined in the architecture ADR:
  [On-chain Venue Architecture ADR](../plans/onchain_venue_architecture_adr.md).

## Fee and Accounting Surface

On-chain broker results expose:
- tx lifecycle-derived order status
- `fee_breakdown` including LP fee impact, gas, approval gas, replacement gas
- diagnostics (`tx_state`, `tx_hash`, `chain_id`, `attempt_id`, replacement/fallback flags)

## Security Notes

- Use encrypted local key material and never log private key data.
- Keep signer implementations behind the signer abstraction.
- See [On-chain Signer Security Checklist](../plans/onchain_signer_security_checklist.md).

## Troubleshooting

For operational failure handling (stuck tx, replacement, fallback, reorg), see:
- [On-chain Operations Runbook](../support/onchain_ops_runbook.md)

