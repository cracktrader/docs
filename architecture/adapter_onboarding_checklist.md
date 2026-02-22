# Exchange Adapter Onboarding Checklist

Use this checklist when adding a new adapter under `src/cracktrader/exchanges/adapters/`.

1. Implement the `ExchangeAdapter` protocol surface:
- `submit_order_async`
- `cancel_order_async`
- `normalize_order_event`
- `normalize_balance_event`
- `capabilities`

2. Reuse standardized capability keys:
- Start from `default_adapter_capabilities()` in `adapters/base.py`.
- Only override values when behavior is intentionally different.

3. Normalize status text through engine-domain normalization:
- Use `normalize_order_status_text(...)` for order event status output.
- Ensure `normalize_order_event` is idempotent for identical payloads.

4. Add adapter contract coverage:
- Extend `tests/contracts/adapters/test_exchange_adapter_contract.py` for submit/cancel and normalization.
- Verify capability matrix key parity with existing adapters.

5. Route one real broker/store path through the adapter seam:
- Avoid broad rewrites in the same slice.
- Add/adjust unit coverage for the touched live path.

6. Update architecture decision log:
- Append a decision entry in `docs/architecture/refactor_decisions.md`.
