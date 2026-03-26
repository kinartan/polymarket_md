# Polymarket CLOB Orders — Technical Guide

Complete guide to placing orders on Polymarket's Central Limit Order Book (CLOB) using the `@polymarket/clob-client` SDK.

---

## Overview

Polymarket uses a hybrid on-chain/off-chain order book. Orders are signed off-chain and matched by the CLOB server, then settled on-chain (Polygon). The SDK handles signing and submission.

### Order Types

| Type | Name | Behavior |
|------|------|----------|
| **GTC** | Good-Till-Cancelled | Rests on order book until filled, cancelled, or expired |
| **FAK** | Fill-And-Kill | Fill as much as possible immediately, cancel unfilled remainder |
| **FOK** | Fill-Or-Kill | Fill entire order immediately or reject entirely |
| **GTD** | Good-Till-Date | Like GTC but auto-expires at specified timestamp |

### Two Creator Functions

| Function | Used For | Size Param | When |
|----------|----------|-----------|------|
| `createOrder()` | GTC, GTD | `size` (shares) | Limit orders — specify exact share count and price |
| `createMarketOrder()` | FAK, FOK | `amount` (USD for BUY, shares for SELL) | Market orders — fill against existing liquidity |

---

## Setup

```typescript
import { ClobClient, Side, OrderType } from "@polymarket/clob-client";
import { Wallet } from "@ethersproject/wallet";

const signer = new Wallet(PRIVATE_KEY);
const clobClient = new ClobClient(
  "https://clob.polymarket.com",
  137,  // Polygon chain ID
  signer,
  undefined,                    // creds (auto-derived below)
  undefined,                    // signatureType
  FUNDER_ADDRESS                // proxy wallet (if using PROXY mode)
);

// Derive API credentials (required for order placement)
const creds = await clobClient.createOrDeriveApiKey();
await clobClient.setClobClient(creds);
```

### Authentication Modes

| Mode | Description |
|------|-------------|
| **EOA** | Direct wallet signing. `signer` = trading wallet. |
| **PROXY** | Funder wallet delegates to signer. Pass `funderAddress` to constructor. |

---

## Token IDs

Every Polymarket market has two outcome tokens (e.g., "Yes"/"No" or "Up"/"Down"). Each token has a unique ID (large integer string).

Get token IDs from the Gamma API:
```
GET https://gamma-api.polymarket.com/markets?slug={market-slug}
→ response.clobTokenIds = ["token_id_yes", "token_id_no"]
→ response.outcomes = ["Yes", "No"]
```

Token IDs map to outcomes by index: `outcomes[0]` → `clobTokenIds[0]`.

---

## Common Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `tokenID` | string | Outcome token ID (long integer string) |
| `side` | `Side.BUY` \| `Side.SELL` | Order direction |
| `price` | number | Price per share ($0.01 – $0.99, 2 decimals) |
| `size` | number | Share count (for `createOrder`) |
| `amount` | number | USD amount for BUY, share count for SELL (for `createMarketOrder`) |
| `feeRateBps` | number | Fee rate in basis points. `0` = no fee, `1000` = 10% |
| `nonce` | number | Order nonce. Use `0` for auto-generation |
| `expiration` | number | Unix timestamp for GTD orders. `0` = no expiration (GTC) |

### Minimums

| Constraint | Value |
|------------|-------|
| Minimum shares per order | 5 |
| Minimum order cost | $1.00 |
| Minimum price | $0.01 |
| Maximum price | $0.99 |
| Price precision | 2 decimal places (cents) |

---

## BUY Orders

### GTC BUY — Limit Buy (rests on book)

```typescript
// Buy 100 shares of YES at $0.50 each
const order = await clobClient.createOrder({
  tokenID: "71321045679252212594626385532706912750332728571942532289631379312455583992563",
  side: Side.BUY,
  size: 100,        // shares
  price: 0.50,      // price per share
  feeRateBps: 0,
  nonce: 0,
});

const resp = await clobClient.postOrder(order, OrderType.GTC);
```

- Order rests on the book at $0.50
- May partially fill immediately if sell orders exist at ≤ $0.50
- Remaining shares stay on book until filled or cancelled

### FAK BUY — Market Buy (partial fills OK)

