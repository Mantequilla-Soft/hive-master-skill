# HafSQL — SQL Access to Hive Chain Data

HafSQL is a PostgreSQL layer over HAF (Hive Application Framework) indexed blockchain data. It covers every operation, including virtual ops, plus hivemind social data, all in one database you can query with plain SQL. Use it for account history, reward analytics, dashboards and anything that would take thousands of RPC calls otherwise.

The full reference lives in the standalone [hafsql-hive](https://github.com/rantree-hive/hafsql-hive) skill. This file covers the parts that bite most often.

---

## 1. Connecting to the Public Instance

A read-only public instance is maintained by @mahdiyari:

```
host=hafsql-sql.mahdiyari.info  port=5432  dbname=haf_block_log
user=hafsql_public  password=hafsql_public  sslmode=disable
```

```typescript
import { Pool } from "pg";

const pool = new Pool({
  host: "hafsql-sql.mahdiyari.info",
  port: 5432,
  database: "haf_block_log",
  user: "hafsql_public",
  password: "hafsql_public",
  ssl: false,
  max: 5,                        // shared node, so keep the pool small
  idleTimeoutMillis: 5000,
  connectionTimeoutMillis: 5000, // fail fast
});

async function query<T = any>(sql: string, params: unknown[] = [], timeoutMs = 15000): Promise<T[]> {
  const client = await pool.connect();
  try {
    await client.query(`SET statement_timeout = ${timeoutMs}`); // per query, not global
    return (await client.query(sql, params)).rows;
  } finally {
    client.release();
  }
}
```

**It is a shared node.** Always filter on an indexed column plus a time bound (Section 4), cache results on your side, and don't retry on `max_client_conn` errors, since that means the server is at capacity. For production traffic, add a retry for transient errors (`ECONNRESET`, `terminated`) and a circuit breaker. The hafsql-hive skill has a complete pool setup.

## 2. Schema Map

Prefer the **dedicated operation tables** over the generic views. They are faster and already have typed columns.

### Dedicated operation tables (fast, typed)

| Table | Filter column | Key columns |
|-------|---------------|-------------|
| `hafsql.operation_transfer_table` | `from_account`, `to_account` | `amount`, `symbol`, `memo` |
| `hafsql.operation_vote_view` | `voter`, `author` | `weight` (negative = downvote), `timestamp` |
| `hafsql.operation_comment_view` | `author` | `permlink`, `parent_author`, `body` (**one row per edit**, see Section 6) |
| `hafsql.operation_custom_json_view` | `custom_id` | `json` (**text**, not `payload`), `required_auths`, `required_posting_auths` |
| `hafsql.operation_author_reward_table` | `author` | `hive_payout`, `hbd_payout`, `vesting_payout` |
| `hafsql.operation_curation_reward_table` | `curator` | `reward` (VESTS) |
| `hafsql.operation_comment_benefactor_reward_table` | `benefactor` | `hive_payout`, `hbd_payout`, `vesting_payout` |
| `hafsql.operation_comment_reward_table` | `author` | `payout`, `author_rewards` (fires at payout, see Section 5) |
| `hafsql.operation_delegate_vesting_shares_table` | `delegator`, `delegatee` | `vesting_shares` |
| `hafsql.operation_fill_vesting_withdraw_table` | `from_account`, `to_account` | `withdrawn`, `deposited` |
| `hafsql.operation_proposal_pay_table` | `receiver`, `proposal_id` | `payment`, `symbol`, `payer` (DHF payouts) |
| `hafsql.operation_effective_comment_vote_view` | `voter`, `author` | `rshares`, `vote_value`, `pending_payout` |

### State views

| View | Purpose |
|------|---------|
| `hafsql.accounts` | HP, delegation totals, reputation, creator, recovery account |
| `hafsql.balances` | Current HIVE / HBD / VESTS / `hp_equivalent` per `account_name` |
| `hafsql.dynamic_global_properties` | Chain globals, **one row per block** (see Section 7) |
| `hafsql.reputations` | `rep` is **already display-scale** (25–80); `reputation` is raw |
| `hafsql.delegations`, `hafsql.rc_delegations` | Active HP and RC delegations |
| `hafsql.proposal_approvals` | Current DHF proposal votes |
| `hafsql.comments` | Latest state of every post and reply (`parent_author = ''` means root posts; also `last_edited`, `deleted`, `beneficiaries`) |
| `hivemind_app.hive_accounts_info_view` | `followers`, `following`, `reputation`, `posting_json_metadata`, `created_at` |
| `hivemind_app.hive_votes_view` | Per-vote `rshares`, `percent`, `is_effective` |

### Generic views (slow, use as a last resort)

| View | Notes |
|------|-------|
| `hafsql.haf_operations` | Every op, real **and** virtual; `body` is `{type, value}` JSONB |
| `hafsql.haf_account_operations` | Account → op index with `account_op_seq_no` |
| `hafsql.haf_operation_types` | Op type registry (`id`, `name`, `is_virtual`) |

### Utility functions

| Function | Use |
|----------|-----|
| `hafsql.id_from_timestamp(ts, round_up)` | Time → op id, for indexed time filtering |
| `hafsql.get_timestamp(id)` | Op id → timestamp (many tables have no timestamp column) |
| `hafsql.to_json(text)` | Parse the custom_json `json` text column to JSONB |

## 3. Generic View vs Dedicated Tables

`hafsql.haf_operations` holds every operation, virtual ones included (rewards, producer_reward, fill_vesting_withdraw, proposal_pay…). But filtering it by JSONB body fields is slow. The dedicated tables are indexed by account and have typed columns, so use them whenever one exists:

```sql
-- ❌ Works, but scans JSONB
SELECT o.body->'value'->>'hbd_payout' FROM hafsql.haf_operations o
WHERE o.op_type_id = 51 AND o.body->'value'->>'author' = $1;

-- ✅ Indexed and typed
SELECT SUM(hbd_payout) FROM hafsql.operation_author_reward_table
WHERE author = $1
  AND id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - INTERVAL '30 days', true);
```

Naming gotchas:

- Op type names carry the full prefix: `hive::protocol::vote_operation`, not `vote_operation`. Filter with the full name and strip it only for display.
- There is **no `hafsql.operation_types`** and **no `hafsql.operations`**. Use `hafsql.haf_operation_types` / `hafsql.haf_operations`. A try/catch around the wrong name fails silently.
- DHF payouts are virtual `proposal_pay` ops, never transfers. They aren't in `operation_transfer_table`, so use `operation_proposal_pay_table` with `payer = 'hive.fund'` and `proposal_id != 0` (proposal 0 is the Return Proposal).

## 4. Time Filtering with `id_from_timestamp`

Operation ids increase over time, and every `operation_*` table is indexed on `(account_column, id)`. Turn the time bound into an id bound so the index is used:

```sql
-- ✅ Indexed
WHERE voter = $1
  AND id > hafsql.id_from_timestamp((NOW() AT TIME ZONE 'utc') - INTERVAL '7 days', true)

-- ❌ Joining to a timestamp, or fetching everything and filtering in JS,
--    is slow and can hit LIMIT before covering the range
```

Several windows in one pass:

```sql
WITH t AS (
  SELECT hafsql.id_from_timestamp((NOW() AT TIME ZONE 'utc') - INTERVAL '30 days', true) AS id_30d,
         hafsql.id_from_timestamp((NOW() AT TIME ZONE 'utc') - INTERVAL '7 days',  true) AS id_7d
)
SELECT SUM(hbd_payout)                                         AS hbd_all,
       SUM(CASE WHEN r.id > t.id_30d THEN hbd_payout ELSE 0 END) AS hbd_30d,
       SUM(CASE WHEN r.id > t.id_7d  THEN hbd_payout ELSE 0 END) AS hbd_7d
FROM hafsql.operation_author_reward_table r, t
WHERE r.author = $1;
```

**Many accounts:** `WHERE voter = ANY($1::text[])` with a long array falls back to a slow bitmap scan. Run one query per account in parallel instead (`Promise.all`), so each one uses the `(voter, id)` index.

**`hafsql.comments` is the exception.** Its `id` is **not** an op id, so `id_from_timestamp` doesn't apply. Filter on `created`, and always bound it for multi-account lookups (`created > NOW() - INTERVAL '3 years'`) to avoid full scans on inactive accounts.

## 5. Reward Timing

`operation_comment_reward_table` (and the author and curation reward tables) fire at **payout time, 7 days after the post**, not when it was created. A 7-day window returns rewards for posts made 7–14 days ago. To relate rewards to recent posting activity, use a window of at least 30 days.

## 6. Comment Edits Are Separate Operations

Every edit broadcasts a new `comment_operation` with the same `author` + `permlink`. `operation_comment_view` has one row per version, so counting rows overcounts and reading `body` may return an old version. Deduplicate to the latest version:

```sql
SELECT DISTINCT ON (c.permlink)
  c.author, c.permlink, c.parent_author, c.json_metadata,
  hafsql.get_timestamp(c.id) AS last_edit
FROM hafsql.operation_comment_view c
WHERE c.author = $1
  AND c.id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - INTERVAL '90 days', true)
ORDER BY c.permlink, c.id DESC;   -- newest version wins
```

`hafsql.comments.body` can lag behind very recent edits. When edit content matters (for example a mention added after posting), re-fetch recent posts with `condenser_api.get_content`.

## 7. Unit Traps: Global Properties, VESTS, Payouts

**`dynamic_global_properties` has one row per block** (it's a view over `hafd.blocks`, ~100M rows):

```sql
-- ✅ Current values (fast, block_num is the primary key)
SELECT vests_per_hive FROM hafsql.dynamic_global_properties ORDER BY block_num DESC LIMIT 1;

-- ❌ LIMIT 1 without ORDER BY returns an arbitrary, possibly years-old block
-- ❌ Joining the view without that subquery multiplies your result by every block in history
```

**`vests_per_hive` is VESTS per HIVE** (~1600), so divide:

```typescript
const [{ vests_per_hive }] = await query(
  `SELECT vests_per_hive FROM hafsql.dynamic_global_properties ORDER BY block_num DESC LIMIT 1`
);
const [bal] = await query(
  `SELECT vests, hp_equivalent FROM hafsql.balances WHERE account_name = $1`, [account]
);
const hp = parseFloat(bal.vests) / parseFloat(vests_per_hive); // ✅ equals bal.hp_equivalent
```

**VESTS units:** every VESTS column in the `hafsql` views and operation tables (`balances.vests`, `accounts.incoming_vests`/`outgoing_vests`, reward `vesting_payout`, `withdraw_vesting.vesting_shares`) is in **whole VESTS**. The exceptions are the raw `total_vesting_fund_hive` / `total_vesting_shares` columns of `dynamic_global_properties`. They are integer chain units (×1e3 and ×1e6), so dividing them directly gives a HIVE-per-VEST ratio **1000× too small**. Use `1 / vests_per_hive` instead.

`hp_equivalent`, `incoming_hp` and `outgoing_hp` are already in HP.

**Payout columns:** `hafsql.comments` payout columns are `numeric` on the current public node. Older HafSQL versions returned `"1.234 HBD"` strings, so code that must run against several nodes can use `SPLIT_PART(col::text, ' ', 1)::numeric`, which handles both forms.

**Reputation:** `hafsql.reputations.rep` is already display-scale. Applying a raw-to-display conversion to it produces nonsense.

## 8. Reading custom_json

The payload column is `json` (text), so parse it with `hafsql.to_json`:

```sql
SELECT hafsql.get_timestamp(id)          AS ts,
       required_posting_auths->>0         AS account,
       (hafsql.to_json(json))->>'action'  AS action
FROM hafsql.operation_custom_json_view
WHERE custom_id = 'your-app-id'
  AND id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - INTERVAL '30 days', true);
-- ❌ to_json(payload): there is no `payload` column
```

Built-in ids like `follow` and `reblog` carry a JSON **array** (`["follow", {...}]`), so read their fields with `->1->>'following'`.

For one account's full custom_json history, go through the account index, **filtered by op type** (custom_json is `op_type_id = 18`) so it uses the `(account_id, op_type_id)` index. Without that filter, active accounts time out:

```sql
SELECT v.id, v.timestamp, hafsql.to_json(v.json) AS data
FROM hafd.accounts a
JOIN hafd.account_operations ao ON ao.account_id = a.id AND ao.op_type_id = 18
JOIN hafsql.operation_custom_json_view v ON v.id = ao.operation_id
WHERE a.name = $1 AND v.custom_id = 'your-app-id'
ORDER BY ao.account_op_seq_no DESC
LIMIT 50;
```

## 9. Account History with Cursor Pagination

Page by `account_op_seq_no`. Pick the page in a `MATERIALIZED` CTE so PostgreSQL walks the `(account_id, account_op_seq_no)` index backwards and joins only the rows it keeps:

```sql
WITH page AS MATERIALIZED (
  SELECT ao.account_op_seq_no, ao.operation_id
  FROM hafd.account_operations ao
  WHERE ao.account_id = (SELECT id FROM hafd.accounts WHERE name = $1)
    AND ao.op_type_id <> 72          -- effective_comment_vote: noisy, fires on every vote
    AND ao.account_op_seq_no < $2    -- cursor; omit on page 1
  ORDER BY ao.account_op_seq_no DESC
  LIMIT 51                           -- hasMore = rows.length === 51; serve the first 50
)
SELECT p.account_op_seq_no AS seq_no, o.id, o.timestamp, ot.name AS op_type, o.body
FROM page p
JOIN hafsql.haf_operations o       ON o.id  = p.operation_id
JOIN hafsql.haf_operation_types ot ON ot.id = o.op_type_id
ORDER BY p.account_op_seq_no DESC;
```

**Don't** bound this with `operation_id >= id_from_timestamp(...)`. There's no index on `(account_id, operation_id)`, so it filters the account's whole history and times out for active accounts. For a date range, keep paging until the last row's timestamp is past the start.

**Body shape:** `haf_operations.body` is always `{ type, value }`, so the op fields live under `value`:

```typescript
const op = row.body.value;   // op.voter, op.author, ...
```

## 10. Hivemind Data in the Same Database

Hivemind tables sit next to HafSQL, so social data is just a join away:

```sql
-- Profile metadata (prefer posting_json_metadata over json_metadata)
SELECT posting_json_metadata, followers, following, reputation
FROM hivemind_app.hive_accounts_info_view WHERE name = $1;

-- Community/category of a post
SELECT hcd.category
FROM hivemind_app.hive_accounts ha
JOIN hivemind_app.hive_permlink_data hpd ON hpd.permlink = $2
JOIN hivemind_app.hive_posts hp ON hp.author_id = ha.id AND hp.permlink_id = hpd.id
LEFT JOIN hivemind_app.hive_category_data hcd ON hcd.id = hp.category_id
WHERE ha.name = $1;
```

`hive_accounts_info_view.post_count` counts **all** authored content, replies included. For root posts only, use `SELECT COUNT(*) FROM hafsql.comments WHERE author = $1 AND parent_author = ''`.

This is also the reliable source for reputation. `condenser_api.get_accounts` on HAFAH-backed nodes returns `reputation: 0` and `comment_count: 0`.

## 11. Index Cheat-Sheet

| Table | Indexes | Write queries that… |
|-------|---------|---------------------|
| `operation_transfer_table` | `(from_account, id)`, `(to_account, id)`, `(memo, id)` | filter on an account **or** an exact memo, plus an `id >` bound |
| `operation_vote_view` | `(voter, id)`, `(author, id)` | use one voter per query |
| most `operation_*_table` | `(account col)`, unique `(id)` | put the account first, then `id_from_timestamp` |
| `hafd.account_operations` | `(account_id, account_op_seq_no)`, `(account_id, op_type_id)` | paginate by seq_no |
| `hafd.operations` | partial on `body_value->>'author'` for vote / comment / effective_comment_vote | look up by author and permlink only |

Filtering on an unindexed column (`memo LIKE '%x%'`, JSONB fields, casts) over a wide id range **will time out** on the public node.

## 12. Pitfalls Checklist

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Filtering `haf_operations` by JSONB fields | Slow / timeouts | Typed, indexed `operation_*` tables (Section 3) |
| `dynamic_global_properties LIMIT 1` without `ORDER BY block_num DESC` | Stale ratio from an old block | `ORDER BY block_num DESC LIMIT 1` (Section 7) |
| `vests * vests_per_hive` | HP millions × too big | `vests / vests_per_hive` |
| Ratio from raw `total_vesting_*` globals | HP 1000× too small | `1 / vests_per_hive` |
| Op type name without `hive::protocol::` | Matches nothing | Use the full prefixed name |
| `hafsql.operation_types` / `hafsql.operations` | `relation does not exist` | `hafsql.haf_operation_types` / `hafsql.haf_operations` |
| `to_json(payload)` on custom_json | `column "payload" does not exist` | `hafsql.to_json(json)` |
| Filtering time in JS after a big fetch | Slow; LIMIT cuts off the range | `id_from_timestamp` in WHERE (Section 4) |
| `voter = ANY(long array)` | Bitmap scan timeout | One query per voter, in parallel |
| `id_from_timestamp` on `hafsql.comments` | Wrong rows | Filter on `created` |
| 7-day window on reward tables | Rewards for posts from the previous week | 30-day window (Section 5) |
| Counting `operation_comment_view` rows | Edits counted twice; old bodies | `DISTINCT ON (permlink) … id DESC` (Section 6) |
| `body.voter` is undefined | Body is `{ type, value }` | `row.body.value.voter` |
| History bounded by `operation_id >= id_from_timestamp()` | Timeout | Page by `account_op_seq_no` (Section 9) |
| Account-index custom_json without `op_type_id = 18` | Timeout | Add the op type filter (Section 8) |
| DHF payouts searched in transfers | 0 rows | `operation_proposal_pay_table` |
| Reputation conversion on `reputations.rep` | Nonsense values | `rep` is already display-scale |
| Huge multi-account scan in one query | Timeouts / Node heap OOM | Chunk ~1000 accounts at a time |
