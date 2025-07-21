# Cracktrader Web API

A comprehensive RESTful API and web interface for managing cryptocurrency trading strategies with Cracktrader.

## Overview

The Cracktrader Web API provides a modern HTTP interface to all Cracktrader functionality, allowing you to:

- Run backtests through HTTP requests
- Manage live and paper trading sessions  
- Monitor system health and performance
- Access real-time trading data and results
- Control strategy execution remotely

## Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│   React Web    │    │   FastAPI        │    │   Cracktrader       │
│   Frontend      │◄──►│   Server         │◄──►│   Core Engine       │
│   (Optional)    │    │                  │    │                     │
└─────────────────┘    └──────────────────┘    └─────────────────────┘
                              │
                              ▼
                       ┌──────────────────┐
                       │  CracktraderRunner │
                       │  (API Wrapper)     │
                       └──────────────────┘
```

### Components

1. **FastAPI Server** (`src/cracktrader/web/server/main.py`)
   - RESTful API endpoints
   - Request/response validation
   - Error handling and logging
   - Health monitoring integration

2. **CracktraderRunner** (`src/cracktrader/web/server/api_runner.py`)
   - API wrapper around core Cracktrader components
   - State management for trading sessions
   - Background task execution
   - Real-time status tracking

3. **React Frontend** (`src/cracktrader/web/frontend/`)
   - Modern web dashboard
   - Real-time monitoring interface
   - Strategy configuration forms
   - Results visualization

## Quick Start

### 1. Install Dependencies

```bash
# Install web dependencies
pip install 'cracktrader[web]'

# Or install manually
pip install fastapi uvicorn websockets pydantic
```

### 2. Start the API Server

```bash
# Using the CLI script
python scripts/run_web_server.py

# Or directly
python -m cracktrader.web.server.main

# Custom host/port
python scripts/run_web_server.py --host 0.0.0.0 --port 8080
```

### 3. Access the API

- **API Documentation**: http://127.0.0.1:8000/docs
- **Health Check**: http://127.0.0.1:8000/api/v1/health
- **Status**: http://127.0.0.1:8000/api/v1/status

### 4. Optional: Start Frontend

```bash
cd src/cracktrader/web/frontend
npm install
npm run dev
```

Visit http://localhost:3000 for the web dashboard.

## API Reference

### Core Endpoints

#### Health & Status

```http
GET /api/v1/health
```
Get system health status including exchange connectivity and performance metrics.

```http
GET /api/v1/status
GET /api/v1/status/detailed
```
Get current runner status with optional detailed metrics.

#### Trading Operations

```http
POST /api/v1/run/backtest
Content-Type: application/json

{
  "strategy_name": "TestStrategy",
  "strategy_params": {"period": 20, "threshold": 0.02},
  "symbols": ["BTC/USDT"],
  "timeframe": "1h",
  "start_date": "2024-01-01T00:00:00Z",
  "end_date": "2024-12-31T23:59:59Z",
  "initial_cash": 10000.0,
  "exchange": "binance"
}
```

```http
POST /api/v1/run/live
Content-Type: application/json

{
  "strategy_name": "TestStrategy",
  "strategy_params": {"period": 20, "threshold": 0.02},
  "symbols": ["BTC/USDT"],
  "timeframe": "1h",
  "exchange": "binance",
  "exchange_config": {
    "apiKey": "your_api_key",
    "secret": "your_secret"
  },
  "mode": "paper"
}
```

```http
POST /api/v1/run/stop
DELETE /api/v1/run/reset
```

#### Results & Configuration

```http
GET /api/v1/results
GET /api/v1/strategies
GET /api/v1/exchanges
```

### Response Formats

#### Status Response
```json
{
  "run_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "running",
  "mode": "backtest",
  "start_time": "2024-01-15T10:30:00Z",
  "runtime_seconds": 45.2,
  "cash": 9850.00,
  "value": 10125.50,
  "pnl": 125.50,
  "trade_count": 5
}
```

#### Results Response
```json
{
  "run_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "completed",
  "final_value": 10125.50,
  "performance_metrics": {
    "sharpe_ratio": 1.25,
    "max_drawdown": 0.05,
    "total_trades": 12,
    "win_rate": 0.67,
    "total_return": 0.0125
  },
  "trades": [...],
  "equity_curve": [...]
}
```

## Usage Examples

### Python Client

```python
import asyncio
import aiohttp

async def run_backtest():
    async with aiohttp.ClientSession() as session:
        # Start backtest
        async with session.post('http://127.0.0.1:8000/api/v1/run/backtest', json={
            "strategy_name": "TestStrategy",
            "symbols": ["BTC/USDT"],
            "timeframe": "1h",
            "initial_cash": 10000.0
        }) as resp:
            result = await resp.json()
            run_id = result['run_id']
        
        # Monitor progress
        while True:
            async with session.get('http://127.0.0.1:8000/api/v1/status') as resp:
                status = await resp.json()
                if status['status'] in ['completed', 'failed']:
                    break
            await asyncio.sleep(1)
        
        # Get results
        async with session.get('http://127.0.0.1:8000/api/v1/results') as resp:
            results = await resp.json()
            print(f"Final value: ${results['final_value']}")

