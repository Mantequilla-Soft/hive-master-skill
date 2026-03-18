# Hive Engine (Layer 2 Tokens)

Hive Engine is a sidechain for custom tokens, NFTs, and a decentralized exchange built on top of Hive.

## API Endpoints

| Service | URL | Purpose |
|---------|-----|---------|
| Main | `https://api.hive-engine.com` | Primary node |
| Alternative | `https://engine.hive.pizza` | Backup node |
| History | `https://history.hive-engine.com` | Transaction history |
| Contracts | `https://api.hive-engine.com/contracts` | Contract queries |

### Node Failover Strategy

Try the two well-known nodes first. Only fetch from **FlowerEngine** (the on-chain, community-maintained node list) if both fail. This avoids an extra Hive API call on every request while still recovering gracefully from outages.

The `@flowerengine` account's `json_metadata` contains a `nodes` array of active base URLs and a `failing_nodes` object mapping dead URLs to their failure reasons.

```typescript
import { Client } from '@hiveio/dhive';

const FALLBACK_NODES = [
  'https://api.hive-engine.com',
  'https://engine.hive.pizza',
];

const hiveClient = new Client(['https://api.hive.blog']);

// Module-level state — persists for the session, rotates on failure
let nodeIndex = 0;
let activeContractsUrl: string | null = null;
let flowerengineNodes: string[] = [];

async function getHiveEngineNodes(): Promise<string[]> {
  const [account] = await hiveClient.database.getAccounts(['flowerengine']);
  if (!account?.json_metadata) throw new Error('Could not fetch flowerengine account');

  const { nodes, failing_nodes = {} } = JSON.parse(account.json_metadata);
  if (!Array.isArray(nodes) || nodes.length === 0) throw new Error('No nodes in flowerengine metadata');

  return nodes.filter((n: string) => !(n in failing_nodes));
}

function nodeListWithFallback(): string[] {
  // Hardcoded nodes first, then any flowerengine nodes appended behind them
  return [...FALLBACK_NODES, ...flowerengineNodes.filter(n => !FALLBACK_NODES.includes(n))];
}

// Returns cached node immediately. On invalidation, advances index and rotates.
// If we've exhausted all known nodes, fetches a fresh list from flowerengine.
async function getEngineContractsUrl(invalidate = false): Promise<string> {
  if (activeContractsUrl && !invalidate) return activeContractsUrl;

  let nodes = nodeListWithFallback();

  if (invalidate) {
    nodeIndex++;
    // Exhausted all known nodes — fetch fresh list from flowerengine
    if (nodeIndex >= nodes.length) {
      flowerengineNodes = await getHiveEngineNodes();
      nodeIndex = 0;
      nodes = nodeListWithFallback();
    }
  }

  activeContractsUrl = `${nodes[nodeIndex]}/contracts`;
  return activeContractsUrl;
}
```

No ping on startup — the first real query drives failover. If it fails, pass `invalidate=true` to advance to the next node. Only hits flowerengine once all known nodes are exhausted.

## Querying Token Data

### Get Token Balances

```typescript
async function getTokenBalances(account: string) {
  return queryContractWithFailover('tokens', 'balances', { account }, 1000);
  // Returns [{ account, symbol, balance, stake, ... }]
}
```

### Get Token Info

```typescript
async function getTokenInfo(symbol: string) {
  const results = await queryContractWithFailover('tokens', 'tokens', { symbol }, 1);
  return results[0] ?? null; // { symbol, name, precision, maxSupply, supply, ... }
}
```

### Get Order Book

```typescript
async function getOrderBook(symbol: string, limit = 50) {
  const [buyOrders, sellOrders] = await Promise.all([
    queryContractWithFailover('market', 'buyBook', { symbol }, limit, 0, [{ index: 'priceDec', descending: true }]),
    queryContractWithFailover('market', 'sellBook', { symbol }, limit, 0, [{ index: 'priceDec', descending: false }]),
  ]);

  return { buy: buyOrders, sell: sellOrders };
}
```

### Get Trade History

```typescript
async function getTradeHistory(symbol: string, limit = 50) {
  return queryContractWithFailover('market', 'tradesHistory', { symbol }, limit, 0, [
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
    queryContractWithFailover('market', 'buyBook', query, 100),
    queryContractWithFailover('market', 'sellBook', query, 100),
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
// Each NFT collection has its own table: ${symbol}instances
async function getNFTs(account: string, symbol: string) {
  return queryContractWithFailover('nft', `${symbol}instances`, { account }, 1000);
}

// Get NFT definition
async function getNFTDefinition(symbol: string) {
  return queryContractWithFailover('nft', 'nfts', { symbol }, 1);
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
const pools = await queryContractWithFailover('marketpools', 'pools', {}, 1000);

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

Use `queryContractWithFailover` for all queries. It handles node caching, rotation on failure, and flowerengine fallback automatically. The lower-level `queryContract` is available if you need to pass a specific URL directly.

```typescript
async function queryContract(
  contract: string,
  table: string,
  query: any,
  limit = 1000,
  offset = 0,
  indexes?: Array<{ index: string; descending: boolean }>,
  contractsUrl = 'https://api.hive-engine.com/contracts'
) {
  const response = await fetch(contractsUrl, {
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

// Wraps queryContract with automatic node caching and rotation on failure.
async function queryContractWithFailover(
  contract: string,
  table: string,
  query: any,
  limit = 1000,
  offset = 0,
  indexes?: Array<{ index: string; descending: boolean }>
) {
  try {
    const url = await getEngineContractsUrl();
    return await queryContract(contract, table, query, limit, offset, indexes, url);
  } catch {
    // Current node failed — invalidate cache and retry once with next node
    const url = await getEngineContractsUrl(true);
    return await queryContract(contract, table, query, limit, offset, indexes, url);
  }
}
```
