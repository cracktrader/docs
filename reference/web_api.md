# Web API Reference

Complete API reference for CrackTrader's REST API and WebSocket endpoints.

## Authentication

### API Key Authentication

```http
Authorization: Bearer your_api_key
```

### Session Authentication

```http
POST /api/auth/login
Content-Type: application/json

{
  "username": "your_username",
  "password": "your_password"
}
```

## REST Endpoints

### Account Information

#### GET /api/account/balance

Get account balance across all exchanges.

**Response:**
```json
{
  "total_balance": 10000.50,
  "exchanges": {
    "binance": {
      "BTC": 0.5,
      "USDT": 1000.0
    }
  }
}
```

#### GET /api/account/positions

Get current positions.

**Response:**
```json
{
  "positions": [
    {
      "symbol": "BTC/USDT",
      "side": "long",
      "size": 0.1,
      "entry_price": 50000,
      "unrealized_pnl": 500
    }
  ]
}
```

### Market Data

#### GET /api/market/ticker/{symbol}

Get ticker data for a symbol.

**Parameters:**
- `symbol`: Trading pair (e.g., BTC/USDT)

**Response:**
```json
{
  "symbol": "BTC/USDT",
  "last": 50000,
  "bid": 49999,
  "ask": 50001,
  "volume": 1000,
  "change": 2.5
}
```

#### GET /api/market/ohlcv/{symbol}

Get OHLCV data.

**Parameters:**
- `symbol`: Trading pair
- `timeframe`: 1m, 5m, 15m, 1h, 4h, 1d
- `limit`: Number of candles (default: 100)

**Response:**
```json
{
  "data": [
    [1640995200000, 50000, 50100, 49900, 50050, 10.5]
  ]
}
```

### Trading

#### POST /api/trading/order

Place a new order.

**Request:**
```json
{
  "symbol": "BTC/USDT",
  "side": "buy",
  "type": "limit",
  "amount": 0.1,
  "price": 50000
}
```

**Response:**
```json
{
  "order_id": "12345",
  "status": "open",
  "symbol": "BTC/USDT",
  "side": "buy",
  "amount": 0.1,
  "price": 50000,
  "timestamp": 1640995200000
}
```

#### GET /api/trading/orders

Get order history.

**Response:**
```json
{
  "orders": [
    {
      "order_id": "12345",
      "status": "filled",
      "symbol": "BTC/USDT",
      "side": "buy",
      "amount": 0.1,
      "price": 50000,
      "filled": 0.1,
      "timestamp": 1640995200000
    }
  ]
}
```

#### DELETE /api/trading/order/{order_id}

Cancel an order.

**Response:**
```json
{
  "order_id": "12345",
  "status": "cancelled"
}
```

### Strategy Management

#### GET /api/strategies

List running strategies.

**Response:**
```json
{
  "strategies": [
    {
      "id": "strategy_1",
      "name": "Moving Average Strategy",
      "status": "running",
      "pnl": 500.0,
      "positions": 2
    }
  ]
}
```

#### POST /api/strategies/{strategy_id}/start

Start a strategy.

#### POST /api/strategies/{strategy_id}/stop

Stop a strategy.

## WebSocket API

### Connection

```javascript
const ws = new WebSocket('ws://localhost:8000/ws');
```

### Subscriptions

#### Market Data

```json
{
  "action": "subscribe",
  "channel": "ticker",
  "symbol": "BTC/USDT"
}
```

#### Order Updates

```json
{
  "action": "subscribe",
  "channel": "orders",
  "user_id": "your_user_id"
}
```

#### Trade Execution

```json
{
  "action": "subscribe",
  "channel": "trades",
  "symbol": "BTC/USDT"
}
```

### Message Formats

#### Ticker Update

```json
{
  "channel": "ticker",
  "symbol": "BTC/USDT",
  "data": {
    "last": 50000,
    "bid": 49999,
    "ask": 50001,
    "volume": 1000,
    "timestamp": 1640995200000
  }
}
```

#### Order Update

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
