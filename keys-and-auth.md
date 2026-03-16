# Keys, Authentication & Signing

## Key Hierarchy

Hive uses a hierarchical key system. Each key has different permissions:

| Key | Authority Level | Used For |
|-----|----------------|----------|
| **Owner** | Highest | Account recovery, changing other keys. Store OFFLINE. |
| **Active** | High | Financial ops: transfers, power up/down, delegations, savings |
| **Posting** | Medium | Social ops: post, comment, vote, follow, reblog, custom_json |
| **Memo** | Read-only | Encrypting/decrypting private memos on transfers |

### Key Format

- **Private keys:** WIF format, starts with `5` (51 chars, base58check encoded)
- **Public keys:** Starts with `STM` (53 chars)

### Deriving Public from Private

```typescript
import { PrivateKey } from '@hiveio/dhive';

const privateKey = PrivateKey.fromString('5J...');
const publicKey = privateKey.createPublic().toString(); // "STM..."
```

### Validating a Key Against On-Chain Authority

```typescript
const [account] = await client.database.getAccounts(['username']);
const derivedPublic = PrivateKey.fromString(postingKeyWIF).createPublic().toString();

// Check if this public key is in the account's posting authorities
const isValid = account.posting.key_auths.some(([pubKey]) => pubKey === derivedPublic);

// Same pattern for active key:
const isActiveValid = account.active.key_auths.some(([pubKey]) => pubKey === derivedPublic);
```

## Username Validation

```typescript
// 3-16 characters, starts with letter, allows dots and hyphens
// Dots permitted for legacy accounts (e.g. smooth.witness)
const HIVE_USERNAME_REGEX = /^[a-z][a-z0-9\-.]{1,14}[a-z0-9]$/;

function isValidUsername(username: string): boolean {
  return HIVE_USERNAME_REGEX.test(username);
}

// Verify account exists on chain
async function accountExists(username: string): Promise<boolean> {
  const accounts = await client.database.getAccounts([username]);
  return accounts.length > 0 && accounts[0].name === username;
}
```

## Authentication Methods

### 1. Direct Key Signing (dhive)

Simplest method — app holds the private key.

```typescript
import { PrivateKey } from '@hiveio/dhive';

const postingKey = PrivateKey.fromString('5J...');
await client.broadcast.vote(voteData, postingKey);
```

### 2. Challenge-Response Authentication

For backend auth — prove key ownership without exposing the key to the server.

```typescript
// Step 1: Server generates a random challenge
// Step 2: Client signs it with their Hive key
// Step 3: Server verifies signature against on-chain public key

// CLIENT SIDE:
import { ec as EC } from 'elliptic';
import { sha256 } from 'js-sha256';
import bs58 from 'bs58';

function signChallenge(username, challenge, timestamp, postingKeyWIF) {
  const ec = new EC('secp256k1');

  // Decode WIF to raw private key bytes
  const decoded = bs58.decode(postingKeyWIF);
  const privateKeyBytes = decoded.slice(1, 33); // Skip 0x80 prefix and checksum
  const key = ec.keyFromPrivate(privateKeyBytes);

  // Construct and hash the message
  const message = `${username}:${challenge}:${timestamp}`;
  const hashHex = sha256(message).toString();
  const hashBuffer = Buffer.from(hashHex, 'hex');

  // Sign with canonical form
  const sigObj = key.sign(hashBuffer, { canonical: true });

  // Pack signature: r(32) + s(32) + recoveryParam(1)
  return Buffer.concat([
    sigObj.r.toArrayLike(Buffer, 'be', 32),
    sigObj.s.toArrayLike(Buffer, 'be', 32),
    Buffer.from([sigObj.recoveryParam ?? 0]),
  ]).toString('hex');
}
```

### 3. Hive Keychain (Browser Extension)

```typescript
// Check if Keychain is available
const hasKeychain = !!window.hive_keychain;

// Request broadcast
window.hive_keychain.requestBroadcast(
  'username',
  [operations],
  'Posting', // or 'Active'
  (response) => {
    if (response.success) {
      console.log('Transaction ID:', response.result.id);
    } else {
      console.error('Keychain error:', response.message);
    }
  }
);

// Request signature (for auth challenges)
window.hive_keychain.requestSignBuffer(
  'username',
  challengeString,
  'Posting',
  (response) => {
    if (response.success) {
      const signature = response.result;
    }
  }
);

// Custom JSON via Keychain
window.hive_keychain.requestCustomJson(
  'username',
  'follow',                    // id
  'Posting',                   // key type
  JSON.stringify(followPayload),
  'Follow @someone',           // display message
  (response) => { ... }
);

// Detect in-app browsers
const isKeychainWebview = !!(
  window.hive_keychain ||
  window.__KEYCHAIN_WEBVIEW__ ||
  window.ReactNativeWebView
);
```

