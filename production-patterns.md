# Production Patterns for Hive dApps

Advanced patterns for production Hive applications, covering multi-account posting, rate limit recovery, content integrity, deduplication, post editing, container/thread architectures, challenge-response auth, and testing strategies.

These patterns complement `battle-tested.md` with higher-level architectural approaches.

---

## 1. Multi-Account Posting Rotation

When a dApp needs to post frequently (e.g., an archival bot, oracle, or feed aggregator), a single account quickly hits Hive's rate limits. The solution is a **satellite account pool** where a central key delegates posting authority to multiple accounts.

### Setup: Delegate Posting Authority

```typescript
import { Client, PrivateKey } from "@hiveio/dhive";

// Grant posting authority from satellite account to your central bot account.
// Run once per satellite account using the satellite's ACTIVE key.
const client = new Client(["https://api.hive.blog"]);

const op = [
  "account_update2",
  {
    account: "botaccount1",
    json_metadata: "",
    posting_json_metadata: "",
    extensions: [],
    posting: {
      weight_threshold: 1,
      account_auths: [["centralbot", 1]], // centralbot can now post as botaccount1
      key_auths: [["STM...", 1]], // keep the satellite's own posting key
    },
  },
];

await client.broadcast.sendOperations([op], PrivateKey.from("satellite-active-key"));
```

### Round-Robin Selection with Cooldown Tracking

```typescript
const ROOT_POST_COOLDOWN_MS = 5 * 60 * 1000 + 10_000; // 5min 10s (safety margin)
const COMMENT_COOLDOWN_MS = 21_000; // 21 seconds

interface PostingAccount {
  id: string;
  username: string;
  postingKeyEnvVar: string; // e.g., "BOT1_POSTING_KEY"
  lastRootPostAt: Date | null;
  lastCommentAt: Date | null;
  enabled: boolean;
}

function selectAccountForComment(
  accounts: PostingAccount[],
  excludeIds: string[] = []
): PostingAccount | null {
  const now = Date.now();
  for (const acct of accounts) {
    if (!acct.enabled) continue;
    if (excludeIds.includes(acct.id)) continue;
    const lastComment = acct.lastCommentAt ? acct.lastCommentAt.getTime() : 0;
    if (now - lastComment >= COMMENT_COOLDOWN_MS) {
      return acct;
    }
  }
  return null; // All accounts are on cooldown
}

function selectAccountForRootPost(
  accounts: PostingAccount[],
  excludeIds: string[] = []
): PostingAccount | null {
  const now = Date.now();
  for (const acct of accounts) {
    if (!acct.enabled) continue;
    if (excludeIds.includes(acct.id)) continue;
    const lastPost = acct.lastRootPostAt ? acct.lastRootPostAt.getTime() : 0;
    if (now - lastPost >= ROOT_POST_COOLDOWN_MS) {
      return acct;
    }
  }
  return null;
}
```

### Recursive Retry on Missing Authority

If a satellite account's authority has been revoked (or was never set up), automatically try the next account:

```typescript
async function postWithFallback(
  content: string,
  parentAuthor: string,
  parentPermlink: string,
  accounts: PostingAccount[],
  excludeIds: string[] = []
): Promise<{ author: string; permlink: string }> {
  const account = selectAccountForComment(accounts, excludeIds);
  if (!account) {
    throw new Error("No posting account available (all on cooldown or excluded)");
  }

  const postingKey = process.env[account.postingKeyEnvVar];
  if (!postingKey) {
    throw new Error(`Env var ${account.postingKeyEnvVar} not set for @${account.username}`);
  }

  const permlink = `re-${parentPermlink}-${Date.now()}`.toLowerCase().replace(/[^a-z0-9-]/g, "-");

  try {
    const key = PrivateKey.fromString(postingKey);
    await client.broadcast.sendOperations(
      [["comment", {
        parent_author: parentAuthor,
        parent_permlink: parentPermlink,
        author: account.username,
        permlink,
        title: "",
        body: content,
        json_metadata: JSON.stringify({ app: "myapp/1.0" }),
      }]],
      key
    );

    // Update cooldown tracking
    account.lastCommentAt = new Date();
    return { author: account.username, permlink };
  } catch (error) {
    const msg = (error as Error).message || "";

    if (msg.includes("Missing Posting Authority") || msg.includes("missing required posting authority")) {
      console.warn(`@${account.username} lacks posting authority, trying next account...`);
      return postWithFallback(content, parentAuthor, parentPermlink, accounts, [...excludeIds, account.id]);
    }

    throw error;
  }
}
```

