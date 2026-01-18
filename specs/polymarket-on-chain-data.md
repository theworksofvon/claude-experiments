# Polymarket On-Chain Data Guidelines

## Overview

Polymarket operates a hybrid-decentralized prediction market on Polygon. The system uses an off-chain operator for order matching while settlement executes on-chain through non-custodial signed messages. This guide covers how to access and interact with Polymarket's on-chain data.

## Architecture

### Data Model Hierarchy

- **Events**: Top-level questions (e.g., "Will X happen?")
- **Markets**: Specific tradable binary outcomes within events
- **Outcomes & Prices**: Arrays mapping outcomes to implied probabilities

```json
{
  "outcomes": ["Yes", "No"],
  "outcomePrices": ["0.20", "0.80"]
}
```

### Smart Contract Addresses (Polygon Mainnet)

| Contract | Address |
|----------|---------|
| USDCe (Collateral) | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` |
| CTF (Conditional Tokens) | `0x4d97dcd97ec945f40cf65f87097ace5ea0476045` |
| CTF Exchange | `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` |
| Neg Risk CTF Exchange | `0xC5d563A36AE78145C45a50134d48A1215220f80a` |
| Neg Risk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` |

---

## REST APIs

### Gamma API (Market Discovery)

Base URL: `https://gamma-api.polymarket.com`

**No authentication required** for read operations.

#### Fetch Active Events
```bash
curl "https://gamma-api.polymarket.com/events?active=true&closed=false&limit=5"
```

#### Fetch Sports Events
```bash
curl "https://gamma-api.polymarket.com/sports"
curl "https://gamma-api.polymarket.com/events?series_id=10345&active=true&closed=false"
```

#### Fetch Events by Tag
```bash
curl "https://gamma-api.polymarket.com/tags?limit=100"
curl "https://gamma-api.polymarket.com/events?tag_id=2&active=true&closed=false"
```

#### Fetch Market Details
```bash
curl "https://gamma-api.polymarket.com/markets?slug=will-bitcoin-reach-100k-by-2025"
```

### CLOB API (Trading & Pricing)

Base URL: `https://clob.polymarket.com`

#### Get Current Price
```bash
curl "https://clob.polymarket.com/price?token_id=YOUR_TOKEN_ID&side=buy"
```
Response: `{"price": "0.65"}`

#### Get Orderbook
```bash
curl "https://clob.polymarket.com/book?token_id=YOUR_TOKEN_ID"
```

#### Get Midpoint Price
```bash
curl "https://clob.polymarket.com/midpoint?token_id=YOUR_TOKEN_ID"
```

### Data API (User Data)

Base URL: `https://data-api.polymarket.com`

- `GET /positions` - User positions
- `GET /activity` - User activity
- `GET /trades` - Trade history

---

## WebSocket (Real-Time Data)

### Endpoints

- **CLOB WebSocket**: `wss://ws-subscriptions-clob.polymarket.com/ws/`
- **Live Data (RTDS)**: `wss://ws-live-data.polymarket.com`

### Channels

| Channel | Purpose |
|---------|---------|
| `market` | Orderbook and price updates (public) |
| `user` | Order status updates (authenticated) |

### Subscription Message

```json
{
  "auth": { /* Auth credentials for user channel */ },
  "markets": ["condition_id_1", "condition_id_2"],
  "assets_ids": ["token_id_1", "token_id_2"],
  "type": "MARKET"
}
```

### Dynamic Subscribe/Unsubscribe

```json
{
  "assets_ids": ["token_id_1"],
  "operation": "subscribe"
}
```

---

## Subgraphs (On-Chain GraphQL Queries)

Polymarket provides hosted subgraphs via Goldsky for direct blockchain state queries.

### Available Subgraphs

| Subgraph | Endpoint |
|----------|----------|
| Orders | `https://api.goldsky.com/api/public/project_cl6mb8i9h0003e201j6li0diw/subgraphs/orderbook-subgraph/0.0.1/gn` |
| Positions | `https://api.goldsky.com/api/public/project_cl6mb8i9h0003e201j6li0diw/subgraphs/positions-subgraph/0.0.7/gn` |
| Activity | `https://api.goldsky.com/api/public/project_cl6mb8i9h0003e201j6li0diw/subgraphs/activity-subgraph/0.0.4/gn` |
| Open Interest | `https://api.goldsky.com/api/public/project_cl6mb8i9h0003e201j6li0diw/subgraphs/oi-subgraph/0.0.6/gn` |
| PNL | `https://api.goldsky.com/api/public/project_cl6mb8i9h0003e201j6li0diw/subgraphs/pnl-subgraph/0.0.14/gn` |

