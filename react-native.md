# React Native Hive Patterns

Patterns specific to building Hive dApps with React Native / Expo.

## dhive on React Native

### Setup

```bash
npm install @hiveio/dhive
# May need polyfills for crypto:
npm install react-native-get-random-values
```

```typescript
// Import polyfill BEFORE dhive
import 'react-native-get-random-values';
import { Client, PrivateKey } from '@hiveio/dhive';
```

### Manual Node Failover (REQUIRED)

dhive's built-in failover does NOT work reliably on React Native. See [battle-tested.md](battle-tested.md) for the manual rotation pattern.

## Secure Key Storage

**ALWAYS use `expo-secure-store`** for private keys. Never AsyncStorage.

```typescript
import * as SecureStore from 'expo-secure-store';

const STORE_OPTIONS = {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
};

// Per-account key storage
await SecureStore.setItemAsync(`account:${username}:postingKey`, key, STORE_OPTIONS);
const key = await SecureStore.getItemAsync(`account:${username}:postingKey`);
await SecureStore.deleteItemAsync(`account:${username}:postingKey`);
```

**Use `AsyncStorage` only for non-sensitive preferences:**

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';

// OK for preferences
await AsyncStorage.setItem('vote_weight', '75');
await AsyncStorage.setItem('selected_node', 'https://api.hive.blog');
```

## Image Handling

### Image Picker + iOS ph:// URIs

```typescript
import * as ImagePicker from 'expo-image-picker';
import * as ImageManipulator from 'expo-image-manipulator';

async function pickImage() {
  // Check permissions
  const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (status !== 'granted') {
    Alert.alert('Permission required', 'Camera roll access is needed');
    return null;
  }

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ImagePicker.MediaTypeOptions.Images,
    quality: 0.8,
  });

  if (result.canceled) return null;

  let uri = result.assets[0].uri;

  // iOS returns ph:// URIs — must normalize
  if (uri.startsWith('ph://')) {
    const normalized = await ImageManipulator.manipulateAsync(uri, [], {
      compress: 0.8,
      format: ImageManipulator.SaveFormat.JPEG,
    });
    uri = normalized.uri;
  }

  return uri;
}
```

### HEIC Conversion

```typescript
async function ensureJpeg(uri: string, mimeType: string): Promise<string> {
  if (mimeType === 'image/heic' || mimeType === 'image/heif') {
    const result = await ImageManipulator.manipulateAsync(uri, [], {
      compress: 0.8,
      format: ImageManipulator.SaveFormat.JPEG,
    });
    return result.uri;
  }
  // Preserve GIFs (don't recompress — kills animation)
  if (mimeType === 'image/gif') return uri;

  return uri;
}
```

### Upload to Hive Image Service

```typescript
async function uploadToHiveImages(uri: string, username: string, postingKey: string) {
  // Read file as buffer
  const response = await fetch(uri);
  const blob = await response.blob();

  // Create signature
  // Note: On RN, you may need a different approach for hashing
  // Consider using expo-crypto or a JS implementation

  const formData = new FormData();
  formData.append('file', {
    uri,
    type: 'image/jpeg',
    name: 'upload.jpg',
  } as any);

  const uploadResponse = await fetch(
    `https://images.hive.blog/${username}/${signature}`,
    { method: 'POST', body: formData }
  );

  const result = await uploadResponse.json();
  return result.url; // The hosted image URL
}
```

## Video Upload (3Speak)

```typescript
// Max duration enforced client-side
const MAX_VIDEO_DURATION = 120; // seconds

// Thumbnail generation
import { createThumbnail } from 'react-native-create-thumbnail';

const thumbnail = await createThumbnail({
  url: videoUri,
  timeStamp: 1000, // 1 second in
});
```

## Platform-Specific UI

```typescript
import { Platform, ActionSheetIOS, Alert } from 'react-native';

function showOptions(options: string[], onSelect: (index: number) => void) {
  if (Platform.OS === 'ios') {
    ActionSheetIOS.showActionSheetWithOptions(
      {
        options: [...options, 'Cancel'],
        cancelButtonIndex: options.length,
      },
      (buttonIndex) => {
        if (buttonIndex < options.length) onSelect(buttonIndex);
      }
    );
  } else {
    Alert.alert('Select Option', undefined,
      options.map((opt, i) => ({
        text: opt,
        onPress: () => onSelect(i),
      })).concat([{ text: 'Cancel', style: 'cancel' }])
    );
  }
}
```

## Networking Patterns

### Request with Timeout + Retry + Cache

```typescript
interface NetworkTarget {
  path: string;
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  body?: any;
  headers?: Record<string, string>;
  shouldCache?: boolean;
  timeoutMs?: number;
  retries?: number;
}