---

## 2. Rate Limit Error Detection & Recovery

Hive returns specific error strings for rate limits. A production app must detect these, distinguish recoverable from fatal errors, and schedule retries.

### Error String Catalog

```typescript
function classifyBroadcastError(errorMsg: string): {
  type: "comment_rate_limit" | "root_rate_limit" | "missing_authority" | "insufficient_rc" | "fatal";
  retryable: boolean;
  retryAfterMs: number;
} {
  // Comment rate limit (21s between comments per account)
  if (
    errorMsg.includes("STEEMIT_MIN_REPLY_INTERVAL") ||
    errorMsg.includes("You may only comment once every")
  ) {
    return { type: "comment_rate_limit", retryable: true, retryAfterMs: 21_000 };
  }

  // Root post rate limit (5 minutes between root posts per account)
  if (
    errorMsg.includes("STEEMIT_MIN_ROOT_COMMENT_INTERVAL") ||
    errorMsg.includes("You may only post once every 5 minutes")
  ) {
    return { type: "root_rate_limit", retryable: true, retryAfterMs: 5 * 60 * 1000 + 10_000 };
  }

  // Missing authority — account fallback, not a time-based retry
  if (
    errorMsg.includes("Missing Posting Authority") ||
    errorMsg.includes("missing required posting authority")
  ) {
    return { type: "missing_authority", retryable: true, retryAfterMs: 0 };
  }

  // Insufficient Resource Credits — wait for RC regeneration
  if (
    errorMsg.includes("Please wait to transact, or power up") ||
    errorMsg.includes("Bandwidth limit exceeded") ||
    errorMsg.includes("insufficient RC")
  ) {
    return { type: "insufficient_rc", retryable: true, retryAfterMs: 60 * 1000 };
  }

  // Everything else is fatal
  return { type: "fatal", retryable: false, retryAfterMs: 0 };
}
```

### Scheduling Retries with `retryAfter`

```typescript
// After catching a rate limit error:
const classification = classifyBroadcastError(error.message);

if (classification.retryable && classification.retryAfterMs > 0) {
  const retryAfter = new Date(Date.now() + classification.retryAfterMs);
  await db.updatePost(postId, {
    status: "pending",
    retryAfter,
    errorMessage: `Rate limited — will retry after ${retryAfter.toISOString()}`,
  });
}

// In your scheduler loop, only process posts where retryAfter has passed:
const readyPosts = await db.getPendingPosts({ retryAfterBefore: new Date() });
```

### Rate Limit Cooldown State Tracking

For global rate limit tracking (e.g., when an external API rate-limits your entire app):

```typescript
let cooldownUntil = 0;
let consecutiveHits = 0;

function recordRateLimit(baseCooldownMs: number = 5 * 60 * 1000): void {
  consecutiveHits++;
  cooldownUntil = Date.now() + baseCooldownMs; // Flat cooldown, no escalation
}

function isCoolingDown(): boolean {
  return Date.now() < cooldownUntil;
}

function cooldownRemainingMs(): number {
  return Math.max(0, cooldownUntil - Date.now());
}

function resetCooldown(): void {
  cooldownUntil = 0;
  consecutiveHits = 0;
}
```

---

## 3. Content Integrity Hashing

For archival or proof-of-existence dApps, store content hashes in `json_metadata` so anyone can verify the content hasn't been tampered with.

### Multi-Algorithm Hashing

```typescript
import { createHash } from "crypto";

interface ContentHashes {
  sha256: string;
  blake2b: string;
  md5: string;
}

function computeContentHashes(content: string, mediaUrls?: string[] | null): ContentHashes {
  // Build canonical form: content + sorted media URLs
  let canonical = content;
  if (mediaUrls && mediaUrls.length > 0) {
    canonical += "\n" + mediaUrls.sort().join("\n");
  }

  const sha256 = createHash("sha256").update(canonical, "utf8").digest("hex");
  const blake2b = createHash("blake2b512").update(canonical, "utf8").digest("hex");
  const md5 = createHash("md5").update(canonical, "utf8").digest("hex");

  return { sha256, blake2b, md5 };
}
```

### Storing Hashes in `json_metadata`

```typescript
const hashes = computeContentHashes(postContent, mediaUrls);

const jsonMetadata = {
  tags: ["myapp", "archive"],
  app: "myapp/1.0",
  format: "markdown",
  content_hashes: {
    sha256: hashes.sha256,
    blake2b: hashes.blake2b,
    md5: hashes.md5,
  },
  hash_of: "post_content", // Describes what was hashed
};
```