```typescript
// Spend $50 buying YES at up to $0.55/share
const order = await clobClient.createMarketOrder({
  tokenID: "71321045679252212594626385532706912750332728571942532289631379312455583992563",
  side: Side.BUY,
  amount: 50,       // USD to spend
  price: 0.55,      // max price willing to pay
  feeRateBps: 0,
  nonce: 0,
});

const resp = await clobClient.postOrder(order, OrderType.FAK);
```

- Fills against existing sell orders at ≤ $0.55
- If only $30 of liquidity available, fills $30 and cancels the remaining $20
- `amount` is in **USD** for BUY orders

### FOK BUY — All-or-Nothing Buy

```typescript
// Buy exactly $50 of YES at max $0.55 — all or nothing
const order = await clobClient.createMarketOrder({
  tokenID: "71321045679252212594626385532706912750332728571942532289631379312455583992563",
  side: Side.BUY,
  amount: 50,
  price: 0.55,
  feeRateBps: 0,
  nonce: 0,
});

const resp = await clobClient.postOrder(order, OrderType.FOK);
```

- Either fills the **entire** $50 or rejects completely
- Use when partial fills would create problems (e.g., balancing operations)

### GTD BUY — Limit Buy with Expiration

```typescript
// Buy 100 YES at $0.50, expires in 5 minutes
// NOTE: Polymarket has a 1-minute security threshold on expiration
// To expire in 5 minutes, set: now + 1min (threshold) + 5min
const expiration = Math.floor((Date.now() + 60_000 + 300_000) / 1000);

const order = await clobClient.createOrder({
  tokenID: "71321045679252212594626385532706912750332728571942532289631379312455583992563",
  side: Side.BUY,
  size: 100,
  price: 0.50,
  feeRateBps: 0,
  nonce: 0,
  expiration: expiration,
});

const resp = await clobClient.postOrder(order, OrderType.GTD);
```

- Same as GTC but auto-cancels at expiration timestamp
- Add 60 seconds to desired expiration for Polymarket's security threshold

---

## SELL Orders

### Token Approval (Required Before First Sell)

Before selling any token for the first time, you must approve the CLOB exchange contract to transfer your conditional tokens:

```typescript
// Check and set approval (one-time per token)
const allowances = await clobClient.getBalanceAllowance({
  asset_type: AssetType.CONDITIONAL,
  token_id: tokenID,
});

if (allowances.allowance === "0") {
  await clobClient.setApproval();  // Approves all conditional tokens
}
```

### FAK SELL — Market Sell (partial fills OK)

```typescript
// Sell 100 shares of YES at min $0.45/share
const order = await clobClient.createMarketOrder({
  tokenID: "71321045679252212594626385532706912750332728571942532289631379312455583992563",
  side: Side.SELL,
  amount: 100,      // SHARES to sell (not USD!)
  price: 0.45,      // minimum price to accept
  feeRateBps: 0,
  nonce: 0,
});

const resp = await clobClient.postOrder(order, OrderType.FAK);
```

