# React Native + Hive Examples

## Complete Login Flow

```typescript
import * as SecureStore from 'expo-secure-store';
import { Client, PrivateKey } from '@hiveio/dhive';

const HIVE_NODES = [
  'https://api.hive.blog',
  'https://api.deathwing.me',
  'https://api.openhive.network',
];
const client = new Client(HIVE_NODES);

async function login(username: string, postingKeyWIF: string): Promise<boolean> {
  // 1. Validate username format
  if (!/^[a-z][a-z0-9\-.]{1,14}[a-z0-9]$/.test(username)) {
    throw new Error('Invalid Hive username format');
  }

  // 2. Verify account exists on chain
  const [account] = await client.database.getAccounts([username]);
  if (!account) throw new Error('Account not found on Hive');

  // 3. Derive public key and verify against on-chain authority
  let publicKey: string;
  try {
    publicKey = PrivateKey.fromString(postingKeyWIF).createPublic().toString();
  } catch {
    throw new Error('Invalid private key format');
  }

  const isAuthorized = account.posting.key_auths.some(
    ([pk]: [string, number]) => pk === publicKey
  );
  if (!isAuthorized) throw new Error('Key does not match account posting authority');

  // 4. Store securely
  await SecureStore.setItemAsync(`account:${username}:postingKey`, postingKeyWIF, {
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  });
  await SecureStore.setItemAsync('active_username', username);

  return true;
}
```

## useHivePost Hook

```typescript
import { useState, useEffect } from 'react';
import { Client } from '@hiveio/dhive';

const client = new Client([
  'https://api.hive.blog',
  'https://api.deathwing.me',
]);

interface HivePost {
  author: string;
  permlink: string;
  title: string;
  body: string;
  created: string;
  net_votes: number;
  pending_payout_value: string;
  json_metadata: any;
  replies: HivePost[];
}

export function useHivePost(author: string, permlink: string) {
  const [post, setPost] = useState<HivePost | null>(null);
  const [replies, setReplies] = useState<HivePost[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let active = true;

    async function fetchPost() {
      try {
        setLoading(true);
        const [postData, replyData] = await Promise.all([
          client.call('bridge', 'get_post', { author, permlink }),
          client.database.call('get_content_replies', [author, permlink]),
        ]);

        if (!active) return;

        if (postData) {
          setPost({
            ...postData,
            json_metadata: typeof postData.json_metadata === 'string'
              ? JSON.parse(postData.json_metadata)
              : postData.json_metadata,
          });
        }

        // Sort replies by payout (highest first)
        const sorted = (replyData || []).sort((a, b) => {
          const payA = parseFloat(a.pending_payout_value?.replace(' HBD', '') || '0');
          const payB = parseFloat(b.pending_payout_value?.replace(' HBD', '') || '0');
          return payB - payA;
        });
        setReplies(sorted);
      } catch (err: any) {
        if (active) setError(err.message);
      } finally {
        if (active) setLoading(false);
      }
    }

    if (author && permlink) fetchPost();
    return () => { active = false; };
  }, [author, permlink]);

  return { post, replies, loading, error };
}
```

## useVote Hook with Optimistic Updates