### Verification Pattern

Third parties can verify content integrity by re-computing hashes from the post body:

```typescript
async function verifyPostIntegrity(author: string, permlink: string): Promise<boolean> {
  const post = await client.call("bridge", "get_post", { author, permlink });
  if (!post) return false;

  const metadata = JSON.parse(post.json_metadata || "{}");
  if (!metadata.content_hashes) return false;

  const mediaUrls = metadata.image || [];
  const recomputed = computeContentHashes(post.body, mediaUrls);

  return (
    recomputed.sha256 === metadata.content_hashes.sha256 &&
    recomputed.blake2b === metadata.content_hashes.blake2b &&
    recomputed.md5 === metadata.content_hashes.md5
  );
}
```

---

## 4. Deduplication on Hive

Before broadcasting, check if the content already exists on-chain to avoid duplicate posts.

### Pre-Broadcast Deduplication

```typescript
async function isAlreadyPosted(contentId: string): Promise<{
  exists: boolean;
  author?: string;
  permlink?: string;
}> {
  // Check your local database first (fastest)
  const existing = await db.findPostByContentId(contentId);
  if (existing && existing.hivePermlink) {
    return { exists: true, author: existing.hiveAuthor, permlink: existing.hivePermlink };
  }

  // Optionally check a secondary table (e.g., selective/manual archives)
  const secondary = await db.findSecondaryByContentId(contentId);
  if (secondary && secondary.hivePermlink) {
    return { exists: true, author: secondary.hiveAuthor, permlink: secondary.hivePermlink };
  }

  return { exists: false };
}
```

### Idempotent Handling

When a duplicate is found, mark it as "posted" without consuming quota or broadcasting:

```typescript
const dedup = await isAlreadyPosted(contentId);
if (dedup.exists) {
  console.log(`[DEDUP] Content ${contentId} already archived at @${dedup.author}/${dedup.permlink}`);

  // Mark as posted with reference to original — no broadcast needed
  await db.updatePost(postId, {
    status: "posted",
    hiveAuthor: dedup.author,
    hivePermlink: dedup.permlink,
    errorMessage: `Deduplicated — already archived via @${dedup.author}/${dedup.permlink}`,
    postedAt: new Date(),
  });

  return { permlink: dedup.permlink!, author: dedup.author!, deduplicated: true };
}
```

---

## 5. Post Editing Within 7-Day Window

Hive allows editing posts within 7 days of creation. Re-broadcast the same `author` + `permlink` with the updated body.

### Safety Margin

```typescript
const EDIT_WINDOW_MS = 6.5 * 24 * 60 * 60 * 1000; // 6.5 days (not 7 — safety margin)

function canEdit(postedAt: Date): boolean {
  return Date.now() - postedAt.getTime() < EDIT_WINDOW_MS;
}
```

### Editing a Post

```typescript
async function editPost(
  author: string,
  permlink: string,
  parentAuthor: string,
  parentPermlink: string,
  newBody: string,
  newMetadata: Record<string, any>,
  postingKey: string
): Promise<{ success: boolean; error?: string }> {
  try {
    const key = PrivateKey.fromString(postingKey);
    await client.broadcast.sendOperations(
      [["comment", {
        parent_author: parentAuthor,
        parent_permlink: parentPermlink,
        author,
        permlink, // Same permlink = edit, not new post
        title: "",
        body: newBody,
        json_metadata: JSON.stringify(newMetadata),
      }]],
      key
    );
    return { success: true };
  } catch (error) {
    return { success: false, error: (error as Error).message };
  }
}
```

### Batch Editing with Rate Limit Respect

```typescript
async function batchEdit(
  posts: Array<{ author: string; permlink: string; parentAuthor: string; parentPermlink: string; newBody: string; metadata: Record<string, any>; postedAt: Date }>,
  postingKey: string
): Promise<{ updated: number; skipped: number; errors: number }> {
  let updated = 0, skipped = 0, errors = 0;

  for (const post of posts) {
    if (!canEdit(post.postedAt)) {
      skipped++;
      continue;
    }

    const result = await editPost(
      post.author, post.permlink,
      post.parentAuthor, post.parentPermlink,
      post.newBody, post.metadata,
      postingKey
    );

    if (result.success) {
      updated++;
    } else {
      errors++;
    }

    // Respect Hive rate limits — 3s between edits
    await new Promise(r => setTimeout(r, 3000));
  }

  return { updated, skipped, errors };
}
```

