# Hive Communities

## Community Identifiers

Communities use the format `hive-NNNNNN` (e.g., `hive-123456`). This is both:
- The community "account name" on chain
- The `parent_permlink` for posts published to that community

## Querying Communities

### List Communities

```typescript
const communities = await client.call('bridge', 'list_communities', {
  last: '',        // Pagination cursor
  limit: 100,
  query: '',       // Search filter
  sort: 'rank',    // 'rank' | 'new' | 'subs'
  observer: 'username',
});
```

### Get Community Details

```typescript
const community = await client.call('bridge', 'get_community', {
  name: 'hive-123456',
  observer: 'username',
});
// Returns: { name, title, about, description, lang, is_nsfw, subscribers, ... }
```

### Get User's Role in Community

```typescript
const context = await client.call('bridge', 'get_community_context', {
  name: 'hive-123456',
  account: 'username',
});
// Returns: { role: 'admin'|'mod'|'member'|'guest'|'muted', subscribed: boolean, title: '' }
```

### List Subscribers

```typescript
const subscribers = await client.call('bridge', 'list_subscribers', {
  community: 'hive-123456',
  last: '',   // Pagination
  limit: 100,
});
```

## Community Operations (Custom JSON)

All operations use `id: 'community'` and posting key.

### Subscribe / Unsubscribe

```typescript
await client.broadcast.json({
  required_auths: [],
  required_posting_auths: ['username'],
  id: 'community',
  json: JSON.stringify(['subscribe', { community: 'hive-123456' }]),
}, postingKey);

// Unsubscribe
json: JSON.stringify(['unsubscribe', { community: 'hive-123456' }])
```

### Post to a Community

Posts to communities are regular comment operations with `parent_permlink` set to the community name:

```typescript
await client.broadcast.comment({
  parent_author: '',                // Empty for root post
  parent_permlink: 'hive-123456',   // Community identifier
  author: 'username',
  permlink: 'my-post',
  title: 'Post Title',
  body: 'Post body...',
  json_metadata: JSON.stringify({
    app: 'myapp/1.0',
    tags: ['hive-123456', 'other-tag'],
    // First tag should be the community
  }),
}, postingKey);
```

### Moderation: Set Role

```typescript
json: JSON.stringify(['setRole', {
  community: 'hive-123456',
  account: 'target-user',
  role: 'mod',  // 'admin' | 'mod' | 'member' | 'guest' | 'muted'
}])
```

### Moderation: Pin/Unpin Post

```typescript
json: JSON.stringify(['pinPost', {
  community: 'hive-123456',
  account: 'author',
  permlink: 'post-permlink',
  pin: true,
}])
```

### Moderation: Mute Post

```typescript
json: JSON.stringify(['mutePost', {
  community: 'hive-123456',
  account: 'author',
  permlink: 'post-permlink',
  notes: 'Violates rule #3',
  mute: true,
}])
```

### Moderation: Mute User

```typescript
json: JSON.stringify(['muteUser', {
  community: 'hive-123456',
  account: 'target-user',
  notes: 'Repeated spam',
  mute: true,
}])
```

### Flag Post

```typescript
json: JSON.stringify(['flagPost', {
  community: 'hive-123456',
  account: 'author',
  permlink: 'post-permlink',
  notes: 'Plagiarism',
}])
```

### Update Community Properties

```typescript
json: JSON.stringify(['updateProps', {
  community: 'hive-123456',
  props: {
    title: 'My Community',
    about: 'Short tagline',
    description: 'Full rules and description',
    is_nsfw: false,
    flag_text: 'Rules for flagging content',
    lang: 'en',
  },
}])
```

## Community Roles Hierarchy

| Role | Permissions |
|------|------------|
| `owner` | Full control, can set admins |
| `admin` | Manage mods, update props, all mod actions |
| `mod` | Pin/mute posts, mute users, set member roles |
| `member` | Post without approval (if community requires approval) |
| `guest` | Default role, can post (in open communities) |
| `muted` | Cannot interact with the community |

## Trending Topics (Tags)

```typescript
const topics = await client.call('bridge', 'get_trending_topics', {
  limit: 20,
  observer: 'username',
});
```
