# Hive Master Skill

A comprehensive [Claude Code](https://claude.ai/claude-code) skill that gives Claude deep expertise in Hive blockchain development. Built from patterns mined across 40+ production Hive apps — not just docs, but battle-tested knowledge including workarounds, gotchas, and fixes you won't find anywhere else.

## What's Inside

| File | Coverage |
|------|----------|
| `SKILL.md` | Skill entry point and trigger configuration |
| `api-reference.md` | RPC nodes, API namespaces (condenser, bridge, database, rc_api), image upload signing |
| `operations.md` | Every Hive operation — comment, vote, transfer, power up/down, delegate, savings, convert, witness |
| `content.md` | Posts, replies, votes, feeds, voting power math, optimistic updates, avatar parsing, mute lists |
| `wallet.md` | Transfers, VESTS/HP conversion, savings, HBD conversion, recurrent transfers, transaction history |
| `custom-json.md` | Follow/unfollow/mute, reblog, community ops, RC delegation, Hive Engine ops |
| `keys-and-auth.md` | Key hierarchy, challenge-response signing, Keychain, HiveAuth, HiveSigner, secure storage |
| `communities.md` | Subscribe, post to community, roles, moderation, update properties |
| `streaming-and-realtime.md` | Block streaming, polling patterns, notifications, WebSocket pitfalls, push notification architecture |
| `hive-engine.md` | Layer 2 tokens, balances, order book, staking, NFTs, diesel pools |
| `battle-tested.md` | 18 hard-won lessons — dhive failover fix, broadcast retry dangers, beneficiary sorting, iOS quirks, and more |
| `react-native.md` | React Native / Expo patterns — SecureStore, image handling, networking, optimistic updates |
| `examples/dhive-patterns.md` | Complete code: publish posts, vote with value estimation, paginated feeds, account info |
| `examples/react-native-hive.md` | Complete hooks: login flow, useHivePost, useVote, useResourceCredits, ReplyBox component |

## Setup

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and working

### Option 1: Personal skill (available across all your projects)

```bash
# Clone into your personal Claude skills directory
git clone https://github.com/Mantequilla-Soft/hive-master-skill.git ~/.claude/skills/hive-master
```

That's it. The skill is now available in every project on your machine. Claude will automatically use it when you're doing Hive-related work, or you can invoke it manually with `/hive-master`.

### Option 2: Per-project skill (committed to your repo)

```bash
# From your project root
mkdir -p .claude/skills
git clone https://github.com/Mantequilla-Soft/hive-master-skill.git .claude/skills/hive-master

# Remove the nested .git so it's part of your repo (not a submodule)
rm -rf .claude/skills/hive-master/.git

# Commit it
git add .claude/skills/hive-master
git commit -m "Add hive-master skill"
```

### Option 3: Git submodule (per-project, stays updated)

```bash
# From your project root
mkdir -p .claude/skills
git submodule add https://github.com/Mantequilla-Soft/hive-master-skill.git .claude/skills/hive-master
git commit -m "Add hive-master skill as submodule"
```

To pull updates later:

```bash
git submodule update --remote .claude/skills/hive-master
```

## Usage

Once installed, the skill works in two ways:

**Automatic** — Claude detects you're working on Hive-related code and loads the relevant knowledge. Just start coding and it kicks in.

**Manual** — Type `/hive-master` in Claude Code to explicitly load the skill. You can optionally pass a topic:

```
/hive-master wallet
/hive-master custom json
/hive-master react native
```

## Updating

If you installed as a personal skill (Option 1):

```bash
cd ~/.claude/skills/hive-master && git pull
```

## Contributing

Found a new gotcha? Discovered a better pattern? PRs are welcome — especially for `battle-tested.md`. The goal is to capture the knowledge that saves hours of debugging.

## License

MIT