---

## 6. Container/Thread Pattern

For high-volume data storage on Hive, use a **root post as a container** with individual items as comment replies. This organizes content, avoids root post rate limits, and enables volume-based pagination.

### Container Creation

```typescript
function generateContainerPermlink(prefix: string, volume: number): string {
  const timestamp = Date.now();
  return `${prefix}-vol${volume}-${timestamp}`.toLowerCase().replace(/[^a-z0-9-]/g, "-");
}

async function createContainer(
  account: PostingAccount,
  prefix: string,
  volume: number,
  tags: string[]
): Promise<{ author: string; permlink: string }> {
  const permlink = generateContainerPermlink(prefix, volume);
  const key = PrivateKey.fromString(process.env[account.postingKeyEnvVar]!);

  const operations = [
    ["comment", {
      parent_author: "",
      parent_permlink: tags[0], // First tag = category
      author: account.username,
      permlink,
      title: `${prefix} Archive — Volume ${volume}`,
      body: `# ${prefix} Archive — Volume ${volume}\n\nItems archived as comments below.`,
      json_metadata: JSON.stringify({
        tags,
        app: "myapp/1.0",
        format: "markdown",
        type: "data_container",
        volume,
      }),
    }],
    // Zero payout — this is data storage, not content for rewards
    ["comment_options", {
      author: account.username,
      permlink,
      max_accepted_payout: "0.000 HBD",
      percent_hbd: 10000,
      allow_votes: true,
      allow_curation_rewards: true,
      extensions: [],
    }],
  ];

  await client.broadcast.sendOperations(operations, key);
  return { author: account.username, permlink };
}
```

### Volume-Based Rotation

```typescript
const MAX_COMMENTS_PER_CONTAINER = 200;

async function ensureContainer(prefix: string): Promise<{ author: string; permlink: string }> {
  // Check if active container exists and has capacity
  const active = await db.getActiveContainer(prefix);

  if (active && active.commentCount < MAX_COMMENTS_PER_CONTAINER) {
    return { author: active.author, permlink: active.permlink };
  }

  // Need a new container — calculate next volume number
  const existingContainers = await db.getContainers(prefix);
  const nextVolume = existingContainers.length > 0
    ? existingContainers[0].volume + 1
    : 1;

  const account = selectAccountForRootPost(accounts);
  if (!account) {
    throw new Error("No posting account available for root post");
  }

  const container = await createContainer(account, prefix, nextVolume, ["myapp", "archive"]);

  // Save to DB for tracking
  await db.createContainer({
    prefix,
    author: container.author,
    permlink: container.permlink,
    volume: nextVolume,
    maxComments: MAX_COMMENTS_PER_CONTAINER,
    commentCount: 0,
  });

  return container;
}
```

### Posting Items as Comments

```typescript
async function postItemToContainer(
  content: string,
  metadata: Record<string, any>,
  containerAuthor: string,
  containerPermlink: string,
  accounts: PostingAccount[]
): Promise<{ author: string; permlink: string }> {
  const account = selectAccountForComment(accounts);
  if (!account) {
    throw new Error("No posting account available for comment");
  }

  const permlink = `re-${containerPermlink.slice(0, 20)}-${Date.now()}`
    .toLowerCase()
    .replace(/[^a-z0-9-]/g, "-");

  const key = PrivateKey.fromString(process.env[account.postingKeyEnvVar]!);

  await client.broadcast.sendOperations(
    [["comment", {
      parent_author: containerAuthor,
      parent_permlink: containerPermlink,
      author: account.username,
      permlink,
      title: "",
      body: content,
      json_metadata: JSON.stringify({
        ...metadata,
        app: "myapp/1.0",
      }),
    }],
    ["comment_options", {
      author: account.username,
      permlink,
      max_accepted_payout: "0.000 HBD",
      percent_hbd: 10000,
      allow_votes: true,
      allow_curation_rewards: true,
      extensions: [],
    }]],
    key
  );

  // Update tracking
  account.lastCommentAt = new Date();
  await db.incrementContainerCommentCount(containerPermlink);

  return { author: account.username, permlink };
}
```

---

## 7. Challenge-Response Auth with Server-Side Cache Fallback

Hive Keychain and HiveSigner use challenge-response authentication. A production server needs both session storage and an in-memory cache to handle edge cases (session store outages, clustered deployments).

### Dual-Store Challenge Management

```typescript
import crypto from "crypto";

interface ChallengeEntry {
  challenge: string;
  createdAt: number;
  used: boolean;
}