async function makeRequest<T>(target: NetworkTarget, baseUrl: string): Promise<T> {
  const timeout = target.timeoutMs || 10000;
  const maxRetries = target.retries ?? 0;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeout);

    try {
      const res = await fetch(baseUrl + target.path, {
        method: target.method,
        headers: target.headers,
        body: target.body ? JSON.stringify(target.body) : undefined,
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return await res.json();
    } catch (err) {
      clearTimeout(timeoutId);
      if (attempt < maxRetries) {
        await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 300));
        continue;
      }
      throw err;
    }
  }

  throw new Error('Unreachable');
}
```

## State Management

### Cache with Expiration (Redux/Zustand)

```typescript
interface CacheItem<T> {
  data: T;
  timestamp: number;
  expiresAt: number;
}

const CACHE_DURATIONS = {
  FOLLOWING_LIST: 5 * 60 * 1000,   // 5 minutes
  USER_PROFILE: 10 * 60 * 1000,    // 10 minutes
  NOTIFICATIONS: 2 * 60 * 1000,    // 2 minutes
  HIVE_POSTS: 1 * 60 * 1000,       // 1 minute
  HIVE_DATA: 2 * 60 * 1000,        // 2 minutes
};

function createCacheItem<T>(data: T, duration: number): CacheItem<T> {
  return { data, timestamp: Date.now(), expiresAt: Date.now() + duration };
}

function isCacheExpired<T>(item: CacheItem<T>): boolean {
  return Date.now() > item.expiresAt;
}
```

### Following Set for O(1) Lookups

```typescript
// Cache following list as a Set for fast filtering
const followingSet = useMemo(
  () => new Set(followingList.map(f => f.following)),
  [followingList]
);

// Filter feed efficiently
const filteredPosts = useMemo(
  () => posts.filter(p => followingSet.has(p.author)),
  [posts, followingSet]
);
```

## Optimistic Updates Pattern

```typescript
function useUpvote() {
  const [isVoting, setIsVoting] = useState(false);

  const vote = async (author: string, permlink: string, weight: number) => {
    setIsVoting(true);

    // 1. Optimistic update — UI responds immediately
    updatePostInState(author, permlink, {
      hasUpvoted: true,
      net_votes: currentVotes + 1,
    });

    try {
      // 2. Broadcast to chain
      const key = await SecureStore.getItemAsync(`account:${username}:postingKey`);
      await client.broadcast.vote({
        voter: username, author, permlink, weight: weight * 100,
      }, PrivateKey.fromString(key));
    } catch (err) {
      // 3. Rollback on failure
      updatePostInState(author, permlink, {
        hasUpvoted: false,
        net_votes: currentVotes,
      });
      Alert.alert('Vote Failed', err.message);
    } finally {
      setIsVoting(false);
    }
  };

  return { vote, isVoting };
}
```

## Real-Time: Polling Over WebSocket

On mobile, always prefer polling over WebSocket connections:

```typescript
function usePollFeed(tag: string, intervalMs = 30000) {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    let active = true;

    const poll = async () => {
      try {
        const latest = await client.call('bridge', 'get_ranked_posts', {
          sort: 'created', tag, limit: 20,
        });
        if (active) setPosts(latest);
      } catch {}
    };

    poll(); // Initial fetch
    const interval = setInterval(poll, intervalMs);

    return () => {
      active = false;
      clearInterval(interval);
    };
  }, [tag]);

  return posts;
}
```

## Push Notifications Architecture

For real-time on mobile, use FCM instead of WebSocket:

```
Send Message → Your API Server → MongoDB →
  Server streams Hive blocks (background) →
  Detects relevant op → Fires FCM push →
  Device receives push → Silent data fetch →
  Show local notification
```

This is more battery-efficient and survives app backgrounding.

## Multi-Account Pattern

```typescript
const MAX_ACCOUNTS = 10;

// Account list stored in AsyncStorage (just usernames)
// Keys stored in SecureStore (per account)

async function switchAccount(username: string) {
  // 1. Verify key still exists
  const key = await SecureStore.getItemAsync(`account:${username}:postingKey`);
  if (!key) throw new Error('Key not found — please re-login');

  // 2. Verify key still valid on chain
  const [account] = await client.database.getAccounts([username]);
  const pubKey = PrivateKey.fromString(key).createPublic().toString();
  const isValid = account.posting.key_auths.some(([pk]) => pk === pubKey);
  if (!isValid) throw new Error('Key no longer valid — account may have changed keys');

  // 3. Set as active account
  await AsyncStorage.setItem('active_account', username);
}
```
