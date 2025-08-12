# Web API & Dashboard Guide

Cracktrader provides a comprehensive REST API and modern React dashboard for remote strategy management, real-time monitoring, and results analysis. This guide covers API usage, dashboard features, and integration patterns.

## Overview

### Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   REST API      │    │   Trading       │
│   Dashboard     │◄──►│   (FastAPI)     │◄──►│   Engine        │
│   (React/TS)    │    │                 │    │   (Cerebro)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │              ┌─────────────────┐              │
         │              │  CracktraderRunner              │
         │              │  (API Wrapper)  │              │
         │              └─────────────────┘              │
         │                       │                       │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   WebSocket     │    │   Health        │    │   Data Feeds    │
│   Real-time     │◄──►│   Monitoring    │◄──►│   & Brokers     │
│   Updates       │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Key Features

- **REST API** - Complete trading control via HTTP endpoints
- **React Dashboard** - Modern web interface with real-time updates
- **Strategy Management** - Deploy, configure, and monitor strategies
- **Real-time Monitoring** - Live performance metrics and health status
- **Results Analysis** - Interactive charts and trade history
- **Multi-mode Support** - Backtest, paper, and live trading

## Getting Started

### Starting the Web Server

```bash
# Basic server start
python scripts/run_web_server.py

# Custom configuration
python scripts/run_web_server.py --config config.yaml --port 8080

# Production mode
python scripts/run_web_server.py --workers 4 --log-level info

# With custom host
python scripts/run_web_server.py --host 0.0.0.0 --port 8000
```

### Accessing the Interface

```bash
# Web Dashboard
http://localhost:8000

# API Documentation (Swagger UI)
http://localhost:8000/docs

# Alternative API Documentation (ReDoc)
http://localhost:8000/redoc

# Health Check
http://localhost:8000/api/v1/health
```

## REST API Reference

### Authentication

```python
# API currently supports public access
# Future versions will include API key authentication

import requests

# Base URL
BASE_URL = "http://localhost:8000/api/v1"

# Health check (no auth required)
response = requests.get(f"{BASE_URL}/health")
print(response.json())
```

### Core Endpoints

#### Health and Status

```python
import requests

# System health
def check_health():
    response = requests.get(f"{BASE_URL}/health")
    return response.json()

# Runner status
def get_status():
    response = requests.get(f"{BASE_URL}/status")
    return response.json()

# Detailed status with metrics
def get_detailed_status():
    response = requests.get(f"{BASE_URL}/status/detailed")
    return response.json()

# Usage
health = check_health()
print(f"System status: {health['status']}")

status = get_status()
print(f"Runner status: {status['status']}")
print(f"Current run ID: {status['run_id']}")
```

#### Strategy Management

```python
# List available strategies
def list_strategies():
    response = requests.get(f"{BASE_URL}/strategies")
    return response.json()

# List supported exchanges
def list_exchanges():
    response = requests.get(f"{BASE_URL}/exchanges")
    return response.json()

# Usage
strategies = list_strategies()
print(f"Available strategies: {strategies['strategies']}")

exchanges = list_exchanges()
print(f"Supported exchanges: {exchanges['exchanges']}")
```

#### Backtest Execution

```python
# Start backtest
def start_backtest(strategy_config):
    response = requests.post(f"{BASE_URL}/run/backtest", json=strategy_config)
    return response.json()

# Example backtest configuration
backtest_config = {
    "strategy_name": "MovingAverageCross",
    "strategy_params": {
        "fast_period": 10,
        "slow_period": 30,
        "printlog": True
    },
    "symbols": ["BTC/USDT"],
    "timeframe": "1h",
    "start_date": "2024-01-01T00:00:00Z",
    "end_date": "2024-03-01T23:59:59Z",
    "initial_cash": 10000.0,
    "exchange": "binance"
}

# Start backtest
result = start_backtest(backtest_config)
run_id = result["run_id"]
print(f"Backtest started with ID: {run_id}")

# Monitor progress
import time
while True:
    status = get_status()
    print(f"Status: {status['status']}")

    if status['status'] in ['completed', 'failed']:
        break

    time.sleep(2)

# Get results
def get_results():
    response = requests.get(f"{BASE_URL}/results")
    return response.json()

results = get_results()
print(f"Final portfolio value: ${results['final_value']:,.2f}")
```