const challengeCache = new Map<string, ChallengeEntry>();
const MAX_CACHE_ENTRIES = 5000; // Hard cap to prevent memory exhaustion

function generateChallenge(): string {
  const challenge = crypto.randomBytes(32).toString("hex");

  // Evict oldest if at capacity
  if (challengeCache.size >= MAX_CACHE_ENTRIES) {
    const oldestKey = challengeCache.keys().next().value;
    if (oldestKey) challengeCache.delete(oldestKey);
  }

  challengeCache.set(challenge, {
    challenge,
    createdAt: Date.now(),
    used: false,
  });

  return challenge;
}
```

### One-Time-Use Verification

```typescript
const CHALLENGE_TTL_MS = 5 * 60 * 1000; // 5 minutes

function verifyAndConsumeChallenge(challenge: string): boolean {
  const entry = challengeCache.get(challenge);
  if (!entry) return false;

  // Check expiry
  if (Date.now() - entry.createdAt > CHALLENGE_TTL_MS) {
    challengeCache.delete(challenge);
    return false;
  }

  // One-time use — prevent replay attacks
  if (entry.used) return false;

  entry.used = true;
  challengeCache.delete(challenge); // Clean up after use
  return true;
}
```

### Periodic Cleanup

```typescript
// Run every 10 minutes to prevent stale entries accumulating
setInterval(() => {
  const now = Date.now();
  for (const [key, entry] of challengeCache) {
    if (now - entry.createdAt > CHALLENGE_TTL_MS) {
      challengeCache.delete(key);
    }
  }
}, 10 * 60 * 1000);
```

---

## 8. Testing Hive Patterns

Generic test patterns for Hive dApps using Vitest. These tests validate Hive-specific behaviors without requiring a live blockchain connection.

### Permlink Format Validation

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";

describe("generatePermlink", () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2025-06-15T12:00:00.000Z"));
  });
  afterEach(() => vi.useRealTimers());

  it("returns a lowercase string", () => {
    const result = generatePermlink("RE-Archive", "POST123");
    expect(result).toBe(result.toLowerCase());
  });

  it("contains prefix, id, and timestamp segments", () => {
    const result = generatePermlink("re-data", "item-456");
    const ts = Date.now();
    expect(result).toContain("re-data");
    expect(result).toContain("item-456");
    expect(result).toContain(String(ts));
  });

  it("replaces special characters with dashes", () => {
    const result = generatePermlink("my_prefix!", "item@id#1");
    expect(result).not.toMatch(/[^a-z0-9-]/);
  });

  it("uses the current timestamp controlled by fake timers", () => {
    const now = Date.now();
    const result = generatePermlink("p", "t");
    expect(result).toBe(`p-t-${now}`);
  });
});
```

### Content Hash Determinism

```typescript
describe("computeContentHashes", () => {
  it("returns hex strings of correct lengths (SHA256=64, BLAKE2b=128, MD5=32)", () => {
    const hashes = computeContentHashes("Hello world");
    expect(hashes.sha256).toHaveLength(64);
    expect(hashes.blake2b).toHaveLength(128);
    expect(hashes.md5).toHaveLength(32);
  });

  it("is deterministic — same input produces same output", () => {
    const first = computeContentHashes("deterministic test");
    const second = computeContentHashes("deterministic test");
    expect(first).toEqual(second);
  });

  it("null mediaUrls and empty array produce identical hashes", () => {
    const hashesNull = computeContentHashes("same", null);
    const hashesEmpty = computeContentHashes("same", []);
    expect(hashesNull).toEqual(hashesEmpty);
  });

  it("includes media URLs in the hash when populated", () => {
    const without = computeContentHashes("hello", null);
    const withMedia = computeContentHashes("hello", ["https://img.example.com/a.jpg"]);
    expect(without.sha256).not.toBe(withMedia.sha256);
  });

  it("sorts media URLs so order does not matter", () => {
    const hashesA = computeContentHashes("test", ["https://a.com/1.jpg", "https://b.com/2.jpg"]);
    const hashesB = computeContentHashes("test", ["https://b.com/2.jpg", "https://a.com/1.jpg"]);
    expect(hashesA).toEqual(hashesB);
  });

  it("handles unicode content correctly", () => {
    const hashes = computeContentHashes("Hello world! Decentralised!");
    expect(hashes.sha256).toMatch(/^[a-f0-9]{64}$/);
    expect(hashes.blake2b).toMatch(/^[a-f0-9]{128}$/);
    expect(hashes.md5).toMatch(/^[a-f0-9]{32}$/);
  });
});
```

