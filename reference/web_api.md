# Web API Reference

REST endpoints exposed by the Cracktrader FastAPI server. Base path: `/api/v1`.

Authentication

- No authentication by default (development). Add your own in production.

Health and Status

- GET `/health`: Overall health with checks and metrics
- GET `/status`: Current run status (basic fields)
- GET `/status/detailed`: Extended status (recent trades, equity, metrics)

Runs

- POST `/run/backtest`: Start a backtest
  - Body: BacktestRequest
    - `strategy_name` (str)
    - `strategy_params` (dict)
    - `symbols` (list[str])
    - `timeframe` (str, e.g., `"1h"`)
    - `start_date` (ISO str, optional)
    - `end_date` (ISO str, optional)
    - `initial_cash` (float)
    - `exchange` (str)
- POST `/run/live`: Start paper or live trading
  - Body: LiveTradingRequest
    - `strategy_name` (str)
    - `strategy_params` (dict)
    - `symbols` (list[str])
    - `timeframe` (str)
    - `exchange` (str)
    - `exchange_config` (dict; API keys and settings)
    - `mode` ("paper" | "live")
- POST `/run/stop`: Stop the current run
- DELETE `/run/reset`: Reset the runner to idle

Results and Configuration

- GET `/results`: Full results for the last run
- GET `/strategies`: Available strategies and parameters
- GET `/exchanges`: Common exchanges and sandbox notes

Models (responses)

- HealthResponse
  - `status` (str), `checks` (dict), `metrics` (dict), `timestamp` (ISO)
- StatusResponse
  - `run_id`, `status`, `mode`, `start_time`, `end_time`, `runtime_seconds`, `cash`, `value`, `pnl`, `trade_count`, `error`
- DetailedStatusResponse
  - Inherits StatusResponse; adds `recent_trades`, `equity_curve`, `performance_metrics`, `health`
- RunResponse
  - `run_id`, `status`, `message`
- ResultsResponse
  - `run_id`, `config`, `status`, `start_time`, `end_time`, `trades`, `equity_curve`, `performance_metrics`, `final_value`, `error`

Examples

Start a backtest

```http
POST /api/v1/run/backtest
Content-Type: application/json

{
  "strategy_name": "TestStrategy",
  "strategy_params": {"period": 20, "threshold": 0.02},
  "symbols": ["BTC/USDT"],
  "timeframe": "1h",
  "start_date": "2024-01-01T00:00:00Z",
  "end_date": "2024-03-01T23:59:59Z",
  "initial_cash": 10000.0,
  "exchange": "binance"
}
```

Start paper trading

```http
POST /api/v1/run/live
Content-Type: application/json

{
  "strategy_name": "TestStrategy",
  "strategy_params": {"period": 10, "slow": 30},
  "symbols": ["BTC/USDT"],
  "timeframe": "5m",
  "exchange": "binance",
  "exchange_config": {"apiKey": "...", "secret": "...", "sandbox": true},
  "mode": "paper"
}
```

Stop and reset

```http
POST /api/v1/run/stop
DELETE /api/v1/run/reset
```

```json
{
  "channel": "orders",
  "data": {
    "order_id": "12345",
    "status": "filled",
    "symbol": "BTC/USDT",
    "side": "buy",
    "amount": 0.1,
    "price": 50000,
    "timestamp": 1640995200000
  }
}
```

## Error Handling

### HTTP Status Codes

- `200`: Success
- `400`: Bad Request
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Not Found
- `429`: Rate Limited
- `500`: Internal Server Error

### Error Response Format

```json
{
  "error": {
    "code": "INVALID_SYMBOL",
    "message": "Symbol BTC/USD is not supported",
    "details": {
      "supported_symbols": ["BTC/USDT", "ETH/USDT"]
    }
  }
}
```

## Rate Limits

- REST API: 100 requests per minute per API key
- WebSocket: 10 subscriptions per connection
- Order placement: 10 orders per second

## SDKs and Libraries

### Python

```python
from cracktrader import CrackTraderAPI

api = CrackTraderAPI(api_key="your_key")
balance = api.get_balance()
```

### JavaScript

```javascript
import CrackTrader from 'cracktrader-js';

const api = new CrackTrader({apiKey: 'your_key'});
const balance = await api.getBalance();
```