- Fills against existing buy orders at ≥ $0.45
- `amount` is in **shares** for SELL orders (unlike BUY where it's USD)
- Use `price: 0.01` to sell at any price (emergency exit)

### GTC SELL — Limit Sell (rests on book)

```typescript
// Sell 100 shares of YES at $0.55 — rests on book
const order = await clobClient.createOrder({
  tokenID: "71321045679252212594626385532706912750332728571942532289631379312455583992563",
  side: Side.SELL,
  size: 100,        // shares
  price: 0.55,      // exact sell price
  feeRateBps: 0,
  nonce: 0,
});

const resp = await clobClient.postOrder(order, OrderType.GTC);
```

- Order rests on book at $0.55
- Fills when someone buys at ≥ $0.55

---

## Response Format

All `postOrder()` calls return:

```typescript
interface PostOrderResponse {
  orderID: string;          // Unique order identifier
  takingAmount?: string;    // What you received (see table below)
  makingAmount?: string;    // What you gave (see table below)
  status?: number;          // HTTP status code
  success?: boolean;
  error?: string;           // Error message
  errorMsg?: string;        // Alternative error field
}
```

### Response Field Semantics

**Critical:** The meaning of `takingAmount` and `makingAmount` depends on the side:

| Field | BUY Order | SELL Order |
|-------|-----------|------------|
| `takingAmount` | Shares received | USD received |
| `makingAmount` | USD spent | Shares sold |

### Parsing Examples

**BUY fill:**
```typescript
const sharesReceived = parseFloat(response.takingAmount);   // e.g., "10" → 10 shares
const usdSpent = parseFloat(response.makingAmount);         // e.g., "5.5" → $5.50
const avgPrice = usdSpent / sharesReceived;                 // $0.55/share
```

**SELL fill:**
```typescript
const usdReceived = parseFloat(response.takingAmount);      // e.g., "4.8" → $4.80
const sharesSold = parseFloat(response.makingAmount);       // e.g., "10" → 10 shares
const avgPrice = usdReceived / sharesSold;                  // $0.48/share
```

### GTC Response Behavior

GTC orders may partially fill on placement (taker portion), with the rest resting:

- `takingAmount`/`makingAmount` reflect only the **immediate** fill
- Remaining resting shares fill later — detected via User Channel WebSocket
- If no immediate fill, `takingAmount`/`makingAmount` may be absent or "0"

---

## Order Management

### Cancel Orders

```typescript
// Cancel a single order
await clobClient.cancelOrder({ orderID: "0x1234..." });

// Cancel multiple orders
await clobClient.cancelOrders(["0x1234...", "0x5678..."]);

// Cancel all orders (nuclear)
await clobClient.cancelAll();
```

### Get Open Orders

```typescript
const openOrders = await clobClient.getOpenOrders();
// Returns array of { id, asset_id, side, price, original_size, size_matched, status, ... }
```

### Get Order Book

```typescript
const book = await clobClient.getOrderBook(tokenID);
// Returns { bids: [{ price, size }], asks: [{ price, size }] }
```

---

## Binary Market Mechanics

In Polymarket binary markets:
- Two tokens: YES and NO (or UP and DOWN)
- Prices always sum to ~$1.00 (YES $0.60 + NO $0.40 = $1.00)
- At settlement: winning token pays $1.00, losing token pays $0.00
- Selling YES is equivalent to someone else buying NO (and vice versa)

### Implied Pricing

```
YES price + NO price ≈ $1.00

If YES ask = $0.60 → NO is worth ~$0.40
If YES drops to $0.30 → NO rises to ~$0.70
```

### Settlement

After market resolution:
- Winning tokens redeemable for $1.00 each
- Losing tokens worth $0.00
- Redeem via Builder Relayer (gas-free) or direct contract call

---

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `"not enough balance / allowance"` | Insufficient USDC (buy) or token approval missing (sell) | Check balance / call `setApproval()` |
| `"Could not create api key"` | Stale API credentials | Re-derive with `createOrDeriveApiKey()` |
| Status `400` | Bad order params (price out of range, size too small) | Validate inputs |
| Status `429` | Rate limited | Back off 1-10 seconds and retry |
| `"No orderID returned"` | Order rejected silently | Check market is open and params are valid |

---

## Rate Limits

- **Order placement:** ~1 order per 100ms sustained
- **Cancellation:** ~1 cancel per 100ms
- **Get requests:** More lenient, ~10/s
- **429 response:** Back off exponentially (1s, 2s, 4s...)

---

## WebSocket Fill Detection

Orders placed via REST are confirmed and filled via the User Channel WebSocket:

```
URL: wss://ws-subscriptions-clob.polymarket.com/ws/user
Keepalive: Send "PING" string every 10 seconds
```

### Event Types

| Event | Status | Meaning |
|-------|--------|---------|
| `order` / `LIVE` | Order placed on book |
| `order` / `MATCHED` | Partial fill (size_matched is cumulative) |
| `order` / `CANCELED` | Order cancelled |
| `trade` / `MATCHED` | Fill confirmed by matching engine |
| `trade` / `MINED` | Transaction in block (duplicate — ignore) |
| `trade` / `CONFIRMED` | On-chain confirmed (duplicate — ignore) |

**Best practice:** Use `trade` events with `status: "MATCHED"` for fill detection. They arrive faster than REST responses.

---

## References

- [Polymarket CLOB Client SDK](https://github.com/Polymarket/clob-client)
- [Polymarket CLOB API Docs](https://docs.polymarket.com/)
- CLOB endpoint: `https://clob.polymarket.com`
- Chain: Polygon (137)

---

*Last Updated: 2026-03-26*
