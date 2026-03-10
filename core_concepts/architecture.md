# Legacy Architecture Context

This page remains for historical links and compatibility context.

It is not the primary architecture truth anymore.

## Use These Pages First

If you are trying to understand the current runtime, start here instead:

1. [Runtime Map](../architecture/runtime_map.md)
2. [Architecture Index](../architecture/agent_index.md)
3. [Mode Matrix](../architecture/mode_matrix.md)
4. [Runtime Terms](../architecture/runtime_terms.md)

## Why This Page Changed

Older Cracktrader docs were organized around a simpler mental model:

- Cerebro as the center of the system
- feeds and brokers as the main runtime boundaries
- stores as the dominant integration surface

That framing is still useful when reading legacy compatibility code or migration-oriented pages, but it is no longer the preferred top-level explanation.

The current runtime direction is session-owned and contract-first:

- shared reference data and state coordination
- multi-strategy fanout over one shared snapshot per step
- execution contexts and stable route IDs
- central inventory, exposure, and risk
- unified execution adapters across backtest, paper, sandbox, and live
- post-trade and control-plane hooks around the runtime edge

## How To Read Older Pages

When you encounter older terms, reinterpret them this way:

| Older emphasis | How to read it now |
| --- | --- |
| Cerebro-centric execution | important compatibility surface, not the whole runtime map |
| Feed + broker wiring | one layer inside a broader session-owned orchestration model |
| Store as architecture center | external IO and transport ownership, not runtime ownership |
| Risk manager near broker | one slice of a broader explicit risk engine |

## Where Legacy Framing Still Helps

Older pages are still useful for:

- Backtrader compatibility behavior
- exchange/store implementation detail
- feed or broker-specific usage notes
- migration work where old and new abstractions coexist

## Next Reading

- [Strategies](strategies.md)
- [Feeds](feeds.md)
- [Brokers](brokers.md)
- [Exchanges](exchanges.md)
