# Web API v1 Contract

Base path: `/api/v1`

Versioning and deprecation policy

- Current version: `v1`
- Every HTTP response includes header: `X-Cracktrader-API-Version: v1`
- Deprecations are announced in API metadata (`GET /api/v1/meta`) and use
  standard headers on affected endpoints:
  - `Deprecation`
  - `Sunset`
  - `Link`
- Minimum deprecation notice window: 90 days before removal.

Standard error envelope

```json
{
  "error": {
    "code": "RUN_ALREADY_ACTIVE",
    "message": "run already in progress",
    "details": {}
  }
}
```

Core endpoints

- `GET /health`
  - Response: `status`, `api_version`, `checks`, `metrics`, `timestamp`
- `GET /status`
  - Response: run lifecycle fields (`run_id`, `status`, `mode`, timing, pnl/value/cash, errors)
- `GET /status/detailed`
  - Response: `/status` + `recent_trades`, `equity_curve`, `performance_metrics`, `health`
- `GET /results`
  - Response: `run_id`, `config`, `status`, `start_time`, `end_time`, `trades`, `equity_curve`, `performance_metrics`, `final_value`, `error`

Run lifecycle endpoints

- `POST /run/backtest`
  - Body:
    - `strategy_name` (string)
    - `strategy_params` (object)
    - `symbols` (array[string])
    - `timeframe` (string)
    - `start_date` (string|null)
    - `end_date` (string|null)
    - `initial_cash` (number)
    - `exchange` (string)
- `POST /run/live`
  - Body:
    - `strategy_name` (string)
    - `strategy_params` (object)
    - `symbols` (array[string])
    - `timeframe` (string)
    - `exchange` (string)
    - `exchange_config` (object)
    - `mode` (`paper` | `live`)
- `POST /run/stop`
- `DELETE /run/reset`

Run response schema (`POST /run/backtest`, `POST /run/live`, `POST /run/stop`, `DELETE /run/reset`)

```json
{
  "run_id": "abc123def456",
  "status": "running",
  "message": "backtest started"
}
```

Capabilities for UI forms

- `GET /strategies`
- `GET /exchanges`

Realtime stream

- `WS /api/v1/ws/status`
- Event schema:

```json
{
  "type": "status_update",
  "payload": { "status": "running", "run_id": "abc123def456" },
  "timestamp": "2026-02-26T00:00:00Z"
}
```

Parent tracking

- This contract supports Web UI parent issue `#5` and fulfills contract hardening scope in `#6`.
