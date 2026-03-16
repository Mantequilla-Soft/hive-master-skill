# Common dhive Patterns

## Full Post with Beneficiaries

```typescript
import { Client, PrivateKey, Operation } from '@hiveio/dhive';

const client = new Client([
  'https://api.hive.blog',
  'https://api.deathwing.me',
  'https://api.openhive.network',
]);

async function publishPost({
  author,
  title,
  body,
  tags,
  community,
  beneficiaries = [],
  declineRewards = false,
  powerUp = false,
  postingKeyWIF,
}: {
  author: string;
  title: string;
  body: string;
  tags: string[];
  community?: string;
  beneficiaries?: Array<{ account: string; weight: number }>;
  declineRewards?: boolean;
  powerUp?: boolean;
  postingKeyWIF: string;
}) {
  const permlink = title
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/(^-|-$)/g, '')
    + '-' + Date.now().toString(36);

  const parentPermlink = community || tags[0] || 'general';

  const jsonMetadata = JSON.stringify({
    app: 'myapp/1.0',
    format: 'markdown',
    tags,
    image: extractImagesFromBody(body),
  });

  const ops: Operation[] = [];

  // 1. Comment operation
  ops.push(['comment', {
    parent_author: '',
    parent_permlink: parentPermlink,
    author,
    permlink,
    title,
    body,
    json_metadata: jsonMetadata,
  }]);

  // 2. Comment options (beneficiaries, payout preferences)
  if (beneficiaries.length > 0 || declineRewards || powerUp) {
    // CRITICAL: Sort beneficiaries alphabetically
    const sortedBeneficiaries = [...beneficiaries].sort(
      (a, b) => a.account.localeCompare(b.account)
    );

    ops.push(['comment_options', {
      author,
      permlink,
      max_accepted_payout: declineRewards ? '0.000 HBD' : '1000000.000 HBD',
      percent_hbd: powerUp ? 0 : 10000,
      allow_votes: true,
      allow_curation_rewards: true,
      extensions: sortedBeneficiaries.length > 0
        ? [[0, { beneficiaries: sortedBeneficiaries }]]
        : [],
    }]);
  }

  const postingKey = PrivateKey.fromString(postingKeyWIF);
  const result = await client.broadcast.sendOperations(ops, postingKey);

  return { ...result, permlink };
}

function extractImagesFromBody(body: string): string[] {
  const regex = /!\[.*?\]\((https?:\/\/[^\s)]+)\)/g;
  const images: string[] = [];
  let match;
  while ((match = regex.exec(body)) !== null) {
    images.push(match[1]);
  }
  return images;
}
```

## Full Vote with Value Estimation

```typescript
async function voteWithEstimate({
  voter,
  author,
  permlink,
  weightPercent, // 1-100
  postingKeyWIF,
}: {
  voter: string;
  author: string;
  permlink: string;
  weightPercent: number;
  postingKeyWIF: string;
}) {
  // Fetch required data in parallel
  const [accounts, globalProps, rewardFund] = await Promise.all([
    client.database.getAccounts([voter]),
    client.database.getDynamicGlobalProperties(),
    client.database.call('get_reward_fund', ['post']),
  ]);

  const account = accounts[0];
  const weight = Math.round(weightPercent * 100); // Convert to basis points

  // Calculate estimated vote value
  const vests = parseFloat(account.vesting_shares)
    + parseFloat(account.received_vesting_shares)
    - parseFloat(account.delegated_vesting_shares);

  const votingPower = calculateCurrentVotingPower(account);
  const usedPower = (votingPower * weight) / 10000;
  const rshares = vests * 1e6 * usedPower / 10000;

  const rewardBalance = parseFloat(rewardFund.reward_balance);
  const recentClaims = parseFloat(rewardFund.recent_claims);
  const hbdPrice = parseFloat(globalProps.current_median_history_base);

  const estimatedValue = (rshares * rewardBalance / recentClaims) * hbdPrice;

  // Broadcast vote
  const postingKey = PrivateKey.fromString(postingKeyWIF);
  await client.broadcast.vote({ voter, author, permlink, weight }, postingKey);

  return { estimatedValue: estimatedValue.toFixed(3) };
}
```

## Follow/Unfollow/Mute

