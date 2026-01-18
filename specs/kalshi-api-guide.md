# Kalshi API Guide

## Base URLs

| Environment | REST API | WebSocket |
|-------------|----------|-----------|
| Production | `https://api.elections.kalshi.com/trade-api/v2` | `wss://api.elections.kalshi.com/trade-api/ws/v2` |
| Demo | `https://demo-api.kalshi.co/trade-api/v2` | `wss://demo-api.kalshi.co/trade-api/ws/v2` |

Demo credentials are separate from production.

## Authentication

### Generate API Keys
1. Go to https://kalshi.com/account/profile → API Keys
2. Click "Create New API Key"
3. Save the private key immediately (shown only once)

### Required Headers
```
KALSHI-ACCESS-KEY: {key_id}
KALSHI-ACCESS-TIMESTAMP: {timestamp_ms}
KALSHI-ACCESS-SIGNATURE: {signature}
```

### Signature Generation
Sign: `{timestamp_ms}{METHOD}{path}` (path without query params)

```python
import time, base64
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

def sign(private_key_pem, method, path):
    key = serialization.load_pem_private_key(private_key_pem.encode(), password=None)
    ts = str(int(time.time() * 1000))
    msg = f"{ts}{method.upper()}{path}"
    sig = key.sign(msg.encode(), padding.PSS(mgf=padding.MGF1(hashes.SHA256()),
                   salt_length=padding.PSS.MAX_LENGTH), hashes.SHA256())
    return ts, base64.b64encode(sig).decode()
```

## Rate Limits

| Tier | Read/sec | Write/sec | Qualification |
|------|----------|-----------|---------------|
| Basic | 20 | 10 | Default |
| Advanced | 30 | 30 | Application form |
| Premier | 100 | 100 | 3.75% monthly volume |
| Prime | 400 | 400 | 7.5% monthly volume |

Write limits apply to order endpoints. Batch items count as 1 transaction each (except BatchCancel = 0.2 each).

## Key Endpoints

### Public (No Auth)
- `GET /markets` - List markets
- `GET /markets/{ticker}` - Market details
- `GET /markets/{ticker}/orderbook` - Orderbook
- `GET /events` - List events

### Orders (Auth Required)
- `POST /portfolio/orders` - Create order
- `DELETE /portfolio/orders/{id}` - Cancel order
- `PUT /portfolio/orders/{id}` - Amend order
- `POST /portfolio/orders/batch` - Batch create (max 20)

### Portfolio (Auth Required)
- `GET /portfolio/balance` - Account balance (cents)
- `GET /portfolio/positions` - Your positions
- `GET /portfolio/fills` - Your fills

## Creating Orders

```json
POST /portfolio/orders
{
  "ticker": "MARKET-TICKER",
  "action": "buy",
  "side": "yes",
  "count": 1,
  "type": "limit",
  "yes_price": 30,
  "client_order_id": "unique-uuid"
}
```

- `action`: `buy` or `sell`
- `side`: `yes` or `no`
- `yes_price`: cents (1-99)
- `client_order_id`: UUID for deduplication (resubmit same ID on network failure)

## WebSocket

Subscribe after connecting:
```json
{"id": 1, "cmd": "subscribe", "params": {"channels": ["orderbook_delta"], "market_ticker": "TICKER"}}
```

**Channels:**
- `orderbook_delta` - Orderbook updates (public)
- `ticker` - Price/volume updates (public)
- `trade` - Public trades (public)
- `fill` - Your fills (auth required)
- `market_positions` - Your positions (auth required)

Respond to ping frames every 10 seconds with pong to stay connected.

## Pagination

Cursor-based. Check for `cursor` in response, append `?cursor={value}` to next request until null.

## Pricing

- Standard: cents (1-99)
- Subpenny: `"price_dollars": "0.1234"` (4 decimal places)
- Some WebSocket values in centi-cents (divide by 10000 for dollars)
