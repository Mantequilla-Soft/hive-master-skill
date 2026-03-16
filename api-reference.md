# Hive API Reference

## Public RPC Nodes (Tested & Reliable)

```
https://api.hive.blog          # Official, most stable
https://api.deathwing.me       # Fast, reliable
https://api.openhive.network   # Good fallback
https://techcoderx.com         # Good but can rate-limit
https://api.syncad.com         # Reliable
https://rpc.mahdiyari.info     # Community node
https://anyx.io                # Long-running node
https://hive-api.arcange.eu    # European node
https://hive-api.3speak.tv     # 3Speak's node
```

### Recommended Client Setup (dhive)

```typescript
import { Client } from '@hiveio/dhive';

const HIVE_NODES = [
  'https://api.hive.blog',
  'https://api.deathwing.me',
  'https://api.openhive.network',
];

const client = new Client(HIVE_NODES, {
  timeout: 20000,           // 20s timeout
  failoverThreshold: 2,     // Switch after 2 failures
  consoleOnFailover: true,  // Log when switching
});
```

**CRITICAL (React Native):** dhive does NOT automatically failover on React Native. You must implement manual node rotation. See [battle-tested.md](battle-tested.md) for the fix.

## API Namespaces

### 1. Condenser API (`condenser_api`)

The legacy API — still widely used. Called via `client.call('condenser_api', method, params)`.

| Method | Params | Returns |
|--------|--------|---------|
| `get_accounts` | `[['username1', 'username2']]` | Full account objects |
| `get_content` | `[author, permlink]` | Single post/comment |
| `get_content_replies` | `[author, permlink]` | Direct replies array |
| `get_discussions_by_blog` | `[{tag, limit, start_author?, start_permlink?}]` | Blog posts |
| `get_discussions_by_trending` | `[{tag?, limit, start_author?, start_permlink?}]` | Trending posts |
| `get_discussions_by_hot` | `[{tag?, limit}]` | Hot posts |
| `get_discussions_by_created` | `[{tag?, limit}]` | New posts |
| `get_discussions_by_feed` | `[{tag: username, limit}]` | User's feed |
| `get_following` | `[username, start_following, follow_type, limit]` | Following list |
| `get_followers` | `[username, start_follower, follow_type, limit]` | Follower list |
| `get_active_votes` | `[author, permlink]` | Vote list on content |
| `get_account_history` | `[account, start, limit]` | Transaction history |
| `get_dynamic_global_properties` | `[]` | Chain state |
| `get_reward_fund` | `['post']` | Reward pool info |

### 2. Bridge API (`bridge`)

The modern Hivemind API — preferred for content queries.

| Method | Params | Returns |
|--------|--------|---------|
| `get_post` | `{author, permlink, observer?}` | Full post with metadata |
| `get_discussion` | `{author, permlink, observer?}` | Post + all comments |
| `get_account_posts` | `{sort, account, start_author?, start_permlink?, limit, observer?}` | Account posts |
| `get_ranked_posts` | `{sort, start_author?, start_permlink?, limit, tag?, observer?}` | Ranked feed |
| `get_community` | `{name, observer?}` | Community details |
| `list_communities` | `{last?, limit?, query?, sort?, observer?}` | Communities list |
| `get_community_context` | `{name, account}` | User's role in community |
| `list_subscribers` | `{community, last?, limit?}` | Subscribers list |
| `account_notifications` | `{account, last_id?, limit?}` | Notifications |
| `get_follow_list` | `{observer, follow_type}` | Follow/mute lists |
| `get_profile` | `{account, observer?}` | User profile |
| `get_trending_topics` | `{limit?, observer?}` | Trending tags |

**Sort values for `get_ranked_posts`:** `trending`, `hot`, `created`, `promoted`, `payout`, `payout_comments`, `muted`

**Sort values for `get_account_posts`:** `blog`, `feed`, `posts`, `comments`, `replies`, `payout`

### 3. Database API (`database_api`)

Lower-level chain data access. Used via `client.database.*` methods.

```typescript
// Dynamic global properties
const props = await client.database.getDynamicGlobalProperties();

// Account data
const accounts = await client.database.getAccounts(['username']);

// Single content
const content = await client.database.getContent('author', 'permlink');

// Reward fund (for vote value calculations)
const fund = await client.database.call('get_reward_fund', ['post']);
```

### 4. RC API (`rc_api`)

Resource Credit tracking.

```typescript
const response = await client.call('rc_api', 'find_rc_accounts', {
  accounts: ['username'],
});

const rcData = response.rc_accounts[0];
const currentMana = parseFloat(rcData.rc_manabar.current_mana);
const maxRc = parseFloat(rcData.max_rc);
const percentage = (currentMana / maxRc) * 100;
```

### 5. Account History API

```typescript
// Fetch transaction history in batches
// start: -1 for most recent, limit: max 1000 per call
const history = await client.call('condenser_api', 'get_account_history', [
  'username',
  -1,    // start from most recent
  1000,  // batch size (max 1000)
]);
```

**Pagination:** Use the first operation's index as the new `start` parameter for the next batch.

## Key Endpoints for Specific Services

| Service | Endpoint | Purpose |
|---------|----------|---------|
| Hive Images | `https://images.hive.blog` | Image hosting (signature-based) |
| Hivemind | Same as RPC nodes | Social layer queries |
| HiveAuth | `wss://hive-auth.arcange.eu` | Mobile auth WebSocket |
| HiveSigner | `https://hivesigner.com` | OAuth-based auth |
| Ecency API | `https://ecency.com/api` | Ecency services |
| 3Speak | `https://studio.3speak.tv` | Video hosting |
| HafSQL | `hafbe.openhive.network` | SQL-accessible chain data |

## Image Upload (Hive Images)

```typescript
// CRITICAL: Use 'ImageSigningChallenge' prefix when signing
const hash = crypto.createHash('sha256');
hash.update('ImageSigningChallenge');  // REQUIRED PREFIX
hash.update(imageBuffer);
const hashHex = hash.digest('hex');

const signature = privateKey.sign(Buffer.from(hashHex, 'hex'));

// Upload to: POST https://images.hive.blog/{account}/{signature}
```

## Documentation & Learning Resources

- **Hive Developer Portal:** https://developers.hive.io
- **dhive Library:** https://github.com/openhive-network/dhive
- **Hive Keychain:** https://github.com/nicholasgasior/hive-keychain-extension
- **HiveAuth:** https://hiveauth.com
- **Hivemind:** https://gitlab.syncad.com/hive/hivemind
- **Hive Engine:** https://he.dtools.dev (API docs)