#### Live Trading

```python
# Start paper trading
def start_paper_trading(strategy_config):
    response = requests.post(f"{BASE_URL}/run/live", json=strategy_config)
    return response.json()

# Paper trading configuration
paper_config = {
    "strategy_name": "MovingAverageCross",
    "strategy_params": {
        "fast_period": 10,
        "slow_period": 30
    },
    "symbols": ["BTC/USDT"],
    "timeframe": "5m",
    "exchange": "binance",
    "exchange_config": {
        "apiKey": "your_api_key",
        "secret": "your_secret",
        "sandbox": True  # Use testnet
    },
    "mode": "paper"
}

# Start paper trading
result = start_paper_trading(paper_config)
print(f"Paper trading started: {result['message']}")

# Monitor live trading
while True:
    status = get_detailed_status()
    print(f"Portfolio value: ${status.get('value', 0):,.2f}")
    print(f"Recent trades: {len(status.get('recent_trades', []))}")

    time.sleep(30)  # Check every 30 seconds
```

#### Run Control

```python
# Stop current run
def stop_run():
    response = requests.post(f"{BASE_URL}/run/stop")
    return response.json()

# Reset runner
def reset_runner():
    response = requests.delete(f"{BASE_URL}/run/reset")
    return response.json()

# Usage
if status['status'] == 'running':
    result = stop_run()
    print(f"Stop result: {result['success']}")

# Reset after completion
reset_result = reset_runner()
print(f"Reset: {reset_result['message']}")
```

### Advanced API Usage

#### Async Client

```python
import asyncio
import aiohttp

class CracktraderAPIClient:
    """Async API client for Cracktrader."""

    def __init__(self, base_url="http://localhost:8000/api/v1"):
        self.base_url = base_url
        self.session = None

    async def __aenter__(self):
        self.session = aiohttp.ClientSession()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()

    async def get_status(self):
        async with self.session.get(f"{self.base_url}/status") as response:
            return await response.json()

    async def start_backtest(self, config):
        async with self.session.post(
            f"{self.base_url}/run/backtest",
            json=config
        ) as response:
            return await response.json()

    async def monitor_run(self, check_interval=2):
        """Monitor run until completion."""
        while True:
            status = await self.get_status()

            yield status

            if status['status'] in ['completed', 'failed']:
                break

            await asyncio.sleep(check_interval)

    async def get_results(self):
        async with self.session.get(f"{self.base_url}/results") as response:
            return await response.json()

# Usage
async def run_async_backtest():
    backtest_config = {
        "strategy_name": "TestStrategy",
        "symbols": ["BTC/USDT"],
        "timeframe": "1h",
        "initial_cash": 10000.0
    }

    async with CracktraderAPIClient() as client:
        # Start backtest
        result = await client.start_backtest(backtest_config)
        print(f"Started run: {result['run_id']}")

        # Monitor progress
        async for status in client.monitor_run():
            print(f"Status: {status['status']}")
            if 'value' in status:
                print(f"Portfolio: ${status['value']:,.2f}")

        # Get final results
        results = await client.get_results()
        print(f"Final value: ${results['final_value']:,.2f}")

# Run async
asyncio.run(run_async_backtest())
```

#### WebSocket Real-time Updates

```python
import websockets
import json
import asyncio

async def monitor_live_trading():
    """Monitor live trading via WebSocket."""
    uri = "ws://localhost:8000/ws/status"

    async with websockets.connect(uri) as websocket:
        print("Connected to real-time updates")

        async for message in websocket:
            data = json.loads(message)

            if data['type'] == 'status_update':
                status = data['payload']
                print(f"Portfolio: ${status.get('value', 0):,.2f}")
                print(f"PnL: ${status.get('pnl', 0):,.2f}")

            elif data['type'] == 'trade_execution':
                trade = data['payload']
                print(f"Trade: {trade['side']} {trade['size']} @ ${trade['price']}")

            elif data['type'] == 'error':
                print(f"Error: {data['message']}")
                break

# Run WebSocket monitor
asyncio.run(monitor_live_trading())
```