### 4. HiveAuth (Mobile Auth via WebSocket)

```typescript
// Connect to HiveAuth server
const ws = new WebSocket('wss://hive-auth.arcange.eu');

// Session management
const session = {
  token: generateToken(),
  key: generateEncryptionKey(),
  expiry: Date.now() + (14 * 24 * 60 * 60 * 1000), // 2 weeks
};

// Nonce for request uniqueness (prevents replay)
let nonceCounter = 0;
function generateNonce() {
  return `${Date.now()}_${++nonceCounter}`;
}

// IMPORTANT: Use single-flight connection guard
// Browser visibility events can fire multiple connect attempts
let connecting = false;
async function ensureConnected() {
  if (connecting) return;
  connecting = true;
  try {
    await connect();
  } finally {
    connecting = false;
  }
}
```

### 5. HiveSigner (OAuth)

```typescript
import hivesigner from 'hivesigner';

const hsClient = new hivesigner.Client({
  app: 'your-app-name',
  callbackURL: 'https://your-app.com/callback',
  scope: ['vote', 'comment', 'delete_comment', 'comment_options',
          'custom_json', 'claim_reward_balance', 'offline'],
});

// Generate auth URL
const authUrl = hsClient.getLoginURL();

// After callback, set access token
hsClient.setAccessToken(accessToken);

// Broadcast via HiveSigner
await hsClient.vote('voter', 'author', 'permlink', 10000);
await hsClient.comment(parentAuthor, parentPermlink, author, permlink, title, body, jsonMetadata);

// Token validation: max 7 days old
function isTokenValid(token) {
  const age = Date.now() - token.issuedAt;
  return age < 7 * 24 * 60 * 60 * 1000;
}
```

## Broadcast Fallback Chain

For maximum compatibility, try auth methods in order:

1. **HiveSigner token** (if available and valid)
2. **Hive Keychain** (browser extension)
3. **HiveAuth** (mobile)
4. **Direct key signing** (if key stored locally)

```typescript
async function broadcast(ops, keyType) {
  // 1. Try HiveSigner
  if (hasValidToken()) {
    try { return await broadcastWithHiveSigner(ops); } catch {}
  }

  // 2. Try Keychain
  if (window.hive_keychain) {
    try { return await broadcastWithKeychain(ops, keyType); } catch {}
  }

  // 3. Try HiveAuth
  if (hiveAuthSession) {
    try { return await broadcastWithHiveAuth(ops, keyType); } catch {}
  }

  // 4. Direct key (last resort)
  const key = await getStoredKey(keyType);
  if (key) {
    return await client.broadcast.sendOperations(ops, PrivateKey.fromString(key));
  }

  throw new Error('No auth method available');
}
```

## Secure Key Storage (React Native)

```typescript
import * as SecureStore from 'expo-secure-store';

const STORE_OPTIONS = {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
};

// Store key
await SecureStore.setItemAsync(
  `account:${username}:postingKey`,
  privateKeyWIF,
  STORE_OPTIONS
);

// Retrieve key
const key = await SecureStore.getItemAsync(`account:${username}:postingKey`);

// Delete key
await SecureStore.deleteItemAsync(`account:${username}:postingKey`);

// NEVER store keys in AsyncStorage, localStorage, or plain files
// ALWAYS use OS-level encrypted storage
```

## Multi-Account Support

```typescript
// Per-account key storage pattern
const postingKeyKey = (username: string) => `account:${username}:postingKey`;
const activeKeyKey = (username: string) => `account:${username}:activeKey`;

// Account list management
const MAX_ACCOUNTS = 10;

async function addAccount(username: string, postingKey: string) {
  // 1. Validate username format
  if (!HIVE_USERNAME_REGEX.test(username)) throw new Error('Invalid username');

  // 2. Verify account exists
  const [account] = await client.database.getAccounts([username]);
  if (!account) throw new Error('Account not found');

  // 3. Verify key matches on-chain
  const derivedPublic = PrivateKey.fromString(postingKey).createPublic().toString();
  const isValid = account.posting.key_auths.some(([pk]) => pk === derivedPublic);
  if (!isValid) throw new Error('Key does not match account');

  // 4. Store securely
  await SecureStore.setItemAsync(postingKeyKey(username), postingKey, STORE_OPTIONS);
}
```
