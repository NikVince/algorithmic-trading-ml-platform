# API Reference

## 🚀 Overview

The Algorithmic Trading ML Platform provides a comprehensive REST API for model inference, system monitoring, and trading operations. All endpoints are built with FastAPI and include automatic OpenAPI documentation.

## 📡 Base URL

```
Development: http://localhost:8000
Production: https://api.trading-platform.com
```

## 🔐 Authentication

The API uses API key authentication for all endpoints except health checks.

### Authentication Header
```http
Authorization: Bearer <your-api-key>
```

### Getting API Keys
Contact the system administrator to obtain API keys with appropriate permissions.

## 📊 Core Endpoints

### Health & Status

#### GET /health
Check system health and component status.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-10-25T10:30:00Z",
  "components": {
    "database": "healthy",
    "redis": "healthy",
    "models": "healthy"
  },
  "version": "1.0.0"
}
```

#### GET /metrics
Get Prometheus metrics for system monitoring.

**Response:** Prometheus-formatted metrics

### Model Inference

#### POST /predict
Generate trading signal predictions for a given symbol and timeframe.

**Request Body:**
```json
{
  "symbol": "BTC/USDT",
  "timestamp": "2025-10-25T10:30:00Z",
  "features": {
    "price": 45000.0,
    "volume": 1000000.0,
    "rsi": 65.5
  },
  "model_version": "latest"
}
```

**Response:**
```json
{
  "prediction": {
    "signal": "BUY",
    "confidence": 0.85,
    "probability": 0.78
  },
  "model_info": {
    "version": "v1.2.3",
    "training_date": "2025-10-20T00:00:00Z",
    "accuracy": 0.72
  },
  "timestamp": "2025-10-25T10:30:01Z",
  "request_id": "req_123456789"
}
```

#### POST /predict/batch
Generate predictions for multiple symbols simultaneously.

**Request Body:**
```json
{
  "requests": [
    {
      "symbol": "BTC/USDT",
      "timestamp": "2025-10-25T10:30:00Z"
    },
    {
      "symbol": "ETH/USDT", 
      "timestamp": "2025-10-25T10:30:00Z"
    }
  ],
  "model_version": "latest"
}
```

**Response:**
```json
{
  "predictions": [
    {
      "symbol": "BTC/USDT",
      "prediction": {
        "signal": "BUY",
        "confidence": 0.85
      }
    },
    {
      "symbol": "ETH/USDT",
      "prediction": {
        "signal": "SELL",
        "confidence": 0.72
      }
    }
  ],
  "timestamp": "2025-10-25T10:30:01Z"
}
```

### Model Management

#### GET /models
List available models and their metadata.

**Response:**
```json
{
  "models": [
    {
      "name": "lstm_momentum_v1",
      "version": "1.2.3",
      "status": "active",
      "accuracy": 0.72,
      "training_date": "2025-10-20T00:00:00Z",
      "features": ["price", "volume", "rsi", "macd"]
    }
  ]
}
```

#### GET /models/{model_name}/versions
Get version history for a specific model.

**Response:**
```json
{
  "model_name": "lstm_momentum_v1",
  "versions": [
    {
      "version": "1.2.3",
      "status": "active",
      "accuracy": 0.72,
      "training_date": "2024-01-10T00:00:00Z"
    },
    {
      "version": "1.2.2",
      "status": "deprecated",
      "accuracy": 0.68,
      "training_date": "2024-01-05T00:00:00Z"
    }
  ]
}
```

### Feature Engineering

#### GET /features/{symbol}
Get latest features for a symbol.

**Response:**
```json
{
  "symbol": "BTC/USDT",
  "timestamp": "2025-10-25T10:30:00Z",
  "features": {
    "technical": {
      "rsi": 65.5,
      "macd": 125.3,
      "bollinger_upper": 46000.0,
      "bollinger_lower": 44000.0
    },
    "market_regime": {
      "volatility_regime": "high",
      "trend_direction": "upward"
    },
    "cross_asset": {
      "btc_dominance": 0.45,
      "market_correlation": 0.78
    }
  }
}
```

#### POST /features/compute
Trigger feature computation for a symbol.

**Request Body:**
```json
{
  "symbol": "BTC/USDT",
  "force_refresh": false
}
```

**Response:**
```json
{
  "status": "success",
  "symbol": "BTC/USDT",
  "computation_time": 0.045,
  "features_count": 47,
  "timestamp": "2025-10-25T10:30:01Z"
}
```

### Trading Operations

#### GET /strategies
List available trading strategies.

**Response:**
```json
{
  "strategies": [
    {
      "name": "momentum_v1",
      "status": "active",
      "description": "Dual momentum strategy",
      "risk_level": "medium",
      "expected_return": 0.15
    },
    {
      "name": "ml_ensemble_v1",
      "status": "active", 
      "description": "ML-based ensemble strategy",
      "risk_level": "high",
      "expected_return": 0.25
    }
  ]
}
```

#### POST /strategies/{strategy_name}/signal
Get trading signal from a specific strategy.

**Request Body:**
```json
{
  "symbol": "BTC/USDT",
  "timestamp": "2025-10-25T10:30:00Z",
  "position_size": 1000.0
}
```

**Response:**
```json
{
  "strategy": "momentum_v1",
  "signal": {
    "action": "BUY",
    "size": 0.1,
    "confidence": 0.82,
    "stop_loss": 42000.0,
    "take_profit": 48000.0
  },
  "risk_metrics": {
    "var_95": 0.05,
    "max_drawdown": 0.12,
    "sharpe_ratio": 1.8
  },
  "timestamp": "2025-10-25T10:30:01Z"
}
```

### Backtesting

#### POST /backtest
Run backtest for a strategy and time period.

**Request Body:**
```json
{
  "strategy": "momentum_v1",
  "symbol": "BTC/USDT",
  "start_date": "2024-01-01T00:00:00Z",
  "end_date": "2024-12-31T23:59:59Z",
  "initial_capital": 10000.0,
  "commission": 0.001
}
```

**Response:**
```json
{
  "backtest_id": "bt_123456789",
  "status": "completed",
  "results": {
    "total_return": 0.25,
    "sharpe_ratio": 1.8,
    "max_drawdown": 0.12,
    "win_rate": 0.65,
    "total_trades": 156,
    "avg_trade_duration": "2.5 days"
  },
  "execution_time": 45.2,
  "timestamp": "2025-10-25T10:30:01Z"
}
```

#### GET /backtest/{backtest_id}
Get detailed results for a specific backtest.

**Response:**
```json
{
  "backtest_id": "bt_123456789",
  "status": "completed",
  "strategy": "momentum_v1",
  "symbol": "BTC/USDT",
  "period": {
    "start": "2024-01-01T00:00:00Z",
    "end": "2024-12-31T23:59:59Z"
  },
  "performance": {
    "total_return": 0.25,
    "annualized_return": 0.28,
    "volatility": 0.18,
    "sharpe_ratio": 1.8,
    "max_drawdown": 0.12,
    "calmar_ratio": 2.3
  },
  "trading_stats": {
    "total_trades": 156,
    "winning_trades": 101,
    "losing_trades": 55,
    "win_rate": 0.65,
    "avg_win": 0.08,
    "avg_loss": -0.04,
    "profit_factor": 2.1
  },
  "risk_metrics": {
    "var_95": 0.05,
    "cvar_95": 0.08,
    "sortino_ratio": 2.1,
    "treynor_ratio": 0.15
  }
}
```

## 📈 Data Endpoints

### Market Data

#### GET /market-data/{symbol}
Get latest market data for a symbol.

**Response:**
```json
{
  "symbol": "BTC/USDT",
  "timestamp": "2025-10-25T10:30:00Z",
  "price": 45000.0,
  "volume": 1000000.0,
  "high_24h": 46000.0,
  "low_24h": 44000.0,
  "change_24h": 0.02,
  "change_percent_24h": 2.0
}
```

#### GET /market-data/{symbol}/history
Get historical market data.

**Query Parameters:**
- `start_date`: Start date (ISO 8601)
- `end_date`: End date (ISO 8601)
- `interval`: Data interval (1m, 5m, 15m, 1h, 4h, 1d)
- `limit`: Maximum number of records (default: 1000)

**Response:**
```json
{
  "symbol": "BTC/USDT",
  "interval": "1h",
  "data": [
    {
      "timestamp": "2025-10-25T10:00:00Z",
      "open": 44800.0,
      "high": 45200.0,
      "low": 44600.0,
      "close": 45000.0,
      "volume": 1000000.0
    }
  ],
  "count": 1000
}
```

## 🔧 System Endpoints

### Configuration

#### GET /config
Get system configuration (non-sensitive).

**Response:**
```json
{
  "version": "1.0.0",
  "environment": "production",
  "features": {
    "real_time_trading": true,
    "advanced_analytics": true,
    "multi_exchange": true
  },
  "limits": {
    "max_batch_size": 100,
    "max_history_days": 365,
    "rate_limit_per_minute": 1000
  }
}
```

### Logs

#### GET /logs
Get system logs with filtering.

**Query Parameters:**
- `level`: Log level (DEBUG, INFO, WARNING, ERROR)
- `component`: Component name
- `start_time`: Start time (ISO 8601)
- `end_time`: End time (ISO 8601)
- `limit`: Maximum number of records

**Response:**
```json
{
  "logs": [
    {
      "timestamp": "2025-10-25T10:30:00Z",
      "level": "INFO",
      "component": "inference",
      "message": "Prediction completed successfully",
      "request_id": "req_123456789"
    }
  ],
  "count": 100,
  "total": 10000
}
```

## 🚨 Error Handling

### Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "field": "symbol",
      "issue": "Symbol format is invalid"
    }
  },
  "request_id": "req_123456789",
  "timestamp": "2025-10-25T10:30:00Z"
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Request validation failed |
| `AUTHENTICATION_ERROR` | 401 | Invalid or missing API key |
| `AUTHORIZATION_ERROR` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `MODEL_UNAVAILABLE` | 503 | Model not available |
| `SYSTEM_ERROR` | 500 | Internal server error |

## 🔒 Rate Limiting

### Limits
- **General API**: 1000 requests per minute
- **Prediction endpoints**: 100 requests per minute
- **Batch operations**: 10 requests per minute

### Rate Limit Headers
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642248600
```

