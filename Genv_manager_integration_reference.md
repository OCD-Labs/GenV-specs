# GenV — AI Manager Integration Reference

**Audience:** AI decision engine and backend integration engineer.

**Purpose:** Everything the manager system needs to interact with the Midnight contract and
coordinate with Polygon. This document covers all eight manager-gated circuits, the four
operational loops that trigger them, ordering rules that must not be violated, and what the
manager system explicitly does not touch.

---

## 1. Manager Wallet Setup

The manager runs a programmatic Midnight wallet backed by the manager's seed. This is not a
browser extension — it is a server-side wallet instance. The `own_public_key()` derived from
this seed must match `manager_key` stored on the ledger, otherwise every manager-gated circuit
call will fail proof generation.

```typescript
import {
  Contract,
  type Ledger,
  type WithdrawTicket,
  witnesses,
} from './contract/index.cjs';

import {
  WalletBuilder,
  NodeZkConfigProvider,
} from '@midnight-ntwrk/midnight-js-wallet';

import { indexerPublicDataProvider } from '@midnight-ntwrk/midnight-js-indexer-public-data-provider';

const zkConfig       = new NodeZkConfigProvider(PROOF_SERVER_URL, contractConfig);
const managerWallet  = await WalletBuilder.buildFromSeed(INDEXER_URL, zkConfig, MANAGER_SEED, 'sync');
const provider       = indexerPublicDataProvider(INDEXER_URL, contractConfig);
const contract       = new Contract(witnesses);
```

Keep `MANAGER_SEED` in a secrets manager. Rotating the manager key on-chain is a two-step
process covered in Section 6.

---

## 2. Polygon Wallet Setup

The Polygon execution wallet holds the vault's USDC. The manager uses this wallet to receive
deposits from users, send payouts to withdrawing users, and move capital to and from
Polymarket.

```typescript
import { ethers } from 'ethers';

const polygonProvider = new ethers.JsonRpcProvider(POLYGON_RPC_URL);
const executionWallet = new ethers.Wallet(EXECUTION_WALLET_PRIVATE_KEY, polygonProvider);

// USDC contract on Polygon mainnet
const USDC_ADDRESS = '0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174';
const usdc = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, executionWallet);
```

---

## 3. Reading Vault State

The manager reads the same public ledger as the frontend. Read it before every operational
decision to get current totals, float usage, and ticket queue depth.

```typescript
async function readLedger(): Promise<Ledger> {
  return provider.queryContractState(CONTRACT_ADDRESS);
}

function totalAssets(ledger: Ledger): bigint {
  return ledger.total_liquid + ledger.float_outstanding;
}

function effectiveSupply(ledger: Ledger): bigint {
  return ledger.total_share_supply + ledger.shares_pending_withdrawal;
}

function maxFloat(ledger: Ledger): bigint {
  return (totalAssets(ledger) * BigInt(ledger.max_float_bps)) / 10_000n;
}

function floatRemaining(ledger: Ledger): bigint {
  const max = maxFloat(ledger);
  return max > ledger.float_outstanding ? max - ledger.float_outstanding : 0n;
}
```

---

## 4. Ticket Enumeration

Enumerate pending withdrawal tickets on every poll cycle. The pattern is identical to the
frontend — derive each key from its sequential index.

```typescript
import { persistentHash } from '@midnight-ntwrk/compact-runtime';
// NOTE: verify this import path against the compiled contract output. The SDK may
// re-export persistentHash directly from the contract module depending on compiler version.

function ticketKey(index: bigint): Uint8Array {
  return persistentHash(index);
}

async function fetchPendingTickets(totalTickets: bigint): Promise<WithdrawTicket[]> {
  const pending: WithdrawTicket[] = [];

  for (let i = 0n; i < totalTickets; i++) {
    const key    = ticketKey(i);
    const ticket = await provider.queryMapEntry('tickets', key) as WithdrawTicket;
    if (ticket && !ticket.is_filled && !ticket.is_cancelled) {
      pending.push(ticket);
    }
  }

  return pending;
}
```

---

## 5. Operational Loops

There are four loops. Each one has a trigger, an off-chain action, and a Midnight circuit call
that records the result. The ordering within each loop is strict — see Section 7.

### 5.1 Deposit Loop

**Trigger:** the manager's Polygon monitor detects a confirmed USDC transfer to the execution
wallet from a known depositor address.

**Off-chain first:** confirm the transfer has reached finality (recommended: 20+ Polygon block
confirmations). Generate a random 32-byte nonce for this authorization. Deliver the nonce to
the user through the DApp's coordination channel.

**Then call on Midnight:**

```typescript
// authorize_deposit: creates the on-chain authorization the user will claim.
// amount is in micro-USDC (6 decimal places). 1 USDC = 1_000_000.
// nonce is the 32-byte value you generated and delivered to the user.
export circuit authorize_deposit(
  user_key: Bytes<32>,  // the depositor's Midnight public key commitment
  amount:   Uint<128>,  // confirmed deposit amount in micro-USDC
  nonce:    Bytes<32>   // unique per-authorization nonce
): []
```

```typescript
async function authorizeDeposit(userKey: Uint8Array, amount: bigint, nonce: Uint8Array) {
  return contract.impureCircuits.authorize_deposit(userKey, amount, nonce);
}
```

The user's `user_key` is their `own_public_key()` as derived by their Midnight Wallet. The
DApp registration flow must collect and store this when the user first connects their wallet.

---

### 5.2 Withdrawal Processing Loop

**Trigger:** the manager polls `total_tickets` and finds pending tickets. For each pending
ticket, the manager checks whether `total_liquid` is sufficient to cover the payout.

The payout amount the manager must send on Polygon is:

```typescript
function computePayout(ticket: WithdrawTicket, ledger: Ledger): bigint {
  const ta = totalAssets(ledger);
  const es = effectiveSupply(ledger);
  // Mirror of the on-chain formula. Floor division.
  return (ticket.shares * ta) / es;
}
```

**Off-chain first:** send the computed payout amount in USDC from the execution wallet to the
user's Polygon address. Wait for Polygon finality.

**Then call on Midnight:**

```typescript
// process_withdraw: marks the ticket filled and updates vault accounting.
// ticketKey is derived from the ticket's sequential index.
export circuit process_withdraw(
  ticket_key: Bytes<32>  // persistentHash(ticket.ticket_index)
): []
```

```typescript
async function processWithdraw(ticket: WithdrawTicket) {
  const key = ticketKey(ticket.ticket_index);
  return contract.impureCircuits.process_withdraw(key);
}
```

---

### 5.3 Capital Deployment Loop

**Trigger:** the probability model produces a positive allocation decision with a target market,
direction, and USDC amount.

**Float cap check first:** never proceed if deploying the requested amount would breach the cap.

```typescript
async function canDeploy(amount: bigint): Promise<boolean> {
  const ledger = await readLedger();
  return amount <= floatRemaining(ledger);
}
```

**Off-chain first:** execute the Polymarket trade on Polygon (USDC approval + CLOB API order).
Wait for the Polygon transaction to be confirmed and the order to be filled.

**Then call on Midnight:**

```typescript
// manager_withdraw: records that capital has been deployed externally.
// Increments float_outstanding, decrements total_liquid.
export circuit manager_withdraw(
  amount: Uint<128>  // micro-USDC deployed to Polymarket
): []
```

```typescript
async function recordDeployment(amount: bigint) {
  return contract.impureCircuits.manager_withdraw(amount);
}
```

---

### 5.4 Capital Return Loop

**Trigger:** a Polymarket position settles or is exited, and USDC is confirmed returned to the
execution wallet on Polygon.

**Off-chain first:** confirm the USDC has arrived in the execution wallet on Polygon with
sufficient finality.

**Then call on Midnight:**

```typescript
// manager_deposit: records that capital has returned to the vault.
// Decrements float_outstanding by min(amount, float_outstanding).
// Any excess above float_outstanding goes directly to total_liquid as profit.
export circuit manager_deposit(
  amount: Uint<128>  // micro-USDC returned from Polymarket
): []
```

```typescript
async function recordReturn(amount: bigint) {
  return contract.impureCircuits.manager_deposit(amount);
}
```

If the returned amount exceeds `float_outstanding` (profitable position), the excess raises
`total_liquid` and therefore raises the share price for all outstanding shares. This is the
profit distribution mechanism — no separate distribution step is required.

---

## 6. Vault Lifecycle Circuits

These are called once or infrequently. They are not part of any polling loop.

### `initialize_vault`

Called once on deployment. Sets the manager key and float cap.

```typescript
export circuit initialize_vault(
  initial_float_bps: Uint<16>  // e.g. 8000 for 80% cap
): []
```

### `set_float_cap`

Updates the float cap. Takes effect immediately on the next manager_withdraw check.

```typescript
export circuit set_float_cap(
  new_max_float_bps: Uint<16>
): []
```

### `update_manager` / `accept_manager`

Two-step manager key rotation. The current manager nominates a new key. The nominee then
calls `accept_manager` to prove control of the new key and complete the transfer.

```typescript
export circuit update_manager(new_key: Bytes<32>): []
export circuit accept_manager(): []
```

After `accept_manager` succeeds, all subsequent manager-gated calls must come from the wallet
whose seed produces the new key. Rebuild the manager wallet instance with the new seed.

---

## 7. Ordering Rules

These are hard rules. Violating them produces either corrupted vault accounting or unfulfilled
user obligations.

**Deposits:** confirm Polygon USDC received first, then call `authorize_deposit`. Never
authorize a deposit you have not yet received — the user can claim shares against an
authorization that has no backing funds.

**Withdrawals:** send USDC on Polygon first, then call `process_withdraw`. Never call
`process_withdraw` before the Polygon transfer is confirmed — the contract marks the obligation
fulfilled and the user has no recourse if the payout never arrives.

**Deployments:** deploy on Polygon first (Polymarket trade confirmed), then call
`manager_withdraw`. Calling `manager_withdraw` before the Polygon transaction confirms inflates
`float_outstanding` against capital that may not have actually moved.

**Returns:** receive USDC on Polygon first (settlement confirmed), then call `manager_deposit`.
Calling `manager_deposit` before the funds arrive inflates `total_liquid` and raises the share
price against capital that has not returned.

---

## 8. Polling Schedule

| Task | Recommended interval |
|---|---|
| Read ledger state | Every 12 seconds |
| Enumerate pending tickets | Every 12 seconds |
| Monitor Polygon for incoming deposits | Every Polygon block (~2 seconds) |
| Check open Polymarket positions for settlement | Every 60 seconds |
| Float cap compliance check | Before every deployment decision |

---

## 9. What the AI Manager Does Not Do

The following are handled entirely by the user through the frontend and Midnight Wallet. The
manager system must not call these circuits under any circumstances — they require the user's
private key to satisfy the ZK proof, and calling them from the manager wallet will fail:

- `claim_deposit` — user proves they are the authorized depositor and mints their own shares
- `request_withdraw` — user spends their own shielded share coins to create a ticket
- `cancel_withdraw` — user proves ownership of a ticket and reclaims their shares

The manager also has no visibility into individual user share balances, individual deposit
amounts after authorization is claimed, or any shielded coin data. That state is private to
each user's Midnight Wallet and is inaccessible by design.