```typescript
import { useState, useCallback } from 'react';
import * as SecureStore from 'expo-secure-store';
import { Client, PrivateKey } from '@hiveio/dhive';
import AsyncStorage from '@react-native-async-storage/async-storage';

const client = new Client([
  'https://api.hive.blog',
  'https://api.deathwing.me',
]);

export function useVote(onUpdatePost?: (author: string, permlink: string, updates: any) => void) {
  const [isVoting, setIsVoting] = useState(false);

  const vote = useCallback(async (
    author: string,
    permlink: string,
    weightPercent: number, // 1-100
  ) => {
    if (isVoting) return;
    setIsVoting(true);

    const username = await SecureStore.getItemAsync('active_username');
    const keyWIF = await SecureStore.getItemAsync(`account:${username}:postingKey`);
    if (!username || !keyWIF) throw new Error('Not logged in');

    // Optimistic update
    onUpdatePost?.(author, permlink, {
      hasUpvoted: true,
      upvoteWeight: weightPercent,
    });

    try {
      const weight = Math.min(10000, Math.max(1, Math.round(weightPercent * 100)));
      await client.broadcast.vote(
        { voter: username, author, permlink, weight },
        PrivateKey.fromString(keyWIF)
      );

      // Persist preferred weight
      await AsyncStorage.setItem('vote_weight', String(weightPercent));
    } catch (err) {
      // Rollback
      onUpdatePost?.(author, permlink, {
        hasUpvoted: false,
        upvoteWeight: 0,
      });
      throw err;
    } finally {
      setIsVoting(false);
    }
  }, [isVoting, onUpdatePost]);

  return { vote, isVoting };
}
```

## useResourceCredits Hook

```typescript
import { useState, useEffect } from 'react';

export function useResourceCredits(username: string | null) {
  const [rcPercent, setRcPercent] = useState(100);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!username) return;

    let active = true;

    async function fetchRC() {
      setLoading(true);
      try {
        const response = await client.call('rc_api', 'find_rc_accounts', {
          accounts: [username],
        });

        if (!active) return;

        if (response?.rc_accounts?.[0]) {
          const rc = response.rc_accounts[0];
          const current = parseFloat(rc.rc_manabar.current_mana);
          const max = parseFloat(rc.max_rc);
          setRcPercent(Math.max(0, Math.min(100, (current / max) * 100)));
        }
      } catch {} finally {
        if (active) setLoading(false);
      }
    }

    fetchRC();
    const interval = setInterval(fetchRC, 60000); // Refresh every minute

    return () => { active = false; clearInterval(interval); };
  }, [username]);

  return { rcPercent, loading, canTransact: rcPercent > 5 };
}
```

## Reply Component Example

```typescript
import React, { useState } from 'react';
import { View, TextInput, Button, Alert } from 'react-native';
import * as SecureStore from 'expo-secure-store';
import { Client, PrivateKey } from '@hiveio/dhive';

const client = new Client(['https://api.hive.blog', 'https://api.deathwing.me']);

interface ReplyProps {
  parentAuthor: string;
  parentPermlink: string;
  onSuccess?: () => void;
}

export function ReplyBox({ parentAuthor, parentPermlink, onSuccess }: ReplyProps) {
  const [text, setText] = useState('');
  const [posting, setPosting] = useState(false);

  const submitReply = async () => {
    if (!text.trim() || posting) return;
    setPosting(true);

    try {
      const username = await SecureStore.getItemAsync('active_username');
      const keyWIF = await SecureStore.getItemAsync(`account:${username}:postingKey`);

      const sanitizedParent = parentAuthor.toLowerCase().replace(/[^a-z0-9-]/g, '');
      const permlink = `re-${sanitizedParent}-${parentPermlink}-${Date.now()}`;

      await client.broadcast.comment({
        parent_author: parentAuthor,
        parent_permlink: parentPermlink,
        author: username!,
        permlink,
        title: '',
        body: text.trim(),
        json_metadata: JSON.stringify({
          app: 'myapp/1.0',
          format: 'markdown',
          tags: ['reply'],
        }),
      }, PrivateKey.fromString(keyWIF!));

      setText('');
      onSuccess?.();
    } catch (err: any) {
      Alert.alert('Failed to post reply', err.message);
    } finally {
      setPosting(false);
    }
  };

  return (
    <View>
      <TextInput
        value={text}
        onChangeText={setText}
        placeholder="Write a reply..."
        multiline
      />
      <Button
        title={posting ? 'Posting...' : 'Reply'}
        onPress={submitReply}
        disabled={!text.trim() || posting}
      />
    </View>
  );
}
```
