# Hive Engine (Layer 2 Tokens)

Hive Engine is a sidechain for custom tokens, NFTs, and a decentralized exchange built on top of Hive.

## API Endpoints

| Service | URL | Purpose |
|---------|-----|---------|
| Main RPC | `https://api.hive-engine.com/rpc` | JSON-RPC queries |
| Alternative | `https://engine.rishipanthee.com/rpc` | Backup RPC |
| History | `https://history.hive-engine.com` | Transaction history |
| Contracts | `https://api.hive-engine.com/rpc/contracts` | Contract queries |

## Querying Token Data

### Get Token Balances

```typescript
async function getTokenBalances(account: string) {
  const response = await fetch('https://api.hive-engine.com/rpc/contracts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 1,
      method: 'find',
      params: {
        contract: 'tokens',
        table: 'balances',
        query: { account },
        limit: 1000,
      },
    }),
  });

  const { result } = await response.json();
  return result; // [{ account, symbol, balance, stake, ... }]
}
```

### Get Token Info

```typescript
async function getTokenInfo(symbol: string) {
  const response = await fetch('https://api.hive-engine.com/rpc/contracts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 1,
      method: 'findOne',
      params: {
        contract: 'tokens',
        table: 'tokens',
        query: { symbol },
      },
    }),
  });

  const { result } = await response.json();
  return result; // { symbol, name, precision, maxSupply, supply, ... }
}
```

### Get Order Book

```typescript
async function getOrderBook(symbol: string, limit = 50) {
  const [buyOrders, sellOrders] = await Promise.all([
    queryContract('market', 'buyBook', { symbol }, limit, 0, [{ index: 'priceDec', descending: true }]),
    queryContract('market', 'sellBook', { symbol }, limit, 0, [{ index: 'priceDec', descending: false }]),
  ]);

  return { buy: buyOrders, sell: sellOrders };
}
```

### Get Trade History

```typescript
async function getTradeHistory(symbol: string, limit = 50) {
  return queryContract('market', 'tradesHistory', { symbol }, limit, 0, [
    { index: '_id', descending: true }
  ]);
}
```

### Get Open Orders

```typescript
async function getOpenOrders(account: string, symbol?: string) {
  const query: any = { account };
  if (symbol) query.symbol = symbol;

  const [buyOrders, sellOrders] = await Promise.all([
    queryContract('market', 'buyBook', query, 100),
    queryContract('market', 'sellBook', query, 100),
  ]);

  return { buy: buyOrders, sell: sellOrders };
}
```

## Broadcasting Hive Engine Operations

All Hive Engine operations are broadcast as custom_json on the Hive chain:

```typescript
const SIDECHAIN_ID = 'ssc-mainnet-hive';

async function broadcastHiveEngineOp(username: string, contractName: string, contractAction: string, payload: any, activeKey: any) {
  await client.broadcast.json({
    required_auths: [username],        // Active authority for financial ops
    required_posting_auths: [],
    id: SIDECHAIN_ID,
    json: JSON.stringify({
      contractName,
      contractAction,
      contractPayload: payload,
    }),
  }, activeKey);
}
```

### Token Transfer

```typescript
await broadcastHiveEngineOp(username, 'tokens', 'transfer', {
  symbol: 'BEE',
  to: 'recipient',
  quantity: '10.000',
  memo: 'Payment',
}, activeKey);
```

### Stake Tokens

```typescript
await broadcastHiveEngineOp(username, 'tokens', 'stake', {
  to: username,   // Can stake to another account
  symbol: 'BEE',
  quantity: '100',
}, activeKey);
```

### Unstake Tokens

```typescript
await broadcastHiveEngineOp(username, 'tokens', 'unstake', {
  symbol: 'BEE',
  quantity: '50',
}, activeKey);
```

### Delegate Tokens

```typescript
await broadcastHiveEngineOp(username, 'tokens', 'delegate', {
  to: 'delegatee',
  symbol: 'BEE',
  quantity: '100',
}, activeKey);

// Undelegate
await broadcastHiveEngineOp(username, 'tokens', 'undelegate', {
  from: 'delegatee',
  symbol: 'BEE',
  quantity: '100',
}, activeKey);
```

### Market Buy Order

```typescript
await broadcastHiveEngineOp(username, 'market', 'buy', {
  symbol: 'BEE',
  quantity: '10',     // Amount of tokens to buy
  price: '0.5',       // Price per token in SWAP.HIVE
}, activeKey);
```

### Market Sell Order

```typescript
await broadcastHiveEngineOp(username, 'market', 'sell', {
  symbol: 'BEE',
  quantity: '10',
  price: '0.6',
}, activeKey);
```

### Cancel Order

```typescript
await broadcastHiveEngineOp(username, 'market', 'cancel', {
  type: 'buy',  // or 'sell'
  id: 'order-id',
}, activeKey);
```

## NFT Operations

### Query NFTs

```typescript
// Get NFT instances owned by account
async function getNFTs(account: string, symbol: string) {
  return queryContract('nft', 'instances', {
    account,
    'properties.symbol': symbol,
  }, 1000);
}

// Get NFT definition
async function getNFTDefinition(symbol: string) {
  return queryContract('nft', 'nfts', { symbol });
}
```

### Transfer NFT

```typescript
await broadcastHiveEngineOp(username, 'nft', 'transfer', {
  to: 'recipient',
  nfts: [{ symbol: 'MYNFT', ids: ['1', '2', '3'] }],
}, activeKey);
```

## Diesel Pools (Liquidity Pools)

```typescript
// Get pool info
const pools = await queryContract('marketpools', 'pools', {}, 1000);

// Add liquidity
await broadcastHiveEngineOp(username, 'marketpools', 'addLiquidity', {
  tokenPair: 'BEE:SWAP.HIVE',
  baseQuantity: '100',
  quoteQuantity: '50',
}, activeKey);

// Swap tokens
await broadcastHiveEngineOp(username, 'marketpools', 'swapTokens', {
  tokenPair: 'BEE:SWAP.HIVE',
  tokenSymbol: 'BEE',        // Token you're sending
  tokenAmount: '10',
  tradeType: 'exactInput',   // or 'exactOutput'
  minAmountOut: '4.5',       // Slippage protection
}, activeKey);
```

## Helper: Generic Contract Query

```typescript
async function queryContract(
  contract: string,
  table: string,
  query: any,
  limit = 1000,
  offset = 0,
  indexes?: Array<{ index: string; descending: boolean }>
) {
  const response = await fetch('https://api.hive-engine.com/rpc/contracts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 1,
      method: 'find',
      params: { contract, table, query, limit, offset, indexes },
    }),
  });

  const { result } = await response.json();
  return result || [];
}
```