### Example GraphQL Query

```graphql
{
  positions(first: 10) {
    id
    user
    tokenId
    balance
  }
}
```

Source code: [github.com/Polymarket/polymarket-subgraph](https://github.com/Polymarket/polymarket-subgraph)

---

## SDKs

### TypeScript/JavaScript

```bash
npm install @polymarket/clob-client ethers
```

```typescript
import { ClobClient } from "@polymarket/clob-client";
import { Wallet } from "ethers";

const HOST = "https://clob.polymarket.com";
const CHAIN_ID = 137; // Polygon mainnet
const signer = new Wallet(process.env.PRIVATE_KEY);

// Create client and derive API credentials
const tempClient = new ClobClient(HOST, CHAIN_ID, signer);
const apiCreds = await tempClient.createOrDeriveApiKey();

// Initialize authenticated client
const signatureType = 0; // 0=EOA, 1=POLY_PROXY, 2=GNOSIS_SAFE
const client = new ClobClient(HOST, CHAIN_ID, signer, apiCreds, signatureType);

// Fetch market data
const price = await client.getPrice("TOKEN_ID", "buy");
const book = await client.getOrderBook("TOKEN_ID");

// Place an order
const response = await client.createAndPostOrder({
  tokenID: "YOUR_TOKEN_ID",
  price: 0.65,
  size: 10,
  side: "BUY",
});

// Check orders
const openOrders = await client.getOpenOrders();
const trades = await client.getTrades();
```

### Python

```bash
pip install py-clob-client
```

---

## Authentication

Polymarket uses a two-tier authentication system:

### L1 Authentication (Private Key)

For proving wallet ownership. Uses EIP-712 signed messages.

**Required Headers:**
- `POLY_ADDRESS`: Signer's Polygon address
- `POLY_SIGNATURE`: EIP-712 signature
- `POLY_TIMESTAMP`: Current UNIX timestamp
- `POLY_NONCE`: Default is 0

### L2 Authentication (API Credentials)

For authenticated API requests. Derived from L1.

**Credentials:**
- `apiKey`: UUID identifier
- `secret`: Base64-encoded string
- `passphrase`: Random string

**Required Headers:**
- `POLY_ADDRESS`: Signer address
- `POLY_SIGNATURE`: HMAC-SHA256 signature
- `POLY_TIMESTAMP`: Current UNIX timestamp
- `POLY_API_KEY`: API key
- `POLY_PASSPHRASE`: Passphrase

### Signature Types

| Type | Value | Use Case |
|------|-------|----------|
| EOA | 0 | Standard Ethereum wallets |
| POLY_PROXY | 1 | Magic Link users |
| GNOSIS_SAFE | 2 | Multisig wallets |

**Important:** Never commit private keys to version control. Use environment variables.

---

## Exchange Mechanics

### Order Settlement

The CTF Exchange enables atomic swaps between:
- Binary outcome tokens (ERC1155)
- Collateral assets (ERC20 - USDCe)

### Matching Scenarios

| Type | Description |
|------|-------------|
| NORMAL | Direct token swaps between outcome tokens and collateral |
| MINT | Creates complementary token pairs from collateral |
| MERGE | Consolidates complementary pairs back into collateral |

### Fee Structure

```
feeQuote = baseRate × min(price, 1-price) × size
```

Current rates: 0 bps for both maker and taker.

---

## Quick Reference

### Fetching Market Data (No Auth Required)

```bash
# 1. Get active events
curl "https://gamma-api.polymarket.com/events?active=true&closed=false&limit=10"

# 2. Get market details (extract clobTokenIds)
curl "https://gamma-api.polymarket.com/markets?slug=YOUR_MARKET_SLUG"

# 3. Get current price using token ID
curl "https://clob.polymarket.com/price?token_id=YOUR_TOKEN_ID&side=buy"

# 4. Get full orderbook
curl "https://clob.polymarket.com/book?token_id=YOUR_TOKEN_ID"
```

### Key Tips

1. **Always filter markets**: Use `active=true&closed=false` for tradable markets
2. **Token IDs are essential**: Extract from market details for pricing queries
3. **Public data needs no auth**: Market data, prices, and orderbooks are freely accessible
4. **Use subgraphs for historical data**: GraphQL queries for on-chain state and history

---

## Resources

- Exchange Contract Source: [github.com/Polymarket/ctf-exchange](https://github.com/Polymarket/ctf-exchange/tree/main/src)
- Subgraph Source: [github.com/Polymarket/polymarket-subgraph](https://github.com/Polymarket/polymarket-subgraph)
- Official Docs: [docs.polymarket.com](https://docs.polymarket.com)
