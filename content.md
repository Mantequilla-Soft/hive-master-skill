# Content: Posts, Comments, Votes, Reblogs, Feeds

## Reading Content

### Single Post (Bridge API — Preferred)

```typescript
const post = await client.call('bridge', 'get_post', {
  author: 'username',
  permlink: 'post-permlink',
  observer: 'viewer-username', // Optional: includes vote status
});
```

### Single Post (Condenser API — Legacy)

```typescript
const post = await client.database.getContent('author', 'permlink');
```

### Get Replies to a Post

```typescript
// Direct replies only
const replies = await client.database.call('get_content_replies', [author, permlink]);

// Full discussion tree (post + all nested comments)
const discussion = await client.call('bridge', 'get_discussion', {
  author, permlink, observer
});
```

### Feeds (Ranked Content)

```typescript
// Trending, hot, new, promoted
const posts = await client.call('bridge', 'get_ranked_posts', {
  sort: 'trending',        // 'trending' | 'hot' | 'created' | 'promoted'
  tag: 'hive-123456',      // Optional: filter by community
  limit: 20,
  start_author: '',        // Pagination cursor
  start_permlink: '',
  observer: 'viewer',
});
```

### Account Posts

```typescript
const posts = await client.call('bridge', 'get_account_posts', {
  sort: 'posts',  // 'blog' | 'feed' | 'posts' | 'comments' | 'replies' | 'payout'
  account: 'username',
  limit: 20,
  start_author: '',       // Pagination cursor
  start_permlink: '',
  observer: 'viewer',
});
```

### User Feed (Posts from Followed Accounts)

```typescript
const feed = await client.call('bridge', 'get_account_posts', {
  sort: 'feed',
  account: 'username',
  limit: 20,
});
```

## Creating Content

### Root Post (to a Community)

```typescript
const comment = {
  parent_author: '',              // Empty = root post
  parent_permlink: 'hive-123456', // Community tag
  author: 'username',
  permlink: generatePermlink(title),
  title: 'My Post Title',
  body: markdownBody,
  json_metadata: JSON.stringify({
    app: 'myapp/1.0',
    format: 'markdown',
    tags: ['hive-123456', 'tag2', 'tag3'],
    image: ['https://images.hive.blog/...'],
    description: 'Short preview text',
  }),
};
```

### Reply to a Post/Comment

```typescript
const reply = {
  parent_author: 'original-author',
  parent_permlink: 'original-permlink',
  author: 'replier',
  permlink: `re-${parentAuthor}-${parentPermlink}-${Date.now()}`,
  title: '',  // Always empty for replies
  body: 'Reply text with markdown support',
  json_metadata: JSON.stringify({
    app: 'myapp/1.0',
    format: 'markdown',
    tags: ['reply'],
    image: [...imageUrls],
  }),
};
```

### Embedding Media in Body

```typescript
let body = text.trim();

// Images
images.forEach(url => { body += `\n![image](${url})`; });

// GIFs
gifs.forEach(url => { body += `\n![gif](${url})`; });

// 3Speak video embed
if (videoUrl) { body += `\n${videoUrl}`; }  // Just the URL, 3Speak renders it

// Audio embed
if (audioUrl) { body += `\n${audioUrl}`; }
```

## Voting

### Cast a Vote

```typescript
await client.broadcast.vote({
  voter: 'username',
  author: 'post-author',
  permlink: 'post-permlink',
  weight: 10000,  // 100% upvote
}, postingKey);

// Weight scale:
//  10000 = 100% upvote
//   5000 = 50% upvote
//      0 = remove vote
//  -5000 = 50% downvote
// -10000 = 100% downvote
```

### Vote Value Calculation

```typescript
async function calculateVoteValue(account, weight, globalProps, rewardFund) {
  const vestingShares = parseFloat(account.vesting_shares);
  const receivedShares = parseFloat(account.received_vesting_shares);
  const delegatedShares = parseFloat(account.delegated_vesting_shares);
  const effectiveVests = vestingShares + receivedShares - delegatedShares;

  const totalVestingShares = parseFloat(globalProps.total_vesting_shares);
  const totalVestingFund = parseFloat(globalProps.total_vesting_fund_hive);

  const rewardBalance = parseFloat(rewardFund.reward_balance);
  const recentClaims = parseFloat(rewardFund.recent_claims);

  // Voting power (regenerates 20% per day)
  const votingPower = calculateCurrentVotingPower(account);

  // Used power for this vote
  const usedPower = (votingPower * Math.abs(weight)) / 10000;

  // rshares
  const rshares = effectiveVests * 1e6 * usedPower / 10000;

  // Estimated value
  const estimate = rshares * rewardBalance / recentClaims;
  const hivePrice = parseFloat(globalProps.current_median_history_base); // HBD per HIVE

  return estimate * hivePrice; // Value in HBD
}
```

### Voting Power (Mana) Calculation

