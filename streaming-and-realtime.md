# Streaming & Real-Time Data

## Block Streaming (dhive)

Stream new blocks/operations as they happen on the chain.

```typescript
import { Client } from '@hiveio/dhive';

const client = new Client(['https://api.hive.blog']);

// Stream all operations
const stream = client.blockchain.getOperationsStream();
stream.on('data', (operation) => {
  console.log(operation.op[0], operation.op[1]);
});
stream.on('error', (err) => {
  console.error('Stream error:', err);
  // Reconnect logic needed
});

// Stream specific operation types
for await (const op of client.blockchain.getOperationsStream()) {
  if (op.op[0] === 'comment') {
    const comment = op.op[1];
    console.log(`New comment by ${comment.author}`);
  }
  if (op.op[0] === 'vote') {
    const vote = op.op[1];
    console.log(`${vote.voter} voted on ${vote.author}/${vote.permlink}`);
  }
  if (op.op[0] === 'transfer') {
    const transfer = op.op[1];
    console.log(`${transfer.from} sent ${transfer.amount} to ${transfer.to}`);
  }
}
```

### Block-Level Streaming

```typescript
// Stream full blocks
for await (const block of client.blockchain.getBlockStream()) {
  console.log(`Block ${block.block_id} with ${block.transactions.length} txs`);
  for (const tx of block.transactions) {
    for (const op of tx.operations) {
      // Process each operation
    }
  }
}
```

### Streaming from a Specific Block

```typescript
// Resume from a known block number
const stream = client.blockchain.getOperationsStream({
  from: 12345678, // Block number to start from
});
```

## Polling Patterns (Mobile-Preferred)

On mobile, WebSocket connections are unreliable. Polling is more battery-efficient and reliable.

### Basic Polling

```typescript
class HivePoller {
  private interval: NodeJS.Timer | null = null;
  private lastBlock = 0;

  start(intervalMs = 3000) { // 3s = 1 block time
    this.interval = setInterval(() => this.poll(), intervalMs);
  }

  stop() {
    if (this.interval) clearInterval(this.interval);
  }

  private async poll() {
    const props = await client.database.getDynamicGlobalProperties();
    const headBlock = props.head_block_number;

    if (headBlock > this.lastBlock) {
      // Process new blocks
      for (let i = this.lastBlock + 1; i <= headBlock; i++) {
        const block = await client.database.getBlock(i);
        this.processBlock(block);
      }
      this.lastBlock = headBlock;
    }
  }

  private processBlock(block: any) {
    for (const tx of block.transactions) {
      for (const op of tx.operations) {
        this.handleOperation(op);
      }
    }
  }
}
```

### Content Polling After Broadcast

Wait for content to appear after publishing:

```typescript
async function waitForContent(author: string, permlink: string, maxWaitMs = 15000) {
  const startTime = Date.now();
  const pollInterval = 3000; // 3s (1 block time)

  // Initial wait for first block
  await new Promise(r => setTimeout(r, 3000));

  while (Date.now() - startTime < maxWaitMs) {
    try {
      const content = await client.database.getContent(author, permlink);
      if (content && content.author === author) return content;
    } catch {}
    await new Promise(r => setTimeout(r, pollInterval));
  }

  return null; // Timed out
}
```

### Feed Polling with Cache

```typescript
class FeedPoller {
  private cache = new Map<string, { data: any; expiresAt: number }>();
  private CACHE_TTL = 60_000; // 1 minute

  async getLatestPosts(tag: string, limit = 20) {
    const cacheKey = `feed:${tag}`;
    const cached = this.cache.get(cacheKey);

    if (cached && Date.now() < cached.expiresAt) {
      return cached.data;
    }

    const posts = await client.call('bridge', 'get_ranked_posts', {
      sort: 'created',
      tag,
      limit,
    });

    this.cache.set(cacheKey, {
      data: posts,
      expiresAt: Date.now() + this.CACHE_TTL,
    });

    return posts;
  }
}
```

## Notifications

### Hive Notifications (Bridge API)

```typescript
const notifications = await client.call('bridge', 'account_notifications', {
  account: 'username',
  last_id: null, // Pagination: last notification ID
  limit: 50,
});

// Notification types: vote, reply, mention, follow, reblog, transfer, etc.
notifications.forEach(n => {
  console.log(`${n.type}: ${n.msg} at ${n.date}`);
});
```

### Mark Notifications Read

```typescript
// Via custom JSON
await client.broadcast.json({
  required_auths: [],
  required_posting_auths: ['username'],
  id: 'notify',
  json: JSON.stringify(['setLastRead', { date: new Date().toISOString() }]),
}, postingKey);
```

## WebSocket Considerations

### Why WebSocket Often Fails on Mobile

1. **Origin validation** — Many Hive services validate WebSocket origin headers, which mobile apps can't set
2. **Connection persistence** — Mobile OS aggressively kills background connections
3. **Battery drain** — Persistent connections drain battery
4. **Network transitions** — WiFi↔cellular handoffs break connections

### When to Use WebSocket vs Polling

| Use Case | Recommendation |
|----------|---------------|
| Server-side block streaming | WebSocket/stream |
| Mobile app feed updates | Polling (30s-60s) |
| Chat/messaging | REST polling or FCM push |
| Real-time vote tracking | Polling (3s-5s) |
| Price feeds | Polling (60s) |

### Push Notifications (Firebase/FCM) Pattern

For true real-time on mobile, use server-side streaming + push:

```
User posts → Hive chain → Your server streams blocks →
  Detects relevant ops → Sends FCM push → Device receives notification →
  App fetches latest data via REST
```

This avoids keeping persistent connections on the device.

## Server-Sent Events (SSE) for Progress

For long-running server operations (e.g., fetching full transaction history):

```typescript
// Server side (Node.js)
app.get('/api/history/:username', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });

  async function streamHistory() {
    let batch = 0;
    let start = -1;

    while (true) {
      const ops = await client.call('condenser_api', 'get_account_history', [
        req.params.username, start, 1000,
      ]);

      if (ops.length === 0) break;

      res.write(`data: ${JSON.stringify({
        batch: ++batch,
        count: ops.length,
        operations: ops,
      })}\n\n`);

      start = ops[0][0] - 1;
      if (start < 0) break;
    }

    res.write('data: {"done": true}\n\n');
    res.end();
  }

  streamHistory();
});
```

## Global Properties Polling

For vote value calculations, price data, etc.:

```typescript
// Cache global properties (refresh every 2 minutes)
let globalPropsCache: any = null;
let globalPropsCacheTime = 0;

async function getGlobalProps() {
  if (globalPropsCache && Date.now() - globalPropsCacheTime < 120_000) {
    return globalPropsCache;
  }

  globalPropsCache = await client.database.getDynamicGlobalProperties();
  globalPropsCacheTime = Date.now();
  return globalPropsCache;
}
```
