---
name: hive-master
description: Expert knowledge on Hive blockchain development — API calls, node failover, streaming, posts, comments, votes, wallet transactions, communities, custom JSON, keys/auth, Hivemind, Hive Engine, and React Native patterns. Use when building anything that interacts with the Hive blockchain, dhive, or Hive APIs.
user-invocable: true
argument-hint: [topic]
---

# Hive Blockchain Expert

You are a Hive blockchain development expert. Apply this knowledge when writing, reviewing, or debugging code that interacts with the Hive blockchain.

## Quick Reference

| Topic | Reference File |
|-------|---------------|
| API nodes, endpoints, RPC methods | [api-reference.md](api-reference.md) |
| All operation builders (vote, comment, transfer, etc.) | [operations.md](operations.md) |
| Posts, comments, votes, reblogs, feeds | [content.md](content.md) |
| Wallet: transfers, delegations, power up/down, savings | [wallet.md](wallet.md) |
| Community operations | [communities.md](communities.md) |
| Custom JSON operations | [custom-json.md](custom-json.md) |
| Keys, auth, Keychain, HiveAuth, HiveSigner | [keys-and-auth.md](keys-and-auth.md) |
| Streaming, polling, real-time data | [streaming-and-realtime.md](streaming-and-realtime.md) |
| Hive Engine (layer 2 tokens) | [hive-engine.md](hive-engine.md) |
| Image uploads: signing, HEIC, fallbacks | [image-uploads.md](image-uploads.md) |
| Battle-tested workarounds and gotchas | [battle-tested.md](battle-tested.md) |
| React Native specific patterns | [react-native.md](react-native.md) |
| Code examples and patterns | [examples/](examples/) |

## Core Principles

1. **Always use multiple RPC nodes** with failover — single node = single point of failure
2. **Never retry broadcast operations** — duplicates hit the chain and cause errors
3. **Beneficiaries must be sorted alphabetically** by account name — protocol requirement
4. **Vote weights are basis points** — 10000 = 100%, -10000 = -100% downvote
5. **Amounts always have 3 decimal places** — `1.000 HIVE`, `0.500 HBD`
6. **Posting key for social ops**, active key for financial ops, memo key for encrypted memos
7. **3-second block times** — wait at least 3s after broadcast before querying
8. **Resource Credits gate all operations** — check RC before broadcasting

## When to Use This Skill

- Building Hive dApps (web or mobile)
- Integrating dhive or Hive APIs
- Publishing content (posts, comments, votes)
- Wallet operations (transfers, delegations, staking)
- Community management
- Custom JSON operations (follow, reblog, app-specific)
- Authentication with Hive keys
- Video/image publishing on Hive (3Speak, etc.)
- Price feeds, witness operations
- Transaction history and account data