## Dashboard Features

### Strategy Configuration

```typescript
// Frontend strategy configuration
interface StrategyConfig {
  name: string;
  params: Record<string, any>;
  symbols: string[];
  timeframe: string;
  mode: 'backtest' | 'paper' | 'live';
  exchange: string;
  exchangeConfig?: {
    apiKey?: string;
    secret?: string;
    sandbox?: boolean;
  };
}

// Example configuration form
const strategyConfig: StrategyConfig = {
  name: "MovingAverageCross",
  params: {
    fast_period: 10,
    slow_period: 30,
    stop_loss: 0.05,
    take_profit: 0.15
  },
  symbols: ["BTC/USDT", "ETH/USDT"],
  timeframe: "1h",
  mode: "backtest",
  exchange: "binance"
};
```

### Real-time Monitoring

```typescript
// Real-time status monitoring
interface TradingStatus {
  run_id: string;
  status: 'idle' | 'starting' | 'running' | 'completed' | 'failed';
  mode: 'backtest' | 'paper' | 'live';
  start_time: string;
  runtime_seconds: number;
  cash: number;
  value: number;
  pnl: number;
  trade_count: number;
  recent_trades: Trade[];
  equity_curve: EquityPoint[];
  performance_metrics: PerformanceMetrics;
}

// Dashboard component example
function TradingDashboard() {
  const [status, setStatus] = useState<TradingStatus>();
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    const interval = setInterval(async () => {
      const response = await fetch('/api/v1/status/detailed');
      const newStatus = await response.json();
      setStatus(newStatus);
      setIsRunning(newStatus.status === 'running');
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  return (
    <div className="trading-dashboard">
      <StatusCard status={status} />
      <PerformanceChart equityCurve={status?.equity_curve} />
      <TradeHistory trades={status?.recent_trades} />
      <ControlPanel isRunning={isRunning} />
    </div>
  );
}
```

### Performance Visualization

```typescript
// Performance chart component
interface EquityPoint {
  timestamp: string;
  value: number;
  cash: number;
  pnl: number;
}

function PerformanceChart({ equityCurve }: { equityCurve: EquityPoint[] }) {
  const chartData = useMemo(() => {
    return equityCurve?.map(point => ({
      time: new Date(point.timestamp).getTime(),
      value: point.value,
      pnl: point.pnl,
      benchmark: 10000 // Starting value
    }));
  }, [equityCurve]);

  return (
    <div className="performance-chart">
      <LineChart data={chartData}>
        <Line dataKey="value" stroke="#2563eb" name="Portfolio Value" />
        <Line dataKey="benchmark" stroke="#6b7280" name="Buy & Hold" />
        <XAxis dataKey="time" type="number" domain={['dataMin', 'dataMax']} />
        <YAxis />
        <Tooltip labelFormatter={(value) => new Date(value).toLocaleString()} />
        <Legend />
      </LineChart>
    </div>
  );
}
```

### Trade Analysis

```typescript
// Trade history component
interface Trade {
  id: string;
  timestamp: string;
  symbol: string;
  side: 'buy' | 'sell';
  size: number;
  price: number;
  value: number;
  pnl?: number;
  commission: number;
}

function TradeHistory({ trades }: { trades: Trade[] }) {
  const [sortBy, setSortBy] = useState<keyof Trade>('timestamp');
  const [filterSymbol, setFilterSymbol] = useState<string>('');

  const filteredTrades = useMemo(() => {
    let filtered = trades || [];

    if (filterSymbol) {
      filtered = filtered.filter(trade => trade.symbol.includes(filterSymbol));
    }

    return filtered.sort((a, b) => {
      if (sortBy === 'timestamp') {
        return new Date(b.timestamp).getTime() - new Date(a.timestamp).getTime();
      }
      return (b[sortBy] as number) - (a[sortBy] as number);
    });
  }, [trades, sortBy, filterSymbol]);

  return (
    <div className="trade-history">
      <div className="filters">
        <input
          type="text"
          placeholder="Filter by symbol..."
          value={filterSymbol}
          onChange={(e) => setFilterSymbol(e.target.value)}
        />
        <select value={sortBy} onChange={(e) => setSortBy(e.target.value as keyof Trade)}>
          <option value="timestamp">Time</option>
          <option value="pnl">P&L</option>
          <option value="value">Value</option>
        </select>
      </div>

      <table className="trades-table">
        <thead>
          <tr>
            <th>Time</th>
            <th>Symbol</th>
            <th>Side</th>
            <th>Size</th>
            <th>Price</th>
            <th>Value</th>
            <th>P&L</th>
          </tr>
        </thead>
        <tbody>
          {filteredTrades.map(trade => (
            <TradeRow key={trade.id} trade={trade} />
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

## Integration Examples

### Python Client Library

```python
# cracktrader_client.py
import requests
import asyncio
import aiohttp
from typing import Dict, Any, Optional, AsyncGenerator

