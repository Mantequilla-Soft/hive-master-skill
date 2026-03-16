# Battle-Tested Workarounds & Gotchas

Hard-won lessons from production Hive apps. These are the things that docs don't tell you.

## 1. dhive Does NOT Auto-Failover on React Native

**Problem:** `@hiveio/dhive` claims to support automatic node failover via the node array, but on React Native (Expo), it does NOT reliably switch nodes when one goes down.

**Fix:** Implement manual node rotation with health checking:

```typescript
const HIVE_NODES = [
  'https://api.hive.blog',
  'https://api.deathwing.me',
  'https://api.openhive.network',
  'https://techcoderx.com',
  'https://api.syncad.com',
  'https://rpc.mahdiyari.info',
];

let currentNodeIndex = 0;
let client = new Client([HIVE_NODES[currentNodeIndex]]);

async function callWithFailover<T>(fn: (client: Client) => Promise<T>): Promise<T> {
  const maxAttempts = HIVE_NODES.length;
  let lastError: Error | null = null;

  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await fn(client);
    } catch (err) {
      lastError = err as Error;
      // Rotate to next node
      currentNodeIndex = (currentNodeIndex + 1) % HIVE_NODES.length;
      client = new Client([HIVE_NODES[currentNodeIndex]], {
        timeout: 20000,
        failoverThreshold: 2,
      });
    }
  }

  throw lastError;
}

// Usage:
const post = await callWithFailover(c => c.database.getContent('author', 'permlink'));
```

**Source:** Discovered in hivesnaps (commit 63bdc52 - "Fix/hive node failover")

## 2. Never Retry Broadcast Operations

**Problem:** If a broadcast appears to fail (timeout, network error), the transaction may have ALREADY been included in a block. Retrying creates duplicate operations on chain.

**Fix:** Never retry `client.broadcast.sendOperations()` or any broadcast call. Instead:
1. Wait 6-9 seconds (2-3 blocks)
2. Check if the operation was included by querying the chain
3. Only re-broadcast if the operation is confirmed NOT on chain

```typescript
// BAD
try {
  await client.broadcast.vote(voteData, key);
} catch {
  await client.broadcast.vote(voteData, key); // DANGEROUS: may duplicate
}

// GOOD
try {
  await client.broadcast.vote(voteData, key);
} catch (err) {
  // Wait for blocks to propagate
  await new Promise(r => setTimeout(r, 9000));

  // Check if vote already applied
  const votes = await client.call('condenser_api', 'get_active_votes', [author, permlink]);
  const alreadyVoted = votes.some(v => v.voter === voter);

  if (!alreadyVoted) {
    await client.broadcast.vote(voteData, key); // Safe to retry
  }
}
```

## 3. Beneficiaries Must Be Sorted Alphabetically

**Problem:** The Hive protocol requires beneficiaries to be sorted by account name. Unsorted beneficiaries cause the transaction to fail with a cryptic error.

```typescript
// BAD
const beneficiaries = [
  { account: 'threespeakfund', weight: 1000 },
  { account: 'alice', weight: 500 },
];

// GOOD
const beneficiaries = [
  { account: 'alice', weight: 500 },
  { account: 'threespeakfund', weight: 1000 },
].sort((a, b) => a.account.localeCompare(b.account));
```

## 4. Block Time is 3 Seconds — Plan Accordingly

After broadcasting, data takes at least 3 seconds to appear on chain. For queries via Hivemind (bridge API), add extra time as Hivemind indexes lag behind head block.

```typescript
// After broadcast, poll with appropriate delays:
await new Promise(r => setTimeout(r, 3000)); // Wait 1 block minimum
// Then poll every 1s for up to 4 retries
```

## 5. Alternate Node Verification for Sync Lag

**Problem:** A node might be behind on sync and return `null` for content that exists on other nodes.

**Fix:** When primary returns null, verify on alternate nodes:

```typescript
async function getPostWithVerification(author: string, permlink: string) {
  const post = await client.call('bridge', 'get_post', { author, permlink });

  if (post) return post;

  // Primary returned null — try alternates
  for (const node of HIVE_NODES.slice(1, 3)) {
    try {
      const altClient = new Client([node], { timeout: 10000 });
      const altPost = await altClient.call('bridge', 'get_post', { author, permlink });
      if (altPost) return altPost;
    } catch {}
  }

  return null; // Genuinely doesn't exist
}
```

**Source:** vision-next `/packages/sdk/src/modules/bridge/verify-on-alternate-node.ts`

## 6. iOS ph:// URIs Break Image Uploads

**Problem:** On iOS, the image picker returns `ph://` URIs (Photos framework) which can't be directly read as files for upload.

**Fix:** Normalize using `expo-image-manipulator`:

```typescript
import * as ImageManipulator from 'expo-image-manipulator';

async function normalizeImageUri(uri: string): Promise<string> {
  if (uri.startsWith('ph://')) {
    const result = await ImageManipulator.manipulateAsync(
      uri,
      [], // No transformations
      { compress: 0.8, format: ImageManipulator.SaveFormat.JPEG }
    );
    return result.uri; // Now a file:// URI
  }
  return uri;
}
```

**Source:** hivesnaps commit 3695ba6

## 7. HEIC Images Need Conversion Before Upload

iOS photos are often HEIC format which most Hive image services don't support:

```typescript
// Convert HEIC → JPEG before upload
if (mimeType === 'image/heic' || mimeType === 'image/heif') {
  const result = await ImageManipulator.manipulateAsync(
    uri, [],
    { compress: 0.8, format: ImageManipulator.SaveFormat.JPEG }
  );
  uri = result.uri;
  mimeType = 'image/jpeg';
}
// Exception: GIFs should NOT be recompressed (lose animation)
```

## 8. ImageSigningChallenge Prefix for Hive Image Uploads

**Problem:** Uploading to `images.hive.blog` requires a specific signing scheme.

**Fix:** The SHA-256 hash MUST be prefixed with `'ImageSigningChallenge'`:

```typescript
const hash = crypto.createHash('sha256');
hash.update('ImageSigningChallenge'); // REQUIRED PREFIX — without this, auth fails
hash.update(imageBuffer);
const signature = privateKey.sign(Buffer.from(hash.digest('hex'), 'hex'));

// Upload: POST https://images.hive.blog/{account}/{signature}
```

## 9. Hive Timestamps Need 'Z' Suffix for UTC

**Problem:** Hive API returns timestamps without timezone info. JavaScript `new Date()` may interpret them as local time.

```typescript
// BAD
const date = new Date(post.created); // Might be off by hours

// GOOD
const date = new Date(post.created + 'Z'); // Force UTC interpretation
```

## 10. WebSocket Origin Validation Kills Mobile Chat

**Problem:** Ecency chat WebSocket (`wss://...`) validates the `Origin` header. React Native can't set custom Origin headers, causing connection rejection.

**Fix:** Use REST API polling instead of WebSocket for mobile:

```typescript
// Instead of WebSocket connection:
// const ws = new WebSocket('wss://ecency.com/...');  // FAILS on mobile

// Use REST polling:
const messages = await fetch('https://ecency.com/api/mattermost/...', {
  headers: { Authorization: `Bearer ${token}` },
});
```

## 11. Cloudflare 52x Errors — Don't Retry

When `images.hive.blog` or other Hive services return 520-529 status codes, it means the origin server is down. Retrying is pointless:

```typescript
if (error.status >= 520 && error.status < 530) {
  console.error(`Origin unavailable (${error.status}), skipping retries`);
  break; // Move to fallback immediately
}
```

## 12. TUS Upload Resume Fails on Some Services

**Problem:** `embed.3speak.tv` doesn't return `Upload-Length` on HEAD requests, so TUS resume always fails.

**Fix:** Clear stale TUS fingerprints:

```typescript
Object.keys(localStorage).forEach(key => {
  if (key.startsWith('tus::') && key.includes('embed.3speak.tv')) {
    localStorage.removeItem(key);
  }
});
```

## 13. posting_json_metadata vs json_metadata

Two metadata fields exist on accounts:
- `posting_json_metadata` — Set via posting key, contains profile data
- `json_metadata` — Set via active key, older field

**Always check `posting_json_metadata` first**, fall back to `json_metadata`:

```typescript
let profile;
try { profile = JSON.parse(account.posting_json_metadata)?.profile; } catch {}
if (!profile) {
  try { profile = JSON.parse(account.json_metadata)?.profile; } catch {}
}
```

## 14. Vote Weight Persistence

Users expect their vote weight slider to stay where they left it:

```typescript
// Save after each vote
await AsyncStorage.setItem('vote_weight', String(voteWeight));

// Load on mount
const saved = await AsyncStorage.getItem('vote_weight');
const weight = saved !== null ? Number(saved) : 100; // Default 100%
```

## 15. Common Chain Error Messages

| Error | Meaning | Solution |
|-------|---------|----------|
| `You may only post once every 5 minutes` | Cooldown on root posts | Wait or use replies |
| `Your current vote is identical` | Duplicate vote | Skip — already voted |
| `Please wait to transact, or power up` | Insufficient RC | Wait for RC regen or get delegation |
| `Cannot delete a comment with net positive votes` | Has curation rewards | Can't delete — has value |
| `Missing Active Authority` | Wrong key type | Use active key instead of posting |
| `Missing Posting Authority` | Wrong key type | Use posting key instead of active |
| `Account does not have sufficient funds` | Balance too low | Check balance before transfer |
| `Bandwidth limit exceeded` | Old error for RC | Same as insufficient RC |

## 16. HiveAuth Single-Flight Connection Guard

**Problem:** Browser visibility events fire multiple times, spawning duplicate WebSocket connections.

```typescript
let connecting = false;

async function connectHiveAuth() {
  if (connecting) return; // Already in progress
  connecting = true;
  try {
    await doConnect();
  } finally {
    connecting = false;
  }
}
```

## 17. Beekeeper Session Limits

When using Hive's Beekeeper wallet manager (for witnesses/automated signing), there's a hard limit of 64 concurrent sessions. Always clean up:

```typescript
// Close session when done
await beekeeper.closeSession(sessionToken);
// Use max timeout: 2147483647ms for long-running processes
```

## 18. Downvote Mana is Separate from Upvote

Downvotes use a separate 25% mana pool. Don't confuse it with upvote mana:

```typescript
// Upvote pool: 100% of effective vesting
const upvoteMax = effectiveVests;

// Downvote pool: 25% of effective vesting
const downvoteMax = effectiveVests / 4;

// Both regenerate over 5 days independently
```