asyncio.run(run_backtest())
```

### JavaScript/Node.js Client

```javascript
const axios = require('axios');

const API_BASE = 'http://127.0.0.1:8000/api/v1';

async function runBacktest() {
  // Start backtest
  const { data: startResult } = await axios.post(`${API_BASE}/run/backtest`, {
    strategy_name: 'TestStrategy',
    symbols: ['BTC/USDT'],
    timeframe: '1h',
    initial_cash: 10000.0
  });
  
  console.log(`Backtest started: ${startResult.run_id}`);
  
  // Monitor progress
  while (true) {
    const { data: status } = await axios.get(`${API_BASE}/status`);
    console.log(`Status: ${status.status}`);
    
    if (['completed', 'failed'].includes(status.status)) {
      break;
    }
    
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  // Get results
  const { data: results } = await axios.get(`${API_BASE}/results`);
  console.log(`Final value: $${results.final_value}`);
}

runBacktest().catch(console.error);
```

### cURL Examples

```bash
# Check health
curl http://127.0.0.1:8000/api/v1/health

# Start backtest
curl -X POST http://127.0.0.1:8000/api/v1/run/backtest \\
  -H "Content-Type: application/json" \\
  -d '{
    "strategy_name": "TestStrategy",
    "symbols": ["BTC/USDT"],
    "timeframe": "1h",
    "initial_cash": 10000.0
  }'

# Check status
curl http://127.0.0.1:8000/api/v1/status

# Get results
curl http://127.0.0.1:8000/api/v1/results

# Stop run
curl -X POST http://127.0.0.1:8000/api/v1/run/stop

# Reset runner
curl -X DELETE http://127.0.0.1:8000/api/v1/run/reset
```

## Configuration

### Server Configuration

The server can be configured through environment variables or command-line arguments:

```bash
# Environment variables
export CRACKTRADER_API_HOST="0.0.0.0"
export CRACKTRADER_API_PORT="8000"
export CRACKTRADER_LOG_LEVEL="info"

# Command line
python scripts/run_web_server.py --host 0.0.0.0 --port 8000 --log-level debug
```

### Exchange Configuration

For live and paper trading, configure exchange API credentials:

```json
{
  "exchange_config": {
    "apiKey": "your_api_key_here",
    "secret": "your_secret_here",
    "password": "your_passphrase_if_required",
    "sandbox": true,
    "rateLimit": 1200,
    "enableRateLimit": true
  }
}
```

**Security Note**: Never expose API keys in logs or client-side code. Use environment variables or secure configuration management.

## Error Handling

The API uses standard HTTP status codes and returns structured error responses:

```json
{
  "error": "Runner is not idle. Current status: running",
  "status_code": 400,
  "path": "/api/v1/run/backtest"
}
```

Common error codes:
- `400` - Bad Request (invalid parameters)
- `404` - Not Found (endpoint doesn't exist)
- `409` - Conflict (runner in wrong state)
- `500` - Internal Server Error

## Real-time Updates

For real-time updates, poll the status endpoint or implement WebSocket connections (future enhancement).

```python
# Polling example
async def monitor_run():
    while True:
        status = await get_status()
        if status['status'] in ['completed', 'failed']:
            break
        await asyncio.sleep(1)  # Poll every second
```

## Production Deployment

### Security Considerations

1. **Authentication**: Implement proper authentication for production use
2. **HTTPS**: Use TLS/SSL for encrypted communication
3. **API Keys**: Secure exchange API credentials
4. **Rate Limiting**: Implement request rate limiting
5. **CORS**: Configure appropriate CORS policies

### Deployment Options

#### Docker (Recommended)

```dockerfile
FROM python:3.11-slim

COPY . /app
WORKDIR /app

RUN pip install -e .'[web]'

EXPOSE 8000

CMD ["python", "scripts/run_web_server.py", "--host", "0.0.0.0"]
```

#### Cloud Platforms

- **AWS**: Deploy using ECS, Lambda, or Elastic Beanstalk
- **Google Cloud**: Use Cloud Run or App Engine
- **Azure**: Deploy with Container Instances or App Service
- **Heroku**: Direct deployment with Procfile

### Monitoring

- Use the `/api/v1/health` endpoint for health checks
- Integrate with monitoring tools (Prometheus, Grafana)
- Set up alerts for failed runs or system issues
- Monitor logs for errors and performance metrics

## Limitations

- Single concurrent run per runner instance
- Strategy hot-reloading requires server restart
- WebSocket real-time updates not yet implemented
- No built-in user authentication
- Limited to strategies available in the Python environment

## Contributing

To contribute to the web API:

1. Add new endpoints in `routes.py`
2. Update the `CracktraderRunner` for new functionality
3. Add Pydantic models for request/response validation
4. Update this documentation
5. Add tests for new endpoints

## Support

For issues with the web API:

1. Check the server logs for error details
2. Verify all dependencies are installed correctly
3. Test endpoints with the interactive docs at `/docs`
4. Report bugs with full request/response details