class CracktraderClient:
    """Python client for Cracktrader REST API."""

    def __init__(self, base_url: str = "http://localhost:8000"):
        self.base_url = base_url.rstrip('/')
        self.api_base = f"{self.base_url}/api/v1"

    # Synchronous methods
    def get_health(self) -> Dict[str, Any]:
        """Get system health status."""
        response = requests.get(f"{self.api_base}/health")
        response.raise_for_status()
        return response.json()

    def get_status(self) -> Dict[str, Any]:
        """Get current runner status."""
        response = requests.get(f"{self.api_base}/status")
        response.raise_for_status()
        return response.json()

    def list_strategies(self) -> Dict[str, Any]:
        """List available strategies."""
        response = requests.get(f"{self.api_base}/strategies")
        response.raise_for_status()
        return response.json()

    def start_backtest(self, config: Dict[str, Any]) -> Dict[str, Any]:
        """Start a backtest."""
        response = requests.post(f"{self.api_base}/run/backtest", json=config)
        response.raise_for_status()
        return response.json()

    def start_live_trading(self, config: Dict[str, Any]) -> Dict[str, Any]:
        """Start live trading."""
        response = requests.post(f"{self.api_base}/run/live", json=config)
        response.raise_for_status()
        return response.json()

    def stop_run(self) -> Dict[str, Any]:
        """Stop current run."""
        response = requests.post(f"{self.api_base}/run/stop")
        response.raise_for_status()
        return response.json()

    def reset_runner(self) -> Dict[str, Any]:
        """Reset runner to idle state."""
        response = requests.delete(f"{self.api_base}/run/reset")
        response.raise_for_status()
        return response.json()

    def get_results(self) -> Dict[str, Any]:
        """Get run results."""
        response = requests.get(f"{self.api_base}/results")
        response.raise_for_status()
        return response.json()

    def wait_for_completion(self, check_interval: float = 2.0) -> Dict[str, Any]:
        """Wait for current run to complete."""
        import time

        while True:
            status = self.get_status()

            if status['status'] in ['completed', 'failed']:
                return status

            time.sleep(check_interval)

    # Context manager for session management
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # Clean up if needed
        pass

# Usage example
def run_strategy_optimization():
    """Example: Optimize strategy parameters."""

    with CracktraderClient() as client:
        # Check system health
        health = client.get_health()
        if health['status'] != 'healthy':
            print(f"System not healthy: {health}")
            return

        # Parameter grid
        periods = [10, 15, 20, 25, 30]
        thresholds = [0.01, 0.02, 0.03]

        best_params = None
        best_return = -float('inf')

        for period in periods:
            for threshold in thresholds:
                config = {
                    "strategy_name": "MovingAverageCross",
                    "strategy_params": {
                        "fast_period": period,
                        "slow_period": period * 2,
                        "threshold": threshold
                    },
                    "symbols": ["BTC/USDT"],
                    "timeframe": "1h",
                    "start_date": "2024-01-01T00:00:00Z",
                    "end_date": "2024-03-01T23:59:59Z",
                    "initial_cash": 10000.0
                }

                print(f"Testing period={period}, threshold={threshold}")

                # Start backtest
                result = client.start_backtest(config)

                # Wait for completion
                final_status = client.wait_for_completion()

                if final_status['status'] == 'completed':
                    results = client.get_results()
                    final_value = results.get('final_value', 0)
                    total_return = (final_value - 10000) / 10000

                    print(f"Return: {total_return:.2%}")

                    if total_return > best_return:
                        best_return = total_return
                        best_params = (period, threshold)
                else:
                    print(f"Backtest failed: {final_status.get('error', 'Unknown error')}")

                # Reset for next run
                client.reset_runner()

        print(f"Best parameters: period={best_params[0]}, threshold={best_params[1]}")
        print(f"Best return: {best_return:.2%}")

