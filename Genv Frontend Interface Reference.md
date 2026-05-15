# GenV — Frontend Public Interface Reference

**Audience:** UI developer and DApp integration engineer.

**Purpose:** Everything you need to build the frontend and begin writing the Midnight JS SDK
integration. Internal vault mechanics are deliberately omitted. If anything changes during
contract implementation it will be in the circuit bodies, not in the public signatures or
ledger fields described here.

---

## 1. SDK Setup

The frontend connects to the Midnight network through three pieces: the Midnight Wallet browser
extension (which holds the user's private state and signs circuit calls), the Midnight Indexer
(which serves public ledger state as a read API), and the Proof Server (which generates ZK
proofs — runs locally in development, hosted in production).

The Compact compiler generates a fully-typed TypeScript module from the contract source. That
module exports the `Ledger` type (the shape of all public state), the `Contract` class (the
entry point for circuit calls), and witness stubs that the DApp implements. Import from the
compiled output:

```typescript
import {
  Contract,
  type Ledger,
  type DepositAuthorization,
  type WithdrawTicket,
  witnesses,
} from './contract/index.cjs';

import {
  WalletBuilder,
  NodeZkConfigProvider,
} from '@midnight-ntwrk/midnight-js-wallet';

import { indexerPublicDataProvider } from '@midnight-ntwrk/midnight-js-indexer-public-data-provider';
```

Instantiate the contract once and reuse it across the app:

```typescript
const zkConfig = new NodeZkConfigProvider(PROOF_SERVER_URL, contractConfig);
const wallet   = await WalletBuilder.buildFromSeed(INDEXER_URL, zkConfig, seed, 'sync');
const provider = indexerPublicDataProvider(INDEXER_URL, contractConfig);

const contract = new Contract(witnesses);
```

---

## 2. Public Vault State

All fields listed below are read from the Midnight Indexer via the contract's public data
provider. The compiled `Ledger` type gives you full TypeScript coverage. `Counter` fields come
back as `bigint`. `Boolean` comes back as `boolean`. `Maybe<T>` comes back as
`{ is_some: boolean; value: T }`. `Bytes<32>` comes back as `Uint8Array`.

```typescript
type Ledger = {
  // ── Capital accounting ────────────────────────────────────────────────────
  total_liquid:              bigint;   // USDC held liquid in the vault (micro-USDC, 6 dp)
  float_outstanding:         bigint;   // USDC currently deployed externally (micro-USDC)

  // ── Share supply ──────────────────────────────────────────────────────────
  total_share_supply:        bigint;   // live shielded share coins held by users
  shares_pending_withdrawal: bigint;   // shares burned into unfilled withdrawal tickets

  // ── Configuration ─────────────────────────────────────────────────────────
  max_float_bps:             number;   // float cap in basis points (e.g. 8000 = 80%)

  // ── Manager ───────────────────────────────────────────────────────────────
  manager_key:               Uint8Array;                    // Bytes<32>
  pending_manager_key:       { is_some: boolean; value: Uint8Array };

  // ── Withdrawal queue ──────────────────────────────────────────────────────
  total_tickets:             bigint;   // total tickets ever created; use to enumerate
  tickets:                   ContractMap; // keyed by ticket_key — query individually

  // ── Deposit authorizations ────────────────────────────────────────────────
  deposit_authorizations:    ContractMap; // keyed by auth_key — query individually

  // ── Guards ────────────────────────────────────────────────────────────────
  is_initialized:            boolean;
  first_deposit_processed:   boolean;
};
```

---

## 3. Derived Metrics

Compute these client-side from ledger values. Do not fetch them separately — they are not
stored on-chain.

```typescript
function deriveVaultMetrics(ledger: Ledger) {
  const totalAssets     = ledger.total_liquid + ledger.float_outstanding;
  const effectiveSupply = ledger.total_share_supply + ledger.shares_pending_withdrawal;

  // Share price in micro-USDC per share. Display as (sharePrice / 1_000_000) USDC.
  const sharePrice = effectiveSupply > 0n
    ? (totalAssets * 1_000_000n) / effectiveSupply
    : 1_000_000n; // 1.000000 USDC per share before first deposit

  // Float utilization as a percentage (0–100).
  const floatUtilizationPct = totalAssets > 0n
    ? Number((ledger.float_outstanding * 100n) / totalAssets)
    : 0;

  // Configured maximum float in micro-USDC.
  const maxFloat = (totalAssets * BigInt(ledger.max_float_bps)) / 10_000n;

  return { totalAssets, effectiveSupply, sharePrice, floatUtilizationPct, maxFloat };
}
```

---

## 4. Polling Strategy

Midnight has no event system. Poll the indexer on a fixed interval (suggested 10–15 seconds)
to detect state changes and refresh the UI.

```typescript
async function pollVaultState(provider: PublicDataProvider): Promise<Ledger> {
  return provider.queryContractState(CONTRACT_ADDRESS);
}

// On mount, start polling:
const interval = setInterval(async () => {
  const ledger = await pollVaultState(provider);
  updateUI(ledger);
}, 12_000);
```

For a user's own pending tickets, re-derive and query on each poll (see Section 6).

---

## 5. Amounts and Units

All on-chain amounts are in **micro-USDC** (6 decimal places). 1 USDC = 1,000,000 units.
Always work in `bigint` internally and convert to a human-readable string only for display.

```typescript
// Display helper: converts micro-USDC bigint to "1,234.56 USDC" string
function formatUSDC(microUsdc: bigint): string {
  const whole = microUsdc / 1_000_000n;
  const frac  = microUsdc % 1_000_000n;
  return `${whole.toLocaleString()}.${frac.toString().padStart(6, '0')} USDC`;
}

// Parse user input (string "500.00") to micro-USDC bigint
function parseUSDC(input: string): bigint {
  const [w, f = ''] = input.split('.');
  return BigInt(w) * 1_000_000n + BigInt(f.padEnd(6, '0').slice(0, 6));
}
```

---

## 6. User-Facing Circuits

These are the only three circuits the frontend calls on behalf of the user. All three require
the Midnight Wallet to be connected — the SDK routes them through the wallet for proof
generation and submission.

### 6.1 `claim_deposit`

Called after the manager confirms the user's USDC transfer on Polygon and provides a nonce.
The user enters the amount they sent and the nonce they received, and the circuit mints their
shielded share coins.

**Parameters**

| Name | Type | Description |
|---|---|---|
| `amount` | `bigint` | The deposit amount in micro-USDC. Must match the authorized amount exactly. |
| `nonce` | `Uint8Array` | 32-byte nonce provided by the manager. The user receives this via the DApp's coordination channel after their Polygon transfer is confirmed. |

**Returns** `ShieldedCoinInfo` — the minted share coin. The SDK stores this automatically in
the wallet's private state. The frontend does not need to handle the return value directly, but
should await the call and confirm success before updating the UI.

```typescript
async function claimDeposit(amount: bigint, nonce: Uint8Array) {
  const result = await contract.impureCircuits.claim_deposit(amount, nonce);
  // result.public contains the new ledger state snapshot post-claim
  return result;
}
```

**UX note:** the nonce coordination is off-chain. The recommended pattern is that after the
manager calls `authorize_deposit` on-chain, the DApp's backend records the nonce and displays
it to the user when they log in. The user does not type a hex string — the frontend fetches
it from the backend and passes it directly to the circuit. The coordination backend does not
exist yet at the time of writing, so stub this call with a hardcoded object during development
and wire it to the real API when the backend is built. The circuit call itself does not change
when that happens — only the source of the nonce changes.

### 6.2 `request_withdraw`

Burns the user's shielded share coins and creates a withdrawal ticket. The ticket is public
on-chain. The payout is processed asynchronously by the manager when the vault has sufficient
liquid balance.

**Parameters**

| Name | Type | Description |
|---|---|---|
| `sharesToWithdraw` | `bigint` | Number of shares to burn. Must be greater than zero and must not exceed the user's current shielded share balance (the wallet enforces this). |

**Returns** `ShieldedSendResult` — if the user burns a partial balance, the remainder comes
back as a change coin. The SDK stores both outputs in the wallet automatically.

```typescript
async function requestWithdraw(sharesToWithdraw: bigint) {
  const result = await contract.impureCircuits.request_withdraw(sharesToWithdraw);
  return result;
}
```

**UX note:** after calling this, poll `total_tickets` on the next interval — it will have
incremented by one. Derive the new ticket key from `total_tickets - 1` and display the ticket
in the user's pending list (see Section 7).

### 6.3 `cancel_withdraw`

Cancels a pending ticket and remints the shares back to the user. Only callable by the ticket
owner. Only valid if the ticket is neither filled nor already cancelled.

**Parameters**

| Name | Type | Description |
|---|---|---|
| `ticketKey` | `Uint8Array` | The 32-byte key of the ticket to cancel. Derived from the ticket index (see Section 7). |

**Returns** `ShieldedCoinInfo` — the reminted share coin, stored automatically in the wallet.

```typescript
async function cancelWithdraw(ticketKey: Uint8Array) {
  const result = await contract.impureCircuits.cancel_withdraw(ticketKey);
  return result;
}
```

---

## 7. Ticket Discovery and Display

Tickets are stored in the `tickets` Map. The Map is not iterable directly — you must derive
each key from its sequential index and query individually.

```typescript
import { persistentHash } from '@midnight-ntwrk/compact-runtime';
// NOTE: this import path is the most likely thing to shift when the compiled contract
// output lands. The SDK may expose persistentHash differently depending on compiler
// version. If the import fails, check the compiled contract's index.cjs exports first —
// it may re-export the hash utility directly.

// Derive the ticket key for a given index (0-based)
function ticketKey(index: bigint): Uint8Array {
  return persistentHash(index);
}

// Fetch all tickets and filter to the current user's pending ones
async function fetchUserTickets(
  provider: PublicDataProvider,
  totalTickets: bigint,
  userPublicKey: Uint8Array
): Promise<WithdrawTicket[]> {
  const tickets: WithdrawTicket[] = [];

  for (let i = 0n; i < totalTickets; i++) {
    const key    = ticketKey(i);
    const ticket = await provider.queryMapEntry('tickets', key) as WithdrawTicket;
    if (!ticket) continue;

    // Match by user_key — compare Uint8Array byte-by-byte
    const isOwner = ticket.user_key.every((b, idx) => b === userPublicKey[idx]);
    if (isOwner) tickets.push(ticket);
  }

  return tickets;
}
```

**`WithdrawTicket` shape**

```typescript
type WithdrawTicket = {
  user_key:        Uint8Array;  // owner's public key commitment
  shares:          bigint;      // shares burned into this ticket
  assets_snapshot: bigint;      // total_assets at time of request (display only)
  supply_snapshot: bigint;      // effective_supply at time of request (display only)
  ticket_index:    bigint;      // sequential index (0-based)
  is_filled:       boolean;     // true when the manager has processed the payout
  is_cancelled:    boolean;     // true if the user cancelled
  payout_amount:   bigint;      // micro-USDC paid out; 0 until filled
};
```

**Ticket states for UI display**

| `is_filled` | `is_cancelled` | Display state |
|---|---|---|
| `false` | `false` | Pending — show "Cancel" button |
| `true` | `false` | Filled — show payout amount, no action available |
| `false` | `true` | Cancelled — show as closed, no action available |

---

## 8. Reading a User's Share Balance

The user's share balance is private — it does not exist in the public ledger. It is held
entirely in the Midnight Wallet as one or more shielded coin UTXOs. The wallet exposes the
balance through the SDK:

```typescript
const shieldedBalance: bigint = await wallet.getShieldedBalance(CONTRACT_ADDRESS);
```

Display this as a raw share count. To show its estimated USDC value, multiply by the current
share price from Section 3:

```typescript
const estimatedValue = (shieldedBalance * sharePrice) / 1_000_000n;
```

This is an estimate because the actual payout is computed from live vault state at the moment
`process_withdraw` executes, not at the moment the user requests.

---

## 9. What to Display on the Main Dashboard

| Metric | How to get it |
|---|---|
| Total vault assets | `total_liquid + float_outstanding` |
| Share price | `(totalAssets * 1_000_000n) / effectiveSupply` |
| My share balance | `wallet.getShieldedBalance(CONTRACT_ADDRESS)` |
| My estimated position value | `(myBalance * sharePrice) / 1_000_000n` |
| Float utilization | `float_outstanding * 100n / totalAssets` |
| Float cap | `max_float_bps / 100` → display as percentage |
| My pending tickets | Derived from ticket enumeration (Section 7) |
| Vault initialized | `is_initialized` |

---

## 10. What the Frontend Does Not Do

The following operations are performed by the manager's backend system and are not part of
the frontend integration. You do not need to build UI for any of these. If a circuit is not
listed in Sections 6 above, it is not your concern:

- `authorize_deposit` — manager calls this after confirming the user's Polygon USDC transfer
- `process_withdraw` — manager calls this after sending the USDC payout on Polygon
- `manager_withdraw` — manager calls this when deploying capital to Polymarket
- `manager_deposit` — manager calls this when capital returns from Polymarket
- `initialize_vault` — one-time setup, called by manager on deployment
- `update_manager` / `accept_manager` — manager authority transfer
- `set_float_cap` — governance action by manager