```typescript
async function socialAction(
  action: 'follow' | 'unfollow' | 'mute',
  follower: string,
  target: string,
  postingKeyWIF: string,
) {
  const what = action === 'follow' ? ['blog']
    : action === 'mute' ? ['ignore']
    : []; // unfollow = empty

  const postingKey = PrivateKey.fromString(postingKeyWIF);

  await client.broadcast.json({
    required_auths: [],
    required_posting_auths: [follower],
    id: 'follow',
    json: JSON.stringify(['follow', {
      follower,
      following: target,
      what,
    }]),
  }, postingKey);
}
```

## Paginated Feed Loading

```typescript
async function loadFeed(sort: string, tag: string, limit = 20) {
  let allPosts: any[] = [];
  let startAuthor = '';
  let startPermlink = '';

  async function loadMore() {
    const params: any = { sort, limit, tag };
    if (startAuthor) {
      params.start_author = startAuthor;
      params.start_permlink = startPermlink;
    }

    const posts = await client.call('bridge', 'get_ranked_posts', params);

    if (posts.length > 0) {
      // When paginating, first result is duplicate of last from previous page
      const newPosts = startAuthor ? posts.slice(1) : posts;
      allPosts = [...allPosts, ...newPosts];

      const last = posts[posts.length - 1];
      startAuthor = last.author;
      startPermlink = last.permlink;
    }

    return posts.length === limit; // hasMore
  }

  return { loadMore, posts: allPosts };
}
```

## Account Info Helper

```typescript
async function getAccountInfo(username: string) {
  const [account] = await client.database.getAccounts([username]);
  if (!account) return null;

  const globalProps = await client.database.getDynamicGlobalProperties();
  const totalVests = parseFloat(globalProps.total_vesting_shares);
  const totalFund = parseFloat(globalProps.total_vesting_fund_hive);

  const ownVests = parseFloat(account.vesting_shares);
  const receivedVests = parseFloat(account.received_vesting_shares);
  const delegatedVests = parseFloat(account.delegated_vesting_shares);

  const vestsToHP = (v: number) => (v / totalVests) * totalFund;

  // Parse profile
  let profile: any = {};
  try { profile = JSON.parse(account.posting_json_metadata)?.profile || {}; } catch {}
  if (!profile.name) {
    try { profile = JSON.parse(account.json_metadata)?.profile || {}; } catch {}
  }

  return {
    name: account.name,
    displayName: profile.name || account.name,
    about: profile.about || '',
    avatar: profile.profile_image || `https://images.hive.blog/u/${account.name}/avatar`,
    coverImage: profile.cover_image || '',
    location: profile.location || '',
    website: profile.website || '',

    hiveBalance: account.balance,
    hbdBalance: account.hbd_balance,
    savingsHive: account.savings_balance,
    savingsHbd: account.savings_hbd_balance,

    ownHP: vestsToHP(ownVests).toFixed(3),
    receivedHP: vestsToHP(receivedVests).toFixed(3),
    delegatedHP: vestsToHP(delegatedVests).toFixed(3),
    effectiveHP: vestsToHP(ownVests + receivedVests - delegatedVests).toFixed(3),

    votingPower: calculateCurrentVotingPower(account) / 100, // As percentage
    reputation: account.reputation,
    postCount: account.post_count,

    created: new Date(account.created + 'Z'),
    lastVoteTime: new Date(account.last_vote_time + 'Z'),
  };
}
```

## Resource Credits Check

```typescript
async function checkRC(username: string): Promise<{
  percentage: number;
  canTransact: boolean;
  estimatedComments: number;
}> {
  const response = await client.call('rc_api', 'find_rc_accounts', {
    accounts: [username],
  });

  const rc = response.rc_accounts[0];
  const current = parseFloat(rc.rc_manabar.current_mana);
  const max = parseFloat(rc.max_rc);
  const percentage = (current / max) * 100;

  // Rough estimates: a comment costs ~1.5B RC, a vote costs ~400M RC
  const COMMENT_COST = 1_500_000_000;
  const VOTE_COST = 400_000_000;

  return {
    percentage: Math.round(percentage * 100) / 100,
    canTransact: percentage > 5,
    estimatedComments: Math.floor(current / COMMENT_COST),
  };
}
```