if __name__ == "__main__":
    run_strategy_optimization()
```

### JavaScript/Node.js Client

```javascript
// cracktrader-client.js
const axios = require('axios');

class CracktraderClient {
  constructor(baseUrl = 'http://localhost:8000') {
    this.baseUrl = baseUrl.replace(/\/$/, '');
    this.apiBase = `${this.baseUrl}/api/v1`;
    this.client = axios.create({
      baseURL: this.apiBase,
      timeout: 30000
    });
  }

  async getHealth() {
    const response = await this.client.get('/health');
    return response.data;
  }

  async getStatus() {
    const response = await this.client.get('/status');
    return response.data;
  }

  async getDetailedStatus() {
    const response = await this.client.get('/status/detailed');
    return response.data;
  }

  async listStrategies() {
    const response = await this.client.get('/strategies');
    return response.data;
  }

  async startBacktest(config) {
    const response = await this.client.post('/run/backtest', config);
    return response.data;
  }

  async startLiveTrading(config) {
    const response = await this.client.post('/run/live', config);
    return response.data;
  }

  async stopRun() {
    const response = await this.client.post('/run/stop');
    return response.data;
  }

  async resetRunner() {
    const response = await this.client.delete('/run/reset');
    return response.data;
  }

  async getResults() {
    const response = await this.client.get('/results');
    return response.data;
  }

  async waitForCompletion(checkInterval = 2000) {
    return new Promise((resolve, reject) => {
      const check = async () => {
        try {
          const status = await this.getStatus();

          if (status.status === 'completed' || status.status === 'failed') {
            resolve(status);
          } else {
            setTimeout(check, checkInterval);
          }
        } catch (error) {
          reject(error);
        }
      };

      check();
    });
  }

  // Event-based monitoring
  async *monitorRun(checkInterval = 2000) {
    while (true) {
      const status = await this.getDetailedStatus();
      yield status;

      if (status.status === 'completed' || status.status === 'failed') {
        break;
      }

      await new Promise(resolve => setTimeout(resolve, checkInterval));
    }
  }
}

// Usage example
async function runBacktestExample() {
  const client = new CracktraderClient();

  try {
    // Check health
    const health = await client.getHealth();
    console.log('System health:', health.status);

    // Start backtest
    const config = {
      strategy_name: "TestStrategy",
      symbols: ["BTC/USDT"],
      timeframe: "1h",
      initial_cash: 10000
    };

    const result = await client.startBacktest(config);
    console.log('Backtest started:', result.run_id);

    // Monitor progress
    for await (const status of client.monitorRun()) {
      console.log(`Status: ${status.status}`);
      if (status.value) {
        console.log(`Portfolio: $${status.value.toLocaleString()}`);
      }
    }

    // Get final results
    const results = await client.getResults();
    console.log('Final results:', results);

  } catch (error) {
    console.error('Error:', error.message);
  }
}

module.exports = CracktraderClient;

// Run example
if (require.main === module) {
  runBacktestExample();
}
```

### cURL Examples

```bash
# Health check
curl -X GET "http://localhost:8000/api/v1/health"

# Get current status
curl -X GET "http://localhost:8000/api/v1/status"

# List strategies
curl -X GET "http://localhost:8000/api/v1/strategies"