```typescript
function calculateCurrentVotingPower(account) {
  // Voting mana regenerates at 20% per day (full in 5 days)
  const REGEN_SECONDS = 5 * 24 * 60 * 60; // 432000

  if (account.voting_manabar) {
    const currentMana = parseInt(account.voting_manabar.current_mana);
    const lastUpdate = parseInt(account.voting_manabar.last_update_time);
    const maxMana = calculateMaxMana(account);

    const elapsed = Math.floor(Date.now() / 1000) - lastUpdate;
    const regenerated = (elapsed * maxMana) / REGEN_SECONDS;
    const totalMana = Math.min(maxMana, currentMana + regenerated);

    return Math.floor((totalMana / maxMana) * 10000); // 0-10000
  }

  return account.voting_power || 0;
}

function calculateMaxMana(account) {
  const vesting = parseFloat(account.vesting_shares);
  const received = parseFloat(account.received_vesting_shares);
  const delegated = parseFloat(account.delegated_vesting_shares);
  return Math.max(0, vesting + received - delegated);
}
```

### Downvote Mana (Separate Pool)

Downvotes have a separate 25% mana pool that also regenerates over 5 days.

```typescript
function calculateDownvoteMana(account) {
  const REGEN_SECONDS = 5 * 24 * 60 * 60;
  const manabar = account.downvote_manabar;
  const currentMana = parseInt(manabar.current_mana);
  const lastUpdate = parseInt(manabar.last_update_time);
  const maxMana = calculateMaxMana(account) / 4; // 25% of upvote pool

  const elapsed = Math.floor(Date.now() / 1000) - lastUpdate;
  const regenerated = (elapsed * maxMana) / REGEN_SECONDS;
  return Math.min(maxMana, currentMana + regenerated);
}
```

## Optimistic Updates

Update UI immediately, don't wait for blockchain confirmation:

```typescript
// Before broadcast
const optimisticUpdate = {
  hasUpvoted: true,
  upvoteWeight: voteWeight,
  net_votes: currentVotes + 1,
  pending_payout_value: `${(currentPayout + estimatedIncrease).toFixed(3)} HBD`,
};
updatePostInState(author, permlink, optimisticUpdate);

// Then broadcast (if it fails, rollback)
try {
  await client.broadcast.vote(voteData, postingKey);
} catch (err) {
  rollbackOptimisticUpdate(author, permlink);
}
```

## Reply Sorting

```typescript
function sortByPayout(replies) {
  return replies
    .sort((a, b) => {
      const payoutA = parseFloat(a.pending_payout_value?.replace(' HBD', '') || '0');
      const payoutB = parseFloat(b.pending_payout_value?.replace(' HBD', '') || '0');
      return payoutB - payoutA;
    })
    .map(reply => ({
      ...reply,
      replies: reply.replies ? sortByPayout(reply.replies) : undefined,
    }));
}
```

## User Profile & Avatar

```typescript
function getUserAvatar(account) {
  try {
    // Check posting_json_metadata first (account-level profile)
    const meta = JSON.parse(account.posting_json_metadata);
    if (meta?.profile?.profile_image) return meta.profile.profile_image;
  } catch {}

  try {
    // Fallback to json_metadata
    const meta = JSON.parse(account.json_metadata);
    if (meta?.profile?.profile_image) return meta.profile.profile_image;
  } catch {}

  // Ultimate fallback: Hive images service
  return `https://images.hive.blog/u/${account.name}/avatar`;
}
```

## Content Polling After Broadcast

The blockchain needs time to process. Poll for content after publishing:

```typescript
// Wait for at least one block (3 seconds)
await new Promise(resolve => setTimeout(resolve, 3000));

let retries = 0;
const maxRetries = 4;

const poll = async () => {
  const content = await client.database.getContent(author, permlink);
  if (content && content.author) return content;

  if (++retries < maxRetries) {
    await new Promise(resolve => setTimeout(resolve, 1000));
    return poll();
  }
  return null;
};
```

## Following/Followers List

```typescript
// Get who a user follows
const following = await client.call('condenser_api', 'get_following', [
  'username',
  '',       // start_following (pagination)
  'blog',   // follow_type
  1000,     // limit (max 1000)
]);

// Get a user's followers
const followers = await client.call('condenser_api', 'get_followers', [
  'username',
  '',       // start_follower
  'blog',
  1000,
]);

// Use Set for O(1) lookups when filtering feeds
const followingSet = new Set(following.map(f => f.following));
```

## Muted Users & Content Filtering

```typescript
// Get user's personal mute list
const muted = await client.call('bridge', 'get_follow_list', {
  observer: 'username',
  follow_type: 'muted',
});

// Combine with app-level blacklist for defense in depth
const combined = new Set([...personalMuted, ...appBlacklist]);
const filteredPosts = posts.filter(p => !combined.has(p.author));
```
