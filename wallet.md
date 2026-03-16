# Wallet Operations

## Account Balances

```typescript
const [account] = await client.database.getAccounts(['username']);

const hiveBalance = account.balance;                    // "123.456 HIVE"
const hbdBalance = account.hbd_balance;                 // "45.678 HBD"
const savingsHive = account.savings_balance;             // "0.000 HIVE"
const savingsHbd = account.savings_hbd_balance;          // "100.000 HBD"
const vestingShares = account.vesting_shares;            // "12345.678901 VESTS"
const receivedVesting = account.received_vesting_shares; // Delegated to you
const delegatedVesting = account.delegated_vesting_shares; // You delegated out
```

### VESTS to HP Conversion

```typescript
const globalProps = await client.database.getDynamicGlobalProperties();
const totalVestingShares = parseFloat(globalProps.total_vesting_shares);
const totalVestingFund = parseFloat(globalProps.total_vesting_fund_hive);

function vestsToHP(vests: number): number {
  return (vests / totalVestingShares) * totalVestingFund;
}

function hpToVests(hp: number): number {
  return (hp / totalVestingFund) * totalVestingShares;
}

// Effective HP = own + received - delegated
const effectiveVests = parseFloat(vestingShares)
  + parseFloat(receivedVesting)
  - parseFloat(delegatedVesting);
const effectiveHP = vestsToHP(effectiveVests);
```

## Transfer HIVE/HBD (Active Key)

```typescript
await client.broadcast.transfer({
  from: 'sender',
  to: 'receiver',
  amount: '1.000 HIVE', // Always 3 decimal places
  memo: 'Payment for services',
}, activeKey);

// Encrypted memo (requires memo key of recipient to decrypt)
// Prefix memo with '#' to encrypt
await client.broadcast.transfer({
  from: 'sender',
  to: 'receiver',
  amount: '1.000 HBD',
  memo: '#This memo is encrypted',
}, activeKey);
```

### Amount Formatting

```typescript
// ALWAYS format to 3 decimal places with currency symbol
function formatAmount(value: number, currency: 'HIVE' | 'HBD'): string {
  return `${value.toFixed(3)} ${currency}`;
}
// "1.000 HIVE", "0.500 HBD"
// VESTS use 6 decimal places: "1234.567890 VESTS"
```

## Power Up / Power Down

```typescript
// Power Up: HIVE → HP (instant)
await client.broadcast.sendOperations([
  ['transfer_to_vesting', {
    from: 'username',
    to: 'username', // Can power up another account
    amount: '100.000 HIVE',
  }]
], activeKey);

// Power Down: HP → HIVE (13 weekly installments)
await client.broadcast.sendOperations([
  ['withdraw_vesting', {
    account: 'username',
    vesting_shares: '50000.000000 VESTS',
  }]
], activeKey);

// Cancel Power Down
await client.broadcast.sendOperations([
  ['withdraw_vesting', {
    account: 'username',
    vesting_shares: '0.000000 VESTS',
  }]
], activeKey);
```

## Delegations

```typescript
// Delegate HP (in VESTS)
await client.broadcast.sendOperations([
  ['delegate_vesting_shares', {
    delegator: 'from-account',
    delegatee: 'to-account',
    vesting_shares: '10000.000000 VESTS',
  }]
], activeKey);

// Remove delegation (5-day cooldown before VESTS return)
await client.broadcast.sendOperations([
  ['delegate_vesting_shares', {
    delegator: 'from-account',
    delegatee: 'to-account',
    vesting_shares: '0.000000 VESTS',
  }]
], activeKey);
```

## Savings (3-Day Withdrawal Delay)

```typescript
// Deposit to savings
await client.broadcast.sendOperations([
  ['transfer_to_savings', {
    from: 'username',
    to: 'username',
    amount: '100.000 HBD',
    memo: '',
  }]
], activeKey);

// Initiate withdrawal (takes 3 days)
const requestId = Math.floor(Date.now() / 1000); // Unique request ID
await client.broadcast.sendOperations([
  ['transfer_from_savings', {
    from: 'username',
    to: 'username',
    amount: '50.000 HBD',
    memo: '',
    request_id: requestId,
  }]
], activeKey);

// Cancel pending withdrawal
await client.broadcast.sendOperations([
  ['cancel_transfer_from_savings', {
    from: 'username',
    request_id: requestId,
  }]
], activeKey);
```

## HBD Conversion

```typescript
// Convert HBD → HIVE (3.5-day conversion period)
await client.broadcast.sendOperations([
  ['convert', {
    owner: 'username',
    amount: '10.000 HBD',
    requestid: Math.floor(Date.now() / 1000),
  }]
], activeKey);
```

## Recurrent Transfers

```typescript
// Create recurring payment
await client.broadcast.sendOperations([
  ['recurrent_transfer', {
    from: 'sender',
    to: 'receiver',
    amount: '10.000 HIVE',
    memo: 'Monthly subscription',
    recurrence: 720, // Hours (720 = 30 days). Minimum: 24 hours
    executions: 12,  // Run 12 times then stop
  }]
], activeKey);

// Cancel recurrent transfer
await client.broadcast.sendOperations([
  ['recurrent_transfer', {
    from: 'sender',
    to: 'receiver',
    amount: '0.000 HIVE',
    memo: '',
    recurrence: 24,
    executions: 0, // 0 = cancel
  }]
], activeKey);
```

## Claim Rewards

```typescript
const [account] = await client.database.getAccounts(['username']);

const rewardHive = account.reward_hive_balance;   // "0.123 HIVE"
const rewardHbd = account.reward_hbd_balance;      // "0.456 HBD"
const rewardVests = account.reward_vesting_balance; // "1.234567 VESTS"

// Only claim if there's something to claim
if (parseFloat(rewardHive) > 0 || parseFloat(rewardHbd) > 0 || parseFloat(rewardVests) > 0) {
  await client.broadcast.sendOperations([
    ['claim_reward_balance', {
      account: 'username',
      reward_hive: rewardHive,
      reward_hbd: rewardHbd,
      reward_vests: rewardVests,
    }]
  ], postingKey); // NOTE: Posting key is sufficient for claiming rewards
}
```

## Transaction History

```typescript
// Fetch in batches of 1000 (API maximum)
async function getFullHistory(username: string) {
  const allOps = [];
  let start = -1; // -1 = most recent

  while (true) {
    const batch = await client.call('condenser_api', 'get_account_history', [
      username, start, 1000,
    ]);

    if (batch.length === 0) break;
    allOps.push(...batch);

    // Next batch starts from the earliest operation in current batch
    start = batch[0][0] - 1;
    if (start < 0) break;
  }

  return allOps;
}

// Filter for specific operation types
const transfers = history.filter(([_, op]) =>
  op.op[0] === 'transfer' && op.op[1].to === 'username'
);
```

## Resource Credits Check

Always check RC before attempting operations:

```typescript
async function hasEnoughRC(username: string, threshold = 5): Promise<boolean> {
  const response = await client.call('rc_api', 'find_rc_accounts', {
    accounts: [username],
  });
  const rc = response.rc_accounts[0];
  const percentage = (parseFloat(rc.rc_manabar.current_mana) / parseFloat(rc.max_rc)) * 100;
  return percentage > threshold;
}
```

## Witness Price Feed (Active Key)

```typescript
// Publish price feed as a witness
const exchangeRate = { base: '0.350 HBD', quote: '1.000 HIVE' };

await client.broadcast.sendOperations([
  ['feed_publish', {
    publisher: 'witness-account',
    exchange_rate: exchangeRate,
  }]
], activeKey);
```
