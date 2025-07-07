# Mocking Policy

## Philosophy
Use mocking when:
- External API would slow tests
- Side effects are untestable (network, filesystem)

Avoid mocking:
- Internal broker logic
- Backtrader data feeds unless necessary

## Good Examples
- `mock_store.fetch_trading_fees.return_value = ...`
- `patch("asyncio.run_coroutine_threadsafe")`

## Bad Examples
- Mocking `Position` updates
- Mocking `CCXTOrder.update()` directly

