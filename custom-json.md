# Custom JSON Operations

Custom JSON is the Swiss army knife of Hive — used for social actions, app data, layer 2 tokens, and more.

## Structure

```typescript
const customJsonOp: Operation = ['custom_json', {
  required_auths: [],              // Accounts requiring active key (financial)
  required_posting_auths: ['user'], // Accounts requiring posting key (social)
  id: 'operation-id',              // Identifies the operation type
  json: JSON.stringify(payload),   // The actual data
}];
```

**Rule:** Use `required_posting_auths` for social ops, `required_auths` for financial ops.

## Social Operations (Posting Key)

### Follow

```typescript
await client.broadcast.json({
  required_auths: [],
  required_posting_auths: ['follower'],
  id: 'follow',
  json: JSON.stringify(['follow', {
    follower: 'follower',
    following: 'target-user',
    what: ['blog'], // Follow their blog
  }]),
}, postingKey);
```

### Unfollow

```typescript
// Same as follow but with empty 'what' array
json: JSON.stringify(['follow', {
  follower: 'follower',
  following: 'target-user',
  what: [], // Empty = unfollow
}])
```

### Mute (Ignore)

```typescript
json: JSON.stringify(['follow', {
  follower: 'username',
  following: 'target-user',
  what: ['ignore'], // Mute/ignore
}])
// To unmute: set what to [] (same as unfollow)
```

### Reblog (Resteem)

```typescript
await client.broadcast.json({
  required_auths: [],
  required_posting_auths: ['username'],
  id: 'follow',
  json: JSON.stringify(['reblog', {
    account: 'username',
    author: 'post-author',
    permlink: 'post-permlink',
  }]),
}, postingKey);

// Delete reblog
json: JSON.stringify(['delete_reblog', {
  account: 'username',
  author: 'post-author',
  permlink: 'post-permlink',
}])
```

### Mark Notifications Read

```typescript
// Hive Notify
await client.broadcast.json({
  required_auths: [],
  required_posting_auths: ['username'],
  id: 'notify',
  json: JSON.stringify(['setLastRead', { date: new Date().toISOString() }]),
}, postingKey);
```

## Community Operations (Posting Key)

All community ops use `id: 'community'`:

```typescript
const communityOp = (action: string, data: object) => ({
  required_auths: [],
  required_posting_auths: ['username'],
  id: 'community',
  json: JSON.stringify([action, data]),
});
```

### Subscribe / Unsubscribe

```typescript
json: JSON.stringify(['subscribe', { community: 'hive-123456' }])
json: JSON.stringify(['unsubscribe', { community: 'hive-123456' }])
```

### Set User Role (Admin/Mod only)

```typescript
json: JSON.stringify(['setRole', {
  community: 'hive-123456',
  account: 'target-user',
  role: 'mod', // 'admin' | 'mod' | 'member' | 'guest' | 'muted'
}])
```

### Pin/Unpin Post

```typescript
json: JSON.stringify(['pinPost', {
  community: 'hive-123456',
  account: 'author',
  permlink: 'post-permlink',
  pin: true, // false to unpin
}])
```

### Mute/Unmute Post (Moderation)

```typescript
json: JSON.stringify(['mutePost', {
  community: 'hive-123456',
  account: 'author',
  permlink: 'post-permlink',
  notes: 'Reason for muting',
  mute: true, // false to unmute
}])
```

### Mute/Unmute User in Community

```typescript
json: JSON.stringify(['muteUser', {
  community: 'hive-123456',
  account: 'target-user',
  notes: 'Reason',
  mute: true,
}])
```

### Flag Post

```typescript
json: JSON.stringify(['flagPost', {
  community: 'hive-123456',
  account: 'author',
  permlink: 'post-permlink',
  notes: 'Reason for flagging',
}])
```

### Update Community Properties

```typescript
json: JSON.stringify(['updateProps', {
  community: 'hive-123456',
  props: {
    title: 'Community Name',
    about: 'Description',
    is_nsfw: false,
    description: 'Longer description',
    flag_text: 'Community rules for flagging',
  },
}])
```

## Resource Credit Delegation (Posting Key)

```typescript
await client.broadcast.json({
  required_auths: [],
  required_posting_auths: ['delegator'],
  id: 'rc',
  json: JSON.stringify(['delegate_rc', {
    from: 'delegator',
    delegatees: ['user1', 'user2'],
    max_rc: 5000000000, // Amount of RC to delegate
  }]),
}, postingKey);
```

## App-Specific Custom JSON

### Ecency Points Transfer

```typescript
// Requires active authority
json: JSON.stringify(['transfer', {
  sender: 'from-user',
  receiver: 'to-user',
  amount: '100',
  memo: 'Points transfer',
}])
// id: 'ecency_point_transfer'
```

### Ecency Boost/Promote

```typescript
// Boost a post (active authority)
id: 'ecency_boost'
json: JSON.stringify({ user: 'username', author: 'author', permlink: 'permlink', amount: '150' })

// Promote a post
id: 'ecency_promote'
json: JSON.stringify({ user: 'username', author: 'author', permlink: 'permlink', duration: 7 })
```

## Hive Engine Operations

### Token Transfer

```typescript
await client.broadcast.json({
  required_auths: ['username'],  // Active authority for financial ops
  required_posting_auths: [],
  id: 'ssc-mainnet-hive',
  json: JSON.stringify({
    contractName: 'tokens',
    contractAction: 'transfer',
    contractPayload: {
      symbol: 'BEE',
      to: 'recipient',
      quantity: '10',
      memo: 'Payment',
    },
  }),
}, activeKey);
```

### Token Stake/Unstake

```typescript
// Stake
json: JSON.stringify({
  contractName: 'tokens',
  contractAction: 'stake',
  contractPayload: { to: 'username', symbol: 'BEE', quantity: '100' },
})

// Unstake
json: JSON.stringify({
  contractName: 'tokens',
  contractAction: 'unstake',
  contractPayload: { symbol: 'BEE', quantity: '50' },
})
```

### Market Orders

```typescript
// Buy order
json: JSON.stringify({
  contractName: 'market',
  contractAction: 'buy',
  contractPayload: { symbol: 'BEE', quantity: '10', price: '0.5' },
})

// Sell order
json: JSON.stringify({
  contractName: 'market',
  contractAction: 'sell',
  contractPayload: { symbol: 'BEE', quantity: '10', price: '0.6' },
})
```
