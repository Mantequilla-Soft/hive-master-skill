# Hive Operations Reference

All Hive blockchain operations as operation tuples `[op_name, op_body]`.

## Content Operations (Posting Key)

### Comment (Post or Reply)

```typescript
const commentOp: Operation = ['comment', {
  parent_author: '',           // Empty for root post, author for reply
  parent_permlink: 'hive-123', // Community tag for root, parent permlink for reply
  author: 'username',
  permlink: 'my-post-permlink',
  title: 'Post Title',        // Empty string for replies
  body: 'Post body in markdown',
  json_metadata: JSON.stringify({
    app: 'myapp/1.0',
    format: 'markdown',
    tags: ['tag1', 'tag2'],
    image: ['https://...'],    // Array of image URLs
    // For video content:
    video: { platform: '3speak', url: 'embed_url', reusable: true },
    // For audio content:
    audio: { platform: '3speak', url: 'embed_url' },
  }),
}];
```

### Comment Options (Set Beneficiaries, Payout Preferences)

Must be broadcast in the SAME transaction as the comment.

```typescript
const commentOptionsOp: Operation = ['comment_options', {
  author: 'username',
  permlink: 'my-post-permlink',
  max_accepted_payout: '1000000.000 HBD', // '0.000 HBD' to decline rewards
  percent_hbd: 10000,  // 10000 = 50/50 split, 0 = 100% HP (power up)
  allow_votes: true,
  allow_curation_rewards: true,
  extensions: [[0, {
    beneficiaries: [
      { account: 'beneficiary1', weight: 500 },  // 5%
      { account: 'beneficiary2', weight: 1000 }, // 10%
    ]
    // CRITICAL: beneficiaries MUST be sorted alphabetically by account name
    // Weight: 100 = 1%, 10000 = 100%
  }]],
}];

// Broadcast both together:
await client.broadcast.sendOperations([commentOp, commentOptionsOp], postingKey);
```

### Delete Comment

```typescript
const deleteOp: Operation = ['delete_comment', {
  author: 'username',
  permlink: 'permlink-to-delete',
}];
// NOTE: Cannot delete comments with net positive votes/payout
```

### Vote

```typescript
const voteOp: Operation = ['vote', {
  voter: 'username',
  author: 'post-author',
  permlink: 'post-permlink',
  weight: 10000, // -10000 to 10000 (basis points, 10000 = 100% upvote)
}];
```

### Reblog (Custom JSON)

```typescript
const reblogOp: Operation = ['custom_json', {
  required_auths: [],
  required_posting_auths: ['username'],
  id: 'follow',
  json: JSON.stringify(['reblog', {
    account: 'username',
    author: 'post-author',
    permlink: 'post-permlink',
  }]),
}];
```

## Wallet Operations (Active Key)

### Transfer

```typescript
const transferOp: Operation = ['transfer', {
  from: 'sender',
  to: 'receiver',
  amount: '1.000 HIVE',  // or '1.000 HBD' — always 3 decimals
  memo: 'optional memo',  // Prefix with '#' for encrypted memo
}];
```

### Power Up (HIVE to HP)

```typescript
const powerUpOp: Operation = ['transfer_to_vesting', {
  from: 'username',
  to: 'username',    // Can power up another account
  amount: '100.000 HIVE',
}];
```

### Power Down (HP to HIVE — 13 weekly payouts)

```typescript
const powerDownOp: Operation = ['withdraw_vesting', {
  account: 'username',
  vesting_shares: '1000.000000 VESTS', // 6 decimals for VESTS
}];
// To cancel: set vesting_shares to '0.000000 VESTS'
```

### Delegate HP

```typescript
const delegateOp: Operation = ['delegate_vesting_shares', {
  delegator: 'from-account',
  delegatee: 'to-account',
  vesting_shares: '1000.000000 VESTS',
}];
// To undelegate: set vesting_shares to '0.000000 VESTS'
// Undelegation has a 5-day cooldown
```

### Savings Operations

```typescript
// Deposit to savings
const depositOp: Operation = ['transfer_to_savings', {
  from: 'username',
  to: 'username',
  amount: '100.000 HBD',
  memo: '',
}];

// Withdraw from savings (3-day wait)
const withdrawOp: Operation = ['transfer_from_savings', {
  from: 'username',
  to: 'username',
  amount: '100.000 HBD',
  memo: '',
  request_id: Date.now(), // Unique ID for this request
}];

// Cancel withdrawal
const cancelOp: Operation = ['cancel_transfer_from_savings', {
  from: 'username',
  request_id: 12345, // ID from the withdraw request
}];
```

### HBD Conversion (3.5-day conversion)

```typescript
const convertOp: Operation = ['convert', {
  owner: 'username',
  amount: '10.000 HBD',
  requestid: Date.now(),
}];
```

### Recurrent Transfer

```typescript
const recurrentOp: Operation = ['recurrent_transfer', {
  from: 'sender',
  to: 'receiver',
  amount: '1.000 HIVE',
  memo: 'monthly payment',
  recurrence: 24,     // Hours between transfers (minimum 24)
  executions: 12,     // Number of times to execute (0 to cancel)
}];
```

### Claim Rewards

```typescript
const claimOp: Operation = ['claim_reward_balance', {
  account: 'username',
  reward_hive: '0.000 HIVE',
  reward_hbd: '0.000 HBD',
  reward_vests: '0.000000 VESTS',
}];
```

## Governance Operations

### Witness Vote (Active Key)

```typescript
const witnessVoteOp: Operation = ['account_witness_vote', {
  account: 'username',
  witness: 'witness-name',
  approve: true, // false to remove vote
}];
```

### Witness Proxy

```typescript
const proxyOp: Operation = ['account_witness_proxy', {
  account: 'username',
  proxy: 'proxy-account', // Empty string to remove proxy
}];
```

### Proposal Vote

```typescript
const proposalVoteOp: Operation = ['update_proposal_votes', {
  voter: 'username',
  proposal_ids: [123],
  approve: true,
  extensions: [],
}];
```

## Permlink Generation

```typescript
// For replies: prefix with 're-'
const sanitizedAuthor = parentAuthor.toLowerCase().replace(/[^a-z0-9-]/g, '');
const permlink = `re-${sanitizedAuthor}-${parentPermlink}-${Date.now()}`;

// For posts: slugify the title
const permlink = title
  .toLowerCase()
  .replace(/[^a-z0-9]+/g, '-')
  .replace(/(^-|-$)/g, '')
  + '-' + Date.now().toString(36);
```

## Broadcasting

```typescript
// Single operation
await client.broadcast.vote(voteBody, postingKey);
await client.broadcast.comment(commentBody, postingKey);
await client.broadcast.transfer(transferBody, activeKey);

// Multiple operations in one transaction
await client.broadcast.sendOperations(
  [commentOp, commentOptionsOp],
  postingKey
);

// CRITICAL: Never retry broadcast operations — duplicates cause errors
// CRITICAL: Use posting key for social ops, active key for financial ops
```