# Start backtest
curl -X POST "http://localhost:8000/api/v1/run/backtest" \
  -H "Content-Type: application/json" \
  -d '{
    "strategy_name": "TestStrategy",
    "symbols": ["BTC/USDT"],
    "timeframe": "1h",
    "initial_cash": 10000
  }'

# Start paper trading
curl -X POST "http://localhost:8000/api/v1/run/live" \
  -H "Content-Type: application/json" \
  -d '{
    "strategy_name": "TestStrategy",
    "symbols": ["BTC/USDT"],
    "timeframe": "5m",
    "mode": "paper",
    "exchange": "binance",
    "exchange_config": {
      "apiKey": "your_key",
      "secret": "your_secret",
      "sandbox": true
    }
  }'

# Stop current run
curl -X POST "http://localhost:8000/api/v1/run/stop"

# Get results
curl -X GET "http://localhost:8000/api/v1/results"

# Reset runner
curl -X DELETE "http://localhost:8000/api/v1/run/reset"
```

## Production Deployment

### Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY src/ ./src/
COPY scripts/ ./scripts/
COPY config.yaml .

# Install package
RUN pip install -e .

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8000/api/v1/health || exit 1

# Start server
CMD ["python", "scripts/run_web_server.py", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  cracktrader-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - CRACKTRADER_LOG_LEVEL=INFO
      - CRACKTRADER_JSON_LOGS=true
    volumes:
      - ./logs:/app/logs
      - ./data:/app/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Optional: Add nginx reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - cracktrader-api
    restart: unless-stopped
```

### Kubernetes Deployment

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cracktrader-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cracktrader-api
  template:
    metadata:
      labels:
        app: cracktrader-api
    spec:
      containers:
      - name: api
        image: cracktrader:latest
        ports:
        - containerPort: 8000
        env:
        - name: CRACKTRADER_LOG_LEVEL
          value: "INFO"
        - name: CRACKTRADER_JSON_LOGS
          value: "true"
        livenessProbe:
          httpGet:
            path: /api/v1/health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /api/v1/health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: cracktrader-service
spec:
  selector:
    app: cracktrader-api
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

### Security Configuration

```yaml
# config.yaml - Production security settings
web:
  host: "0.0.0.0"
  port: 8000
  cors_enabled: true
  cors_origins:
    - "https://yourdomain.com"
    - "https://app.yourdomain.com"

  # Security headers
  security_headers:
    x_frame_options: "DENY"
    x_content_type_options: "nosniff"
    x_xss_protection: "1; mode=block"
    referrer_policy: "strict-origin-when-cross-origin"

  # Rate limiting
  rate_limiting:
    enabled: true
    requests_per_minute: 60
    burst_size: 10

# API authentication (future feature)
api:
  authentication:
    enabled: false  # Will be true in future versions
    api_keys:
      - key: "your_api_key_here"
        permissions: ["read", "write"]
        rate_limit: 120
```

## Troubleshooting

### Common Issues

```bash
# Issue: Server won't start
# Check port availability
netstat -an | grep 8000
lsof -i :8000

# Check logs
python scripts/run_web_server.py --log-level DEBUG

# Issue: API returns 500 errors
# Check application logs
tail -f logs/cracktrader.log

# Check specific error details
curl -v http://localhost:8000/api/v1/health

# Issue: Frontend won't connect to API
# Check CORS configuration
curl -H "Origin: http://localhost:3000" \
     -H "Access-Control-Request-Method: GET" \
     -X OPTIONS \
     http://localhost:8000/api/v1/status
```

### Performance Optimization

```python
# Optimize API response times
import uvicorn
from cracktrader.web.server.main import create_app

# Production ASGI server configuration
if __name__ == "__main__":
    app = create_app()

    uvicorn.run(
        app,
        host="0.0.0.0",
        port=8000,
        workers=4,              # Multiple worker processes
        loop="uvloop",          # Fast event loop
        http="httptools",       # Fast HTTP parser
        access_log=False,       # Disable access logs for performance
        log_level="info"
    )
```

---

**Next Steps:**
- [Broker Integration](brokers.md) - Order execution and management
- [Advanced Configuration](advanced.md) - Performance tuning and scaling
- [Extending Cracktrader](extending.md) - Building custom integrations