## 📚 SDK Examples

### Python SDK
```python
from trading_platform import TradingPlatformClient

client = TradingPlatformClient(api_key="your-api-key")

# Get prediction
prediction = client.predict(
    symbol="BTC/USDT",
    timestamp="2025-10-25T10:30:00Z"
)

# Run backtest
backtest = client.backtest(
    strategy="momentum_v1",
    symbol="BTC/USDT",
    start_date="2023-01-01",
    end_date="2023-12-31"
)
```

### JavaScript SDK
```javascript
import { TradingPlatformClient } from '@trading-platform/sdk';

const client = new TradingPlatformClient({
  apiKey: 'your-api-key',
  baseUrl: 'https://api.trading-platform.com'
});

// Get prediction
const prediction = await client.predict({
  symbol: 'BTC/USDT',
  timestamp: '2025-10-25T10:30:00Z'
});
```

## 🔄 WebSocket API

### Connection
```javascript
const ws = new WebSocket('wss://api.trading-platform.com/ws');

ws.onopen = () => {
  // Subscribe to real-time predictions
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'predictions',
    symbol: 'BTC/USDT'
  }));
};
```

### Message Format
```json
{
  "type": "prediction",
  "data": {
    "symbol": "BTC/USDT",
    "prediction": {
      "signal": "BUY",
      "confidence": 0.85
    },
    "timestamp": "2025-10-25T10:30:00Z"
  }
}
```

## 📖 OpenAPI Documentation

Interactive API documentation is available at:
- **Development**: http://localhost:8000/docs
- **Production**: https://api.trading-platform.com/docs

The OpenAPI specification is also available at `/openapi.json`.