### Account Rotation Cooldown Enforcement

```typescript
describe("selectAccountForComment", () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2025-06-15T12:00:00.000Z"));
  });
  afterEach(() => vi.useRealTimers());

  const COMMENT_COOLDOWN_MS = 21_000;
  const ROOT_POST_COOLDOWN_MS = 5 * 60 * 1000 + 10_000;

  it("returns null when no accounts exist", () => {
    expect(selectAccountForComment([])).toBeNull();
  });

  it("returns a fresh account (lastCommentAt is null)", () => {
    const acct = makeAccount({ lastCommentAt: null });
    expect(selectAccountForComment([acct])).toBe(acct);
  });

  it("returns null when account commented within cooldown", () => {
    const recentTime = new Date(Date.now() - COMMENT_COOLDOWN_MS + 5000);
    const acct = makeAccount({ lastCommentAt: recentTime });
    expect(selectAccountForComment([acct])).toBeNull();
  });

  it("returns account when cooldown has expired", () => {
    const oldTime = new Date(Date.now() - COMMENT_COOLDOWN_MS - 1000);
    const acct = makeAccount({ lastCommentAt: oldTime });
    expect(selectAccountForComment([acct])).toBe(acct);
  });

  it("skips excluded account IDs", () => {
    const acct1 = makeAccount({ id: "a1", lastCommentAt: null });
    const acct2 = makeAccount({ id: "a2", lastCommentAt: null });
    expect(selectAccountForComment([acct1, acct2], ["a1"])).toBe(acct2);
  });

  it("returns null when all accounts are excluded", () => {
    const acct = makeAccount({ id: "a1", lastCommentAt: null });
    expect(selectAccountForComment([acct], ["a1"])).toBeNull();
  });

  it("COMMENT_COOLDOWN_MS is 21 seconds", () => {
    expect(COMMENT_COOLDOWN_MS).toBe(21_000);
  });

  it("ROOT_POST_COOLDOWN_MS is 5 minutes + 10 seconds", () => {
    expect(ROOT_POST_COOLDOWN_MS).toBe(5 * 60 * 1000 + 10_000);
  });
});
```

### Rate Limit State Transitions

```typescript
describe("Rate limit cooldown state", () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2025-06-15T12:00:00.000Z"));
  });
  afterEach(() => vi.useRealTimers());

  it("returns false initially (no rate limit recorded)", () => {
    expect(isCoolingDown()).toBe(false);
  });

  it("returns true immediately after recording a rate limit", () => {
    recordRateLimit();
    expect(isCoolingDown()).toBe(true);
  });

  it("returns false after the cooldown expires", () => {
    recordRateLimit(); // 5min cooldown
    vi.advanceTimersByTime(5 * 60 * 1000 + 1);
    expect(isCoolingDown()).toBe(false);
  });

  it("uses flat cooldown (no escalation on consecutive hits)", () => {
    recordRateLimit();
    recordRateLimit(); // Second hit should NOT extend
    vi.advanceTimersByTime(5 * 60 * 1000 + 1);
    expect(isCoolingDown()).toBe(false);
  });

  it("returns 0 remaining when not cooling down", () => {
    expect(cooldownRemainingMs()).toBe(0);
  });

  it("returns remaining ms when cooling down", () => {
    recordRateLimit(); // 5min cooldown
    vi.advanceTimersByTime(60 * 1000); // 1min elapsed
    expect(cooldownRemainingMs()).toBe(4 * 60 * 1000);
  });

  it("reset clears all cooldown state", () => {
    recordRateLimit();
    expect(isCoolingDown()).toBe(true);
    resetCooldown();
    expect(isCoolingDown()).toBe(false);
  });
});
```

---

## Summary

| Pattern | When to use |
|---------|-------------|
| **Multi-Account Rotation** | dApps posting > 1 comment/21s or > 1 root post/5min |
| **Rate Limit Recovery** | Any app broadcasting to Hive |
| **Content Integrity Hashing** | Archival, proof-of-existence, legal compliance |
| **Deduplication** | Multi-source ingestion, idempotent pipelines |
| **Post Editing** | Attribution updates, content corrections within 7 days |
| **Container/Thread** | High-volume data storage (100s+ items per topic) |
| **Challenge-Response Auth** | Server-side Hive Keychain/HiveSigner authentication |
| **Testing Patterns** | CI/CD for any Hive dApp — no live chain needed |
