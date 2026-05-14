# GenV — Managed Vault Program: Midnight Design Document

**Author:** Yemi (Ikeh Chukwuka Favour)

**Date:** 15 May, 2026

**Status:** Pre-implementation. Every design decision committed to before a single line of
contract code is written.


## 1. Introduction

This document specifies every design decision made for the GenV Managed Vault, a
privacy-preserving, AI-managed yield vault built as a native Compact smart contract on the
Midnight blockchain. It covers the contract's complete ledger layout, the authorization model,
the shielded share token architecture, the withdrawal queue mechanics, the float cap system,
the underlying asset coordination strategy, and every arithmetic and rounding rule applied
throughout the program.

The vault accepts deposits of an underlying stable asset, issues proportionally priced shielded
shares to depositors, and delegates capital deployment to an autonomous AI decision engine that
identifies mispriced outcomes on Polymarket and similar prediction markets. Users redeem shares
through an asynchronous withdrawal queue that fills as liquid capital is available. The Midnight
ZK architecture ensures that individual deposit amounts, individual share balances, and the
manager's deployed positions remain private, while total vault assets, the current share price,
and compliance with the configured float cap remain publicly verifiable by any observer.

This document is written from Midnight first principles. Every decision is grounded in what Compact provides today: ledger declarations,
circuits, witness functions, the shielded token primitives in CompactStandardLibrary, the
`own_public_key()` authorization mechanism, `disclose()` for controlled data revelation,
`persistentHash` and `persistentCommit` for cryptographic commitments, and `assert` for all
error conditions. Features that Midnight does not currently support — contract-to-contract
calls, on-chain events, Uint256 arithmetic — are either worked around with documented
alternatives or explicitly deferred.

<p align="center">
  <img src="https://i.imgur.com/AZgxUSt.png" alt="genv-systems-architecture-diagram" />
</p>


## 2. Midnight Execution Model: What This Means for Vault Design

Before any design decision, the execution model must be understood because it governs every
structural choice.

In Compact, a smart contract is composed of three distinct layers. The **ledger** is the
contract's public on-chain state. Every field declared with `export ledger` is visible to any
observer on the network, stored persistently, and updated atomically during circuit execution.
The ledger is the only state that is canonical. **Circuits** are the contract's entry points
and internal helper functions. An exported circuit (prefixed with `export`) is callable from
a DApp's TypeScript code. When a circuit runs, it executes off-chain on the caller's machine,
generates a zero-knowledge proof that the execution followed the contract's rules, and submits
that proof along with the proposed ledger state updates to the network. The network verifies the
proof and applies the updates atomically if the proof is valid.
**Witnesses** are TypeScript functions implemented by the DApp that run alongside circuits during
proof generation. A witness can read the caller's private local state (their secret keys,
shielded coin data, or any other information they hold off-chain) and return values that the
circuit uses without those values ever appearing in the public ledger or the proof's public
inputs.

The consequence for vault design is significant. There is no transaction sender field, no
`msg.sender`, and no implicit caller identity. Authorization is always a fact that the caller
proves through the ZK circuit. The canonical pattern is that a user has a secret key known only
to them (provided through a witness), from which a public key commitment is derived inside the
circuit, and that derived key is compared against a stored value in the ledger. The
`own_public_key()` function provides this derived key inside any circuit, making authorization
checks concise and idiomatic. Any data that should remain private — a user's deposit amount, a
manager's deployment decision, a share coin's value — is handled entirely within the circuit
and witness layer, never disclosed to the ledger unless `disclose()` is explicitly called.

Midnight currently does not support contract-to-contract calls. This means the vault cannot
call into a separately deployed stablecoin contract to verify or execute token transfers. The
implications of this limitation and the design choices it forces are addressed fully in Section 9.


## 3. Privacy Boundary: What Is Public and What Is Private

Every field in the following table is classified by whether it lives in the public
ledger (visible to any observer, proven by ZK proofs to be correct) or in the private state of
a specific party (never appearing on-chain, known only to its owner).

| Datum | Classification | Reason |
|---|---|---|
| Total vault assets (liquid + float) | **Public** | Required for share price verification |
| Current share price | **Public** | Derived from public values; any observer can compute it |
| Total share supply (live + pending) | **Public** | Required for share price verification |
| Float outstanding | **Public** | Required for float cap compliance verification |
| Max float cap (basis points) | **Public** | Governance parameter; must be auditable |
| Manager public key | **Public** | Required to verify manager actions are authorized |
| Pending manager public key | **Public** | Required for two-step transfer verification |
| Total withdrawal tickets issued | **Public** | Crankers need this to discover pending tickets |
| Withdrawal ticket (shares, user key, index) | **Public** | Required for permissionless processing |
| Individual user share balance | **Private** | Held as a shielded UTXO coin; invisible on-chain |
| Individual deposit amount | **Private** | Never disclosed in circuit execution |
| Deposit authorization nonce | **Private** | Known only to manager and intended depositor |
| Manager's Polymarket positions | **Private** | On Polygon, structurally disconnected from Midnight |
| User's shielded coin nonce and Merkle index | **Private** | Wallet-held; never disclosed |

The strategy privacy for the AI manager derives not from anything the Midnight contract
enforces but from architectural separation. The vault records that some amount of capital is
deployed externally via `float_outstanding`. It does not record where. The Polymarket positions
are established through a Polygon wallet that is operationally separate from the Midnight
contract (for now). An observer of the Midnight ledger sees `float_outstanding` change, but no on-chain
record anywhere links that delta to specific Polymarket market IDs, position sizes, or exit
times. This structural separation is the competitive moat.


## 4. Compact Ledger State Layout

The complete public state of the vault is declared in the following ledger fields. All
amounts are in the base unit of the underlying token. Uint<128> is Compact's largest integer
type and is used for all token amounts. Counter is used for monotonically managed accumulators
where the increment/decrement interface is appropriate.

```compact
pragma language_version >= 0.16.0;
import CompactStandardLibrary;

// ── Manager Authority ─────────────────────────────────────────────────────────

// The derived public key of the current manager, stored as Bytes<32>.
// Set at initialization. All manager-gated circuits assert own_public_key() == manager_key.
export ledger manager_key: Bytes<32>;

// The derived public key of the manager nominee during a two-step authority transfer.
// None when no transfer is in progress.
export ledger pending_manager_key: Maybe<Bytes<32>>;

// ── Vault Configuration ───────────────────────────────────────────────────────

// The maximum percentage of total assets the manager may have deployed externally
// at any moment, expressed in basis points (1 bp = 0.01%). Range: 0 – 10000.
// Example: 8000 means the manager may deploy at most 80% of total assets.
export ledger max_float_bps: Uint<16>;

// ── Asset Accounting ──────────────────────────────────────────────────────────

// The vault's reported liquid balance of underlying tokens, in base units.
// Incremented by authorize_return and claim_deposit. Decremented by manager_withdraw
// and process_withdraw.
export ledger total_liquid: Counter;

// The amount of underlying tokens currently deployed externally by the manager.
// Incremented by manager_withdraw. Decremented by manager_deposit.
export ledger float_outstanding: Counter;

// ── Share Accounting ──────────────────────────────────────────────────────────

// The number of live shielded share coins currently held by users.
// Incremented when shares are minted (deposit). Decremented when shares are burned
// into a withdrawal ticket (request_withdraw). Incremented again on cancel_withdraw
// (new coin reminted to user).
export ledger total_share_supply: Counter;

// The number of shares that have been burned into pending withdrawal tickets
// but whose corresponding underlying assets have not yet been paid out.
// Incremented on request_withdraw. Decremented on process_withdraw and cancel_withdraw.
// NEVER modified by any other instruction.
//
// DESIGN INVARIANT: Every share price calculation uses
//   effective_supply = total_share_supply + shares_pending_withdrawal
// rather than total_share_supply alone. This ensures that remaining live
// shareholders experience no price distortion when tickets are created or cancelled.
// Price changes only occur at the economically correct moments: deposit (new capital
// enters) and process_withdraw (capital exits proportionally).
export ledger shares_pending_withdrawal: Counter;

// ── Deposit Authorization ─────────────────────────────────────────────────────

// A map from authorization key (Bytes<32>) to the authorization record.
// The authorization key is persistentHash([user_key, amount_bytes, nonce]).
// Created by the manager. Claimed once by the intended depositor.
export ledger deposit_authorizations: Map<Bytes<32>, DepositAuthorization>;

// ── Withdrawal Queue ──────────────────────────────────────────────────────────

// The total number of withdrawal tickets ever created. Used to assign a unique
// sequential index to each ticket and to allow off-chain agents to enumerate all
// tickets without iterating an unbounded structure.
export ledger total_tickets: Counter;

// A map from ticket key (Bytes<32>) to the ticket record.
// The ticket key is persistentHash(ticket_index as Bytes<32>).
// Tickets are never removed from the map. They are marked is_filled or is_cancelled.
export ledger tickets: Map<Bytes<32>, WithdrawTicket>;

// ── Initialization Guard ──────────────────────────────────────────────────────

// True after initialize_vault has been called. Guards against re-initialization.
export ledger is_initialized: Boolean;

// True after the first deposit has been processed. Guards first-deposit logic.
export ledger first_deposit_processed: Boolean;
```

### 4.1 Struct Definitions

```compact
struct DepositAuthorization {
  user_key:   Bytes<32>;   // own_public_key() of the intended depositor
  amount:     Uint<128>;   // authorized deposit amount in base units
  nonce:      Bytes<32>;   // unique nonce, generated by the manager, prevents replay
  is_claimed: Boolean;     // true after the user has successfully claimed this authorization
}

struct WithdrawTicket {
  user_key:        Bytes<32>;   // own_public_key() of the ticket owner
  shares:          Uint<128>;   // shares burned into this ticket
  assets_snapshot: Uint<128>;   // total_assets at request time (for analytics)
  supply_snapshot: Uint<128>;   // effective_supply at request time (for analytics)
  ticket_index:    Uint<64>;    // sequential index matching the ticket key derivation
  is_filled:       Boolean;     // true after process_withdraw completes
  is_cancelled:    Boolean;     // true after cancel_withdraw completes
  payout_amount:   Uint<128>;   // underlying tokens paid out; set on fill, 0 otherwise
}
```


## 5. Authorization Model

Compact provides no implicit caller identity. Authorization is always a claim that the caller
proves true inside the circuit.

The primary mechanism is `own_public_key()`, a standard library function available inside any
circuit that returns a `Bytes<32>` commitment derived from the caller's secret key and the
contract's state. This derived key is stable for a given user interacting with a given contract.
The manager's key is stored in `manager_key` at initialization. Every manager-gated circuit
begins with a single assertion:

```compact
circuit assert_manager(): [] {
  assert(own_public_key() == manager_key, "caller is not the vault manager");
}
```

No helper function can be bypassed because the ZK proof must demonstrate that the assertion
was satisfied. An invalid assertion causes proof generation to fail before anything reaches the
network.

For user-owned state (withdrawal tickets), the same pattern applies. When a ticket is created,
`own_public_key()` is stored in `ticket.user_key` via `disclose()`. When the user calls
`cancel_withdraw`, the circuit asserts `own_public_key() == ticket.user_key`. The ZK proof
guarantees this was true at the time of execution.

The `disclose()` wrapper is required whenever a value derived from private information (such as
`own_public_key()`, which is derived from the caller's secret key via the witness layer) is
written to a public ledger field. Compact's compiler enforces this statically: assigning a
witness-derived value to a ledger field without `disclose()` is a compile error. Every ledger
write in this contract that involves any caller-derived data is explicitly wrapped in `disclose()`.


## 6. Shielded Share Token Architecture

Share tokens in this vault are shielded UTXO coins issued using the `mintShieldedToken` circuit
from CompactStandardLibrary. They are not entries in any public
data structure. A user's share holding is a shielded coin that exists only in their wallet's
private state. The public ledger records only the aggregate counters `total_share_supply` and
`shares_pending_withdrawal`. No on-chain record connects a share balance to a specific user
public key, a specific deposit amount, or a specific deposit transaction.

### 6.1 Domain Separator

Every shielded token in Midnight's ZSwap model is identified by a domain separator: a `Bytes<32>`
value that distinguishes coins of one token type from coins of another. The vault derives its
share token domain separator deterministically from its own contract address, ensuring uniqueness
across all vault deployments:

```compact
circuit share_domain(): Bytes<32> {
  // kernel.self() returns this contract's ContractAddress.
  // persistentHash produces a stable 32-byte identifier.
  return persistentHash<ContractAddress>(kernel.self());
}
```

Any two vaults deployed from the same contract code but for different underlying assets will
have different domain separators because they have different contract addresses. This prevents
cross-vault coin confusion.

### 6.2 Minting Shares

When a user successfully claims a deposit authorization, the circuit calls `mintShieldedToken`
to create a new shielded coin of share value and delivers it to the user's ZSwap public key.
The user's public key is provided through a witness function:

```compact
witness depositorZswapKey(): ZswapCoinPublicKey;

circuit mint_shares_to_user(share_amount: Uint<128>, nonce: Bytes<32>): ShieldedCoinInfo {
  const recipient_key = depositorZswapKey();
  const coin = mintShieldedToken(
    disclose(share_domain()),
    disclose(share_amount),
    disclose(nonce),
    left<ZswapCoinPublicKey, ContractAddress>(disclose(recipient_key))
  );
  total_share_supply.increment(disclose(share_amount));
  return coin;
}
```

The returned `ShieldedCoinInfo` is a private value that only the depositor's wallet receives.
It is not stored in the ledger. The wallet must persist this coin info for future spending. The
nonce is derived deterministically from the deposit authorization nonce to give the wallet a
predictable seed:

```compact
circuit share_nonce(deposit_nonce: Bytes<32>, index: Uint<128>): Bytes<32> {
  return persistentHash<Vector<2, Bytes<32>>>(
    [deposit_nonce, (index as Field as Bytes<32>)]
  );
}
```

### 6.3 Burning Shares

When a user submits a withdrawal request, they burn their shielded share coin. The coin is
spent to `shieldedBurnAddress()`, destroying it. The user provides their committed share coin
via a witness as `QualifiedShieldedCoinInfo` (the coin was minted in a prior transaction and
is now committed to the Merkle tree). If they wish to burn only a portion of their coin's
value, `sendShielded` handles the partial spend and returns the remainder as a change coin to
the user:

```compact
witness userShareCoin(): QualifiedShieldedCoinInfo;
witness withdrawNonce(): Bytes<32>;

circuit burn_shares(shares_to_burn: Uint<128>): ShieldedSendResult {
  const coin = userShareCoin();
  // sendShielded burns shares_to_burn to the burn address.
  // The remaining (coin.value - shares_to_burn) is returned as a change coin
  // to the original coin owner (the user), captured in ShieldedSendResult.change.
  // The wallet MUST persist the change coin for future spending.
  return sendShielded(
    disclose(coin),
    shieldedBurnAddress(),
    disclose(shares_to_burn)
  );
}
```

The `ShieldedSendResult` returned from this helper is the return value of `request_withdraw`.
The wallet receives it, extracts the change coin (the user's remaining shares that were not
withdrawn), and stores it. The burned shares leave `total_share_supply` and enter
`shares_pending_withdrawal`.

### 6.4 Reminting on Cancellation

When a user cancels a pending withdrawal ticket, their shares were already burned into that
ticket. The circuit remints a new shielded coin of the same share value back to the user. This
is not a redistribution — it is a precise restoration of the burned amount:

```compact
circuit remint_shares_to_user(shares: Uint<128>, nonce: Bytes<32>): ShieldedCoinInfo {
  const coin = mintShieldedToken(
    disclose(share_domain()),
    disclose(shares),
    disclose(nonce),
    left<ZswapCoinPublicKey, ContractAddress>(disclose(depositorZswapKey()))
  );
  total_share_supply.increment(disclose(shares));
  return coin;
}
```

The nonce for the reminted coin is derived from the ticket key to ensure determinism and avoid
nonce reuse across the vault's history.


## 7. Effective Supply: The Core Accounting Invariant

Every share price calculation in the vault uses **effective supply** rather than
`total_share_supply` alone. This is the most important single invariant in the contract.

```compact
circuit effective_supply(): Uint<128> {
  // total_share_supply: live shielded coins held by users.
  // shares_pending_withdrawal: burned shares in pending tickets, awaiting payout.
  // Their sum is the economically meaningful supply for pricing purposes.
  return (total_share_supply as Uint<128>) + (shares_pending_withdrawal as Uint<128>);
}

circuit total_assets(): Uint<128> {
  return (total_liquid as Uint<128>) + (float_outstanding as Uint<128>);
}
```

The rationale is as follows. When a user burns their shares into a withdrawal ticket, those
shares are gone as shielded coins but the corresponding vault assets have not yet been paid out.
If `total_share_supply` alone were used for pricing, the share price would jump for remaining
holders the moment any ticket is created, because fewer shares would appear to be outstanding
while the same assets remain. This is economically wrong: the burned shares still represent a
claim on vault assets, and remaining holders should not receive a windfall merely because someone
entered the withdrawal queue. Including `shares_pending_withdrawal` in the effective supply
neutralizes this distortion. The share price changes only when assets actually enter the vault
(deposit, manager return) or leave it (withdrawal payout). This behavior is verified in the
math reference in Section 24.


## 8. Underlying Asset Coordination and the MVP Architecture

Midnight currently does not support contract-to-contract calls. A vault circuit cannot call
into a separately deployed stablecoin contract to verify or execute a token transfer. This is
a real constraint.

The chosen approach for the MVP is a **manager-authorized deposit** model. Actual underlying
token movement occurs off-chain. For the MVP demonstration, this means USDC on Polygon flowing
into and out of the manager's designated Polygon wallet. The Midnight vault contract serves as
the trustless accounting and authorization layer: it enforces share math, float cap compliance,
manager authority, and the ticket queue. It does not custody underlying tokens on-chain in the
MVP.

The trust assumption this creates is explicit and documented: the manager is trusted to
authorize only deposits that correspond to real off-chain value transfers, and to execute
withdrawal payouts off-chain when marking tickets filled. The on-chain enforcement protects
users from the manager exceeding the float cap, minting shares without a matching deposit
authorization, or processing a withdrawal without having the liquid balance to cover it. The
program cannot protect against a manager who refuses to authorize a real deposit or refuses
to pay out a filled ticket off-chain. This is the fundamental trust model of any managed vault
and is consistent with the ERC-7540 design, where the manager's honesty is a stated assumption
and the contract enforces structural rules rather than asset custody.

In a future iteration, once Midnight supports contract-to-contract and cross-chain calls, the manager
authorization circuit would be replaced by a direct CPI into a Midnight-native stablecoin
contract, and as well as the off-chain coordination layer would be eliminated. The design of the remaining
contracts is forward-compatible with this upgrade.


## 9. Design Decision 1 — First Depositor and the Share Inflation Attack

**Decision: on the first deposit, shares are minted 1:1 with the deposit amount, a minimum
first deposit of 1000 base units scaled by the underlying token's decimal precision is
enforced, and a fixed number of dead shares equal to that minimum are immediately burned to
`shieldedBurnAddress()`, permanently removing them from the redeemable supply.**

The share inflation attack is a class of exploit that targets the first deposit into a vault
when `effective_supply == 0`. The formula `shares_out = deposit * effective_supply / total_assets`
is undefined at this boundary. Any special case introduced to handle it is a potential attack
vector. The canonical attack proceeds as follows: an adversary deposits a dust amount to become
the sole first depositor and receive a trivially small number of shares, then directly inflates
`total_assets` by transferring underlying tokens to the vault without going through the deposit
circuit. The next legitimate depositor's `shares_out` rounds down to zero, meaning they
transfer real value and receive nothing. The attacker redeems the only outstanding shares and
claims the entire vault balance including the legitimate depositor's funds.

The defense against this attack operates on two fronts. The first front is the 1:1 initial
mint: on the first deposit, `shares_out = deposit_amount` exactly, with no division involved.
This establishes an unambiguous starting price of 1 share per base unit of underlying token
and removes the undefined-formula boundary condition entirely.

The second front is the permanent burning of dead shares. A fixed amount of shares equal to
`MINIMUM_FIRST_DEPOSIT` (in scaled base units) is minted and immediately burned to
`shieldedBurnAddress()`. These shares are gone: no wallet can ever hold them, no circuit can
ever redeem them. Their corresponding underlying token value stays in the vault permanently.
This permanently locks a non-trivial asset base into the vault that contributes to
`total_assets` but is excluded from `effective_supply`. An attacker attempting the inflation
attack must now inflate the price relative to a base that includes the dead shares' locked
value, raising the cost of the attack proportionally.

```compact
const MINIMUM_FIRST_DEPOSIT: Uint<128> = 1_000; // in raw base units before decimal scaling

circuit process_first_deposit(
  deposit_amount: Uint<128>,
  decimals: Uint<8>,
  deposit_nonce: Bytes<32>
): ShieldedCoinInfo {
  // Scale the minimum by the token's decimal precision.
  const scale: Uint<128> = pow10(decimals as Uint<128>);
  const min_deposit: Uint<128> = MINIMUM_FIRST_DEPOSIT * scale;

  assert(deposit_amount >= min_deposit, "first deposit below minimum");
  assert(!first_deposit_processed, "first deposit already processed");

  // Dead shares: minted then burned immediately to the shielded burn address.
  // They leave total_share_supply at zero (never incremented by the burn path).
  const dead_amount: Uint<128> = min_deposit;
  const dead_nonce: Bytes<32> = persistentHash<Bytes<32>>(deposit_nonce);

  // Mint dead shares to burn address. These are gone permanently.
  mintShieldedToken(
    disclose(share_domain()),
    disclose(dead_amount),
    disclose(dead_nonce),
    shieldedBurnAddress()
  );
  // Dead shares are not counted in total_share_supply because they are not
  // live coins. Their assets remain in total_liquid (deposited but unredeemable).

  // User receives the rest as a live shielded coin.
  const user_shares: Uint<128> = deposit_amount - dead_amount;
  assert(user_shares > 0, "deposit too small after dead share deduction");

  const user_nonce: Bytes<32> = share_nonce(deposit_nonce, 0);
  const user_coin = mint_shares_to_user(user_shares, user_nonce);

  // Update accounting.
  total_liquid.increment(disclose(deposit_amount));
  first_deposit_processed = disclose(true);

  return user_coin;
}
```

After this call, `total_assets() == deposit_amount` and `effective_supply() == user_shares ==
deposit_amount - dead_amount`. The share price is slightly above 1 (equal to `deposit_amount /
(deposit_amount - dead_amount)`), reflecting that the dead shares' underlying value is
distributed across the live shares. Every subsequent depositor enters at this price or above.


## 10. Design Decision 2 — Two-Step Deposit Authorization

**Decision: deposits proceed in two steps. The manager first calls `authorize_deposit` to create
an on-chain authorization record for a specific user and amount. The user then calls
`claim_deposit` to atomically verify the authorization, mint their shares, and mark the
authorization as consumed.**

This design is a direct consequence of the cross-chain interoperability gap between Midnight and Polygon. The underlying asset is USDC, which lives on Polygon because the vault's yield source (Polymarket) settles in USDC on Polygon. Midnight has no bridge to Polygon and no mechanism for a Compact circuit to observe or verify external chain state, so the user's USDC transfer and the minting of shares cannot be atomic. The manager bridges this gap as a trusted cross-chain coordinator: they confirm the Polygon transfer and attest to it on-chain via `authorize_deposit`. The authorization record is that attestation. The user claiming it proves they are the intended recipient through `own_public_key()`, and the nonce ensures each authorization is consumed exactly once. Midnight's lack of contract-to-contract calls is a secondary constraint, it means even once a bridge exists, the vault circuit cannot call into it without C2C support. When both a Midnight-Polygon bridge and C2C calls ship, the two-step process collapses into one atomic operation and the manager trust assumption disappears. The share accounting logic carries forward unchanged.

```compact
// Step 1: Manager creates the authorization. Off-chain, the manager has confirmed
// that the user sent deposit_amount of underlying tokens to the designated address.
export circuit authorize_deposit(
  user_key:        Bytes<32>,   // intended depositor's public key commitment
  amount:          Uint<128>,   // confirmed deposit amount in base units
  nonce:           Bytes<32>    // unique nonce generated by the manager for this authorization
): [] {
  assert_manager();
  assert(is_initialized, "vault not initialized");
  assert(amount > 0, "zero amount");

  // Authorization key is a hash of all three identifying fields.
  // It is unique per (user, amount, nonce) triple.
  const auth_key: Bytes<32> = persistentHash<Vector<3, Bytes<32>>>(
    [user_key, (amount as Field as Bytes<32>), nonce]
  );

  // Prevent re-authorization at the same key (manager cannot accidentally overwrite
  // a valid unclaimed authorization).
  const existing = deposit_authorizations.lookup(auth_key);
  assert(!existing.is_some, "authorization already exists at this key");

  deposit_authorizations.insert(
    disclose(auth_key),
    DepositAuthorization {
      user_key:   disclose(user_key),
      amount:     disclose(amount),
      nonce:      disclose(nonce),
      is_claimed: disclose(false),
    }
  );
}

// Step 2: User claims the authorization by proving they are the intended recipient.
// Returns a ShieldedCoinInfo representing their minted shares.
export circuit claim_deposit(
  amount: Uint<128>,   // must match the authorized amount
  nonce:  Bytes<32>    // must match the nonce the manager provided off-chain
): ShieldedCoinInfo {
  assert(is_initialized, "vault not initialized");
  assert(amount > 0, "zero amount");

  // Reconstruct the authorization key from the caller's identity and the provided fields.
  const caller_key: Bytes<32> = disclose(own_public_key());
  const auth_key: Bytes<32> = persistentHash<Vector<3, Bytes<32>>>(
    [caller_key, (amount as Field as Bytes<32>), nonce]
  );

  const auth_maybe = deposit_authorizations.lookup(auth_key);
  assert(auth_maybe.is_some, "no authorization found");

  const auth = auth_maybe.value;
  assert(!auth.is_claimed, "authorization already claimed");
  assert(auth.user_key == caller_key, "caller is not the authorized depositor");
  assert(auth.amount == amount, "amount does not match authorization");

  // Mark authorization as consumed before any state-mutating operation.
  deposit_authorizations.insert(
    disclose(auth_key),
    DepositAuthorization {
      user_key:   auth.user_key,
      amount:     auth.amount,
      nonce:      auth.nonce,
      is_claimed: disclose(true),
    }
  );

  // Branch on first deposit vs subsequent deposits.
  if (!first_deposit_processed) {
    return process_first_deposit(amount, 6, nonce); // 6 decimals for USDC-equivalent
  }

  // Normal deposit: compute shares using the share price formula.
  const ta: Uint<128> = total_assets();
  const es: Uint<128> = effective_supply();

  // shares_out = floor(amount * effective_supply / total_assets)
  // Both sides of the multiplication are Uint<128>. At practical vault sizes
  // (amounts below 10^15 base units, supply below 10^15) the product fits within Uint<128>.
  const shares_out: Uint<128> = (amount * es) / ta;
  assert(shares_out > 0, "deposit produces zero shares; amount too small relative to price");

  const user_nonce: Bytes<32> = share_nonce(nonce, (total_tickets as Uint<128>));
  const user_coin = mint_shares_to_user(shares_out, user_nonce);

  total_liquid.increment(disclose(amount));

  return user_coin;
}
```


## 11. Design Decision 3 — Share Price Timing: Processing Time

**Decision: the payout amount for a withdrawal ticket is computed at the time `process_withdraw`
executes, using the vault's live `total_assets()` and `effective_supply()` at that moment,
not the values at the time `request_withdraw` was called.**

This is the economically honest representation of a managed yield vault. A user who submits a
withdrawal ticket and whose ticket sits in the queue while the vault generates profit from
successful prediction market positions will receive that profit in their payout. A user whose
ticket sits in a queue while the vault is temporarily at a drawdown receives a reduced payout.
The user is a proportional owner of whatever the vault is worth at settlement, not the holder
of a fixed price promise made at an arbitrary earlier time.

Processing-time pricing allows the vault to compound fully during the queue period without
needing to set aside a fixed reserve per ticket. The vault's capital can continue working while
tickets wait, and the settlement correctly reflects the outcome of that work.

The `assets_snapshot` and `supply_snapshot` fields stored in the ticket serve as analytics
records only. They document what the price was when the user entered the queue, allowing an
off-chain indexer to compute how much price movement occurred during the queue period. They do
not affect the payout calculation.


## 12. Design Decision 4 — Share Burning on Request, Not Escrow

**Decision: on `request_withdraw`, the user's shielded share coin is burned (sent to
`shieldedBurnAddress()`) immediately. The burned shares enter `shares_pending_withdrawal`.
They do not remain as live shielded coins. The `effective_supply()` calculation includes
`shares_pending_withdrawal` to preserve price neutrality for remaining holders.**

True shielded coin escrow in a Compact contract, where the contract itself holds a shielded
coin that it later spends to process or cancel a withdrawal, requires the contract to prove
it controls the coin at spend time. While Midnight's `mintShieldedToken` can mint to a
`ContractAddress`, spending a contract-owned shielded coin requires knowledge of the spending
nonce and the coin's Merkle index at spend time. Managing this nonce deterministically and
retrieving the Merkle index at processing time introduces significant complexity in the witness
layer without providing any additional economic correctness over the counter-based approach.

The counter-based approach is both simpler and equivalent. `shares_pending_withdrawal` is an
exact accounting of shares that have been destroyed as live coins but whose assets have not yet
been released. Including this count in `effective_supply()` means the share price is computed
as if those shares were still live, preserving price neutrality. The math section in Section 24
provides a step-by-step trace confirming that price is unchanged for remaining holders when a
ticket is created, and changes correctly only when a ticket is processed.

Cancellation remints the shares using `mintShieldedToken` with a deterministic nonce derived
from the ticket key, ensuring the user receives a new shielded coin of identical value. The
nonce seed for the reminted coin differs from the original deposit nonce, preventing any
correlation with the user's deposit history.


## 13. Design Decision 5 — Withdrawal Ticket Queue Structure

**Decision: withdrawal tickets are stored in the `tickets` Map keyed by
`persistentHash<Uint<64>>(ticket_index)` where `ticket_index` is the value of
`total_tickets` at the moment the ticket is created, after which `total_tickets` is
incremented.**

```compact
export circuit request_withdraw(
  shares_to_withdraw: Uint<128>
): ShieldedSendResult {
  assert(is_initialized, "vault not initialized");
  assert(shares_to_withdraw > 0, "zero shares");

  // Capture the caller's key for ownership verification at cancel time.
  const caller_key: Bytes<32> = disclose(own_public_key());

  // Record a snapshot of vault state for analytics.
  const ta: Uint<128> = total_assets();
  const es: Uint<128> = effective_supply();

  // Burn the user's shares. sendShielded returns the change coin (remaining unburned
  // shares) as a ShieldedSendResult. The wallet must persist the change coin.
  const burn_result: ShieldedSendResult = burn_shares(shares_to_withdraw);

  // Update share accounting.
  total_share_supply.decrement(disclose(shares_to_withdraw));
  shares_pending_withdrawal.increment(disclose(shares_to_withdraw));

  // Derive the ticket key from the current ticket counter value.
  const ticket_idx: Uint<64> = total_tickets as Uint<64>;
  const ticket_key: Bytes<32> = persistentHash<Uint<64>>(ticket_idx);

  // Create the ticket record.
  tickets.insert(
    disclose(ticket_key),
    WithdrawTicket {
      user_key:        caller_key,
      shares:          disclose(shares_to_withdraw),
      assets_snapshot: disclose(ta),
      supply_snapshot: disclose(es),
      ticket_index:    disclose(ticket_idx),
      is_filled:       disclose(false),
      is_cancelled:    disclose(false),
      payout_amount:   disclose(0 as Uint<128>),
    }
  );

  total_tickets.increment(1);

  // Return the burn result so the wallet receives the change coin.
  return burn_result;
}
```

Any off-chain agent — the user's frontend, the manager's AI agent, a third-party cranker —
can enumerate all tickets by iterating ticket indices from 0 to `total_tickets - 1`, computing
`persistentHash<Uint<64>>(index)` for each, and looking up the result in the `tickets` Map.
This requires no stored index structure and no iteration over the Map itself.


## 14. Design Decision 6 — Queue Ordering and Partial Fills

**Decision: FIFO ordering is not enforced. Any pending ticket can be processed by the manager
at any time. Partial fills are not supported: a ticket is either filled in full or left pending.**

FIFO enforcement would require an on-chain linked list or sorted queue. Both structures create
write contention on every ticket creation and processing operation and add significant complexity
to the contract with no corresponding economic benefit. The manager processing an out-of-order
ticket does not harm any user: a user whose ticket is skipped remains in the queue with their
shares still pending. The manager has a strong incentive to fill tickets in an order that
maximizes vault liquidity utilization, which in practice means filling smaller tickets when only
limited liquidity is available rather than blocking all withdrawals behind one large ticket that
the vault cannot immediately cover.

Users who want to improve their fill probability can split a large withdrawal intent into
multiple smaller tickets submitted separately, each covering a portion of their total shares.
Smaller tickets are more likely to be fillable from whatever liquid balance exists at any given
moment.

```compact
export circuit process_withdraw(ticket_key: Bytes<32>): [] {
  assert_manager();

  const ticket_maybe = tickets.lookup(disclose(ticket_key));
  assert(ticket_maybe.is_some, "ticket not found");

  const ticket = ticket_maybe.value;
  assert(!ticket.is_filled, "ticket already filled");
  assert(!ticket.is_cancelled, "ticket is cancelled");

  // Compute payout at current processing-time prices.
  const ta: Uint<128> = total_assets();
  const es: Uint<128> = effective_supply();

  // amount_out = floor(shares * total_assets / effective_supply)
  const amount_out: Uint<128> = (ticket.shares * ta) / es;

  // All or nothing. If the vault cannot cover the full payout, the instruction
  // fails and the ticket remains pending. No partial fills.
  assert((total_liquid as Uint<128>) >= amount_out, "insufficient liquid balance");

  // Update accounting.
  shares_pending_withdrawal.decrement(disclose(ticket.shares));
  total_liquid.decrement(disclose(amount_out));

  // Mark the ticket filled with the actual payout amount recorded for analytics.
  tickets.insert(
    disclose(ticket_key),
    WithdrawTicket {
      user_key:        ticket.user_key,
      shares:          ticket.shares,
      assets_snapshot: ticket.assets_snapshot,
      supply_snapshot: ticket.supply_snapshot,
      ticket_index:    ticket.ticket_index,
      is_filled:       disclose(true),
      is_cancelled:    disclose(false),
      payout_amount:   disclose(amount_out),
    }
  );

  // The actual transfer of underlying tokens to the user's designated off-chain
  // address is the manager's responsibility and occurs off-chain. The contract
  // records the payout amount; the manager executes the transfer.
}
```


## 15. Design Decision 7 — Float Cap Under Stress

**Decision: when `float_outstanding` already exceeds the cap (due to vault TVL shrinkage from
withdrawals processed while the manager was deployed), new manager withdrawals are blocked.
No action is taken against existing float, user deposits are not blocked, and no recall is
forced.**

Blocking new manager withdrawals is the only lever the contract can pull. The program has no
mechanism to compel the manager to return capital from external positions. New user deposits
are not blocked because they increase `total_assets`, which directly lowers the float ratio
and helps restore compliance. Forcing deposits to stop during a stressed period would damage
the vault at exactly the moment additional liquidity is most needed.

```compact
export circuit manager_withdraw(amount: Uint<128>): [] {
  assert_manager();
  assert(is_initialized, "vault not initialized");
  assert(amount > 0, "zero amount");
  assert((total_liquid as Uint<128>) >= amount, "insufficient liquid balance");

  const ta: Uint<128> = total_assets();

  // max_float = floor(total_assets * max_float_bps / 10000)
  // Rounds down, giving the manager a conservatively lower ceiling.
  const max_float: Uint<128> = (ta * (max_float_bps as Uint<128>)) / 10_000;

  const new_float: Uint<128> = (float_outstanding as Uint<128>) + amount;

  // If float_outstanding is already above max_float due to TVL shrinkage,
  // this assertion fails for any nonzero amount, blocking further deployment
  // until the manager returns enough capital to bring the ratio back within bounds.
  assert(new_float <= max_float, "float cap exceeded");

  total_liquid.decrement(disclose(amount));
  float_outstanding.increment(disclose(amount));
}
```


## 16. Design Decision 8 — Manager-Gated Capital Return

**Decision: `manager_deposit` is gated to the manager. Only the manager may report that capital has been returned to the vault. The circuit validates that the caller is the manager, that the returned amount is non-zero, and decrements `float_outstanding` by the lesser of the returned amount and the current `float_outstanding` balance.**

Because the underlying asset is USDC on Polygon and not a token held by the Midnight contract, the `amount` parameter is an attestation rather than a verified transfer. There is no on-chain evidence the circuit can inspect to confirm that any real USDC moved. The manager gate is what gives the reported amount meaning — only the manager has ground truth about what Polygon actually returned from a Polymarket settlement or position exit, and only they can truthfully attest to it on-chain.

If a Polymarket position closed profitably and the returned amount exceeds the original `float_outstanding`, the circuit handles this gracefully rather than reverting. The excess above `float_outstanding` is credited directly to `total_liquid`, raising `total_assets` and appreciating the share price for all outstanding shares. Returning excess capital is profit distribution, not an error condition.

```compact
export circuit manager_deposit(amount: Uint<128>): [] {
  assert_manager();
  assert(is_initialized, "vault not initialized");
  assert(amount > 0, "zero amount");

  const current_float: Uint<128> = float_outstanding as Uint<128>;
  const float_decrement: Uint<128> =
    if (amount <= current_float) { amount } else { current_float };

  float_outstanding.decrement(disclose(float_decrement));
  total_liquid.increment(disclose(amount));
}
```


## 17. Design Decision 9 — Rounding Direction

**Decision: every division in the vault's share arithmetic truncates toward zero, which for
positive integers is equivalent to flooring. This is applied uniformly at every calculation
site. Every division site in the contract carries a comment documenting the rounding direction
and its consequence.**

The uniform direction of rounding is toward the vault and away from the user. When a depositor
receives shares, they receive the floor of the mathematical result. When a withdrawer receives
underlying tokens, they receive the floor of their proportional claim. When the float cap is
computed, the manager receives the floor of the allowed amount. The fractional amounts that do
not transfer in any of these calculations remain in the vault as unclaimable dust, which
imperceptibly raises the share price over time. This is the same convention used by ERC-4626
and the Uniswap V2 share model, and it is the convention that prevents rounding from being
exploited in any direction against the vault.

Rounding toward the vault is the conservative direction in every case. A depositor cannot
receive more shares than they are entitled to by gaming repeated small deposits. A withdrawer
cannot extract more assets than their proportional claim by manipulating timing. The float cap
rounds down, giving the manager slightly less deployment room than the theoretical maximum,
which is the safe direction for user protection.


## 18. Design Decision 10 — Spam Prevention

**Decision: no program-level minimum share threshold is enforced on withdrawal tickets.
Spam deterrence relies on the DUST transaction fee that every circuit call costs on the
Midnight network.**

Every interaction with the Midnight network requires spending DUST, the network's resource
token generated by holding NIGHT. Creating a withdrawal ticket requires submitting a circuit
call, which costs DUST. Creating 1000 spam tickets requires 1000 DUST-paying transactions.
This is a meaningful economic deterrent for any rational actor.

A program-level minimum share threshold would exclude legitimate small users and create a
UX cliff where a user with a small but valid share balance cannot submit a withdrawal. Since
DUST pricing already deters spam through a continuous cost model rather than a binary
threshold, the program-level threshold is redundant and harmful to accessibility. We therefore
do not implement it.

The one guard that does exist is `assert(shares_to_withdraw > 0, "zero shares")` which
prevents zero-cost ticket creation.


## 19. Design Decision 11 — Two-Step Manager Authority Transfer

**Decision: changing the vault manager requires two on-chain operations: the current manager
nominates a candidate by storing their public key in `pending_manager_key`, and the candidate
accepts by calling `accept_manager`, which proves they control the nominated key through
`own_public_key()` and atomically commits the transfer.**

A single-step transfer that immediately commits `manager_key = new_key` cannot be reversed
if the new key is unreachable or was specified incorrectly. Since the manager key controls
capital deployment for the entire vault, an irrecoverable manager key is a catastrophic and
permanent failure. The two-step approach is the only design that provides a safety net: the
current manager retains control until the nominee proves they can sign with the nominated key,
at which point the transfer is atomic and final.

```compact
// Step 1: Current manager nominates a candidate.
export circuit update_manager(new_manager_key: Bytes<32>): [] {
  assert_manager();
  assert(is_initialized, "vault not initialized");
  pending_manager_key = disclose(some<Bytes<32>>(new_manager_key));
}

// Step 2: Nominee accepts. own_public_key() proves they control the nominated key.
export circuit accept_manager(): [] {
  assert(is_initialized, "vault not initialized");

  const pending_maybe = pending_manager_key;
  assert(pending_maybe.is_some, "no pending manager nomination");

  const pending_key: Bytes<32> = pending_maybe.value;
  assert(own_public_key() == pending_key, "caller is not the nominated manager");

  // Atomic commitment: update manager_key, clear pending_manager_key.
  manager_key = disclose(own_public_key());
  pending_manager_key = none<Bytes<32>>();
}
```

To cancel a pending nomination, the current manager calls `update_manager` with their own
current key as the argument, which overwrites `pending_manager_key` with themselves. They then
call `accept_manager` to accept their own nomination, which clears `pending_manager_key`
without transferring authority. This is an intentional property of the two-step design.


## 20. Design Decision 12 — Cancellation

**Decision: `cancel_withdraw` is implemented. Only the ticket owner can cancel. Cancellation
remints a new shielded share coin of the original share amount to the user.**

Because shares are burned (not escrowed as live coins) when a ticket is created, cancellation
requires reminting rather than returning an existing coin. The reminted coin is a new shielded
UTXO with a deterministic nonce derived from the ticket key, ensuring the user always receives
their shares back under a predictable nonce seed without requiring any stored state beyond the
ticket record itself.

```compact
export circuit cancel_withdraw(
  ticket_key:   Bytes<32>,
  cancel_nonce: Bytes<32>   // nonce for the reminted share coin, provided by user
): ShieldedCoinInfo {
  const ticket_maybe = tickets.lookup(disclose(ticket_key));
  assert(ticket_maybe.is_some, "ticket not found");

  const ticket = ticket_maybe.value;
  assert(!ticket.is_filled, "ticket already filled");
  assert(!ticket.is_cancelled, "ticket already cancelled");

  // Only the ticket owner can cancel.
  assert(own_public_key() == ticket.user_key, "caller is not the ticket owner");

  // Update accounting: shares leave pending state and return to live supply.
  shares_pending_withdrawal.decrement(disclose(ticket.shares));
  // total_share_supply is incremented by remint_shares_to_user below.

  // Mark ticket cancelled.
  tickets.insert(
    disclose(ticket_key),
    WithdrawTicket {
      user_key:        ticket.user_key,
      shares:          ticket.shares,
      assets_snapshot: ticket.assets_snapshot,
      supply_snapshot: ticket.supply_snapshot,
      ticket_index:    ticket.ticket_index,
      is_filled:       disclose(false),
      is_cancelled:    disclose(true),
      payout_amount:   disclose(0 as Uint<128>),
    }
  );

  // Remint shares to the user. The deterministic nonce uses the ticket_key as seed,
  // disambiguated with the cancel_nonce to prevent correlation with the original deposit.
  const remint_nonce: Bytes<32> = persistentHash<Vector<2, Bytes<32>>>(
    [ticket_key, cancel_nonce]
  );
  return remint_shares_to_user(ticket.shares, remint_nonce);
}
```

Cancellation is restricted to the ticket owner. No manager, operator, or third party can
cancel a user's ticket. Cancellation returns shares, not underlying assets, and only the owner
should be empowered to reverse a decision that affects their position in the vault.


## 21. Design Decision 13 — `float_outstanding` Decrement Cap

**Decision: in `manager_deposit`, the decrement to `float_outstanding` is clamped at its
current value via `min(amount, float_outstanding)`. If the deposited amount exceeds
`float_outstanding`, the excess is credited to `total_liquid` without attempting to decrement
`float_outstanding` below zero.**

A Counter in Compact cannot go below zero. If the manager calls `manager_deposit` with an
amount that exceeds `float_outstanding` — because they are returning more than they originally
withdrew, as profit — the excess is simply a net addition to `total_liquid`. This increases
`total_assets` and appreciates all outstanding shares. The circuit does not treat excess
deposits as an error. This is the correct behavior and is explicitly documented in the circuit
comment so that no future reviewer mistakes it for an accounting bug.


## 22. Design Decision 14 — Error Handling

**Decision: every error condition in the contract is an `assert` statement with a descriptive
message string. There are no silent failures, no unchecked arithmetic operations, and no
code paths that allow an invalid state to be committed to the ledger.**

Compact's `assert(condition, "message")` terminates circuit execution and ZK proof generation
if the condition is false. No ledger updates from that circuit execution are applied. Every assertion in this contract is a
deliberate guard on a specific invariant, and every assertion message is written to be
meaningful to a developer reading a failed transaction in a block explorer.

Arithmetic overflow is a concern with Uint<128> at large vault sizes. Every multiplication of
two large values (specifically `shares * total_assets` and `deposit * effective_supply`) must
fit within Uint<128>'s range of approximately 3.4 × 10^38. For practical vault sizes — amounts
below 10^18 base units and supplies below 10^15 — the maximum intermediate product is
approximately 10^33, well within range. The contract enforces an implicit safety constraint
through `assert(shares_out > 0, ...)` which catches the zero-result edge case caused by
precision loss at extreme price ratios, and through `assert(amount_out > 0, ...)` in future
implementation of similar guards on the redemption path. Explicit overflow guard assertions
may be added at every arithmetic site during implementation if the compiler's type system does
not catch overflow statically.


## 23. Design Decision 15 — Observable State Without Events

**Decision: Midnight does not currently support contract events. The vault's operational
transparency is provided entirely through the public ledger state. Off-chain components that
need to track vault history poll the Midnight indexer API to observe ledger state changes.**

Because every meaningful change to the vault's state is reflected in its public ledger fields —
ticket entries are created and updated in the `tickets` Map, counters change when capital moves,
the manager key changes on authority transfer — an off-chain indexer can reconstruct the
complete history of vault operations by polling the Midnight node's indexer API at regular
intervals and diffing successive ledger snapshots. The `total_tickets` counter provides a
monotonically increasing checkpoint that allows an indexer to detect when new tickets have been
created since the last poll. Ticket keys are derivable from indices without querying the Map
directly. The frontend reads current vault state (share price, float utilization, pending
tickets) directly from the public ledger through the Midnight JS SDK.

This design does not rely on any event emission mechanism. When Midnight adds event support in
a future network upgrade, structured event emission can be added to every circuit as an
enhancement without changing any of the core accounting logic.


## 24. Design Decision 16 — Initialization

**Decision: `initialize_vault` is callable once by any caller. The initializer's
`own_public_key()` is stored as the initial `manager_key`. Only one deployment of this contract exists per contract
address, and a deployed contract can be initialized only once due to the `is_initialized`
guard.**

```compact
export circuit initialize_vault(
  max_float_bps_param: Uint<16>
): [] {
  assert(!is_initialized, "vault already initialized");
  assert(max_float_bps_param <= 10_000, "float cap exceeds 100%");

  manager_key          = disclose(own_public_key());
  pending_manager_key  = none<Bytes<32>>();
  max_float_bps        = disclose(max_float_bps_param);
  is_initialized       = disclose(true);
  first_deposit_processed = disclose(false);

  // All Counter fields initialize to zero by default.
  // All Map fields are empty by default.
  // All Boolean fields default to false.
  // No further constructor initialization is required.
}
```

The caller who invokes `initialize_vault` becomes the initial manager. In production deployment,
this call is made by the protocol's deployment script, which means the deployer's wallet key
becomes the initial manager. The manager can subsequently transfer authority to the AI agent's
operational key using the two-step manager transfer described in Section 19.


## 25. Design Decision 17 — Emergency Shutdown (Deferred)

**Decision: emergency shutdown is not implemented in this version.**

Emergency shutdown requires: a second authority account (or a shutdown flag controlled by a
governance mechanism), a `is_shutdown: Boolean` field in the ledger, assertions at the
beginning of every state-mutating circuit that check this flag, and careful specification of
which circuits remain callable during shutdown (at minimum, `manager_deposit` and
`process_withdraw` must remain open to allow capital return and user exits). The design of the
shutdown condition trigger, the authority who can invoke it, and the behavior of the withdrawal
queue during shutdown requires a dedicated security analysis. This analysis belongs in the
second iteration after the base vault has been deployed, tested, and proven correct.


## 26. Complete Circuit Catalogue

### 26.1 Exported Circuits (Callable from TypeScript DApp)

| Circuit | Caller | Description |
|---|---|---|
| `initialize_vault(max_float_bps)` | Anyone (once) | Deploys vault state, sets caller as manager |
| `authorize_deposit(user_key, amount, nonce)` | Manager only | Creates on-chain deposit authorization |
| `claim_deposit(amount, nonce)` | Authorized user | Claims authorization, mints shielded shares |
| `request_withdraw(shares_to_withdraw)` | Share holder | Burns shares, creates withdrawal ticket |
| `cancel_withdraw(ticket_key, cancel_nonce)` | Ticket owner only | Cancels ticket, remints shares |
| `process_withdraw(ticket_key)` | Manager only | Fills ticket, decrements liquid balance |
| `manager_withdraw(amount)` | Manager only | Deploys capital, increments float_outstanding |
| `manager_deposit(amount)` | Manager only | Returns capital, decrements float_outstanding |
| `update_manager(new_manager_key)` | Manager only | Nominates a new manager (Step 1) |
| `accept_manager()` | Nominated key | Accepts authority (Step 2), clears pending |
| `set_float_cap(new_max_float_bps)` | Manager only | Updates the float cap parameter |

### 26.2 Non-Exported Helper Circuits (Internal Use Only)

| Circuit | Returns | Description |
|---|---|---|
| `share_domain()` | `Bytes<32>` | Domain separator derived from contract address |
| `total_assets()` | `Uint<128>` | `total_liquid + float_outstanding` |
| `effective_supply()` | `Uint<128>` | `total_share_supply + shares_pending_withdrawal` |
| `assert_manager()` | `[]` | Asserts `own_public_key() == manager_key` |
| `mint_shares_to_user(amount, nonce)` | `ShieldedCoinInfo` | Mints shielded share coin to caller's ZSwap key |
| `remint_shares_to_user(amount, nonce)` | `ShieldedCoinInfo` | Remints shares on cancellation |
| `burn_shares(amount)` | `ShieldedSendResult` | Sends user's share coin to burn address |
| `process_first_deposit(amount, decimals, nonce)` | `ShieldedCoinInfo` | 1:1 mint with dead share burn |
| `share_nonce(base_nonce, index)` | `Bytes<32>` | Deterministic nonce derivation for share coins |
| `float_cap_check(new_float)` | `[]` | Asserts new_float ≤ max_float |

### 26.3 Witnesses (Implemented in TypeScript)

| Witness | Returns | Provides |
|---|---|---|
| `depositorZswapKey()` | `ZswapCoinPublicKey` | User's ZSwap public key for minted coin recipient |
| `userShareCoin()` | `QualifiedShieldedCoinInfo` | User's committed shielded share coin for burning |
| `withdrawNonce()` | `Bytes<32>` | User-chosen nonce for the withdrawal burn transaction |


## 27. Math Reference — All Share Calculations

This section documents every formula used in the contract, with the types of all operands,
the precision of intermediate values, and the rounding direction at every division site.

**Effective supply (used in all price calculations):**
```
effective_supply = (total_share_supply as Uint<128>) + (shares_pending_withdrawal as Uint<128>)
```
No division. No rounding.

**Total assets (used in all price calculations):**
```
total_assets = (total_liquid as Uint<128>) + (float_outstanding as Uint<128>)
```
No division. No rounding.

**Shares minted on normal deposit:**
```
shares_out = floor(deposit_amount * effective_supply / total_assets)
```
Operand types: `Uint<128> * Uint<128> / Uint<128>`. Rounds down. Depositor receives
`floor(result)`. The fractional amount that is not minted stays in the vault as dust,
imperceptibly raising the share price.

**Shares minted on first deposit:**
```
total_minted  = deposit_amount          // 1:1, no division
dead_shares   = MINIMUM_FIRST_DEPOSIT * 10^decimals
user_shares   = deposit_amount - dead_shares
```
No division. No rounding.

**Payout on withdrawal (processing time):**
```
amount_out = floor(ticket.shares * total_assets() / effective_supply())
```
Operand types: `Uint<128> * Uint<128> / Uint<128>`. Rounds down. Withdrawer receives
`floor(result)`. The fractional amount stays in the vault as dust.

**Float cap:**
```
max_float = floor(total_assets * max_float_bps / 10_000)
```
Operand types: `Uint<128> * Uint<128> / Uint<128>`. Rounds down. Manager receives a
conservatively lower deployment ceiling than the theoretical maximum. This is the safe direction.

**Invariant verification — price neutrality on ticket creation:**

Before ticket creation:
- `total_assets = T`, `effective_supply = S`, `price = T / S`

After ticket creation of `k` shares:
- `total_share_supply` decreases by `k`
- `shares_pending_withdrawal` increases by `k`
- `effective_supply = (total_share_supply - k) + (shares_pending_withdrawal + k) = S` (unchanged)
- `total_assets = T` (unchanged, no assets have moved)
- `price = T / S` (unchanged) ✓

After processing the ticket (payout = `floor(k * T / S)`):
- `shares_pending_withdrawal` decreases by `k`
- `effective_supply = S - k`
- `total_liquid` decreases by `floor(k * T / S)`
- `total_assets = T - floor(k * T / S)`
- `price = (T - floor(k*T/S)) / (S - k)` ≈ `T/S` (unchanged, only dust-level rounding difference) ✓


## 28. Decision Summary Table

| Decision | Choice |
|---|---|
| Share token model | Shielded UTXO coins via `mintShieldedToken` |
| Share balance visibility | Private (never in public ledger) |
| Underlying asset accounting | `total_liquid` Counter (off-chain coordination for MVP) |
| Deposit authorization | Two-step: manager `authorize_deposit`, user `claim_deposit` |
| First depositor defense | 1:1 mint, burn `MINIMUM_FIRST_DEPOSIT` dead shares |
| Share price timing | Processing time (live prices at ticket fill) |
| Share handling on request | Burn to `shieldedBurnAddress()` |
| Price distortion prevention | `shares_pending_withdrawal` Counter in `effective_supply()` |
| Ticket queue structure | `Map<Bytes<32>, WithdrawTicket>` keyed by `persistentHash(index)` |
| Queue ordering | No FIFO; any pending ticket processable by manager at any time |
| Partial fills | Not supported; all or nothing per ticket |
| Float cap under stress | Block new manager withdrawals only |
| Permissionless capital return | `manager_deposit` has no caller gate |
| Rounding direction | Always down (toward vault), documented at every division site |
| Spam prevention | DUST transaction cost only; no program-level share threshold |
| Manager transfer | Two-step: `update_manager` nomination + `accept_manager` proof |
| Cancellation | Implemented; ticket owner only; remints new shielded share coin |
| float_outstanding decrement | Clamped at current value via `min(amount, float_outstanding)` |
| Error handling | `assert(condition, "message")` at every guard; no silent failures |
| Events and logging | Not implemented; not supported by Midnight; indexer polls ledger |
| Initialization | Permissionless; one-time; initializer's `own_public_key()` is manager |
| Emergency shutdown | Deferred to future iteration |
| Contract-to-contract calls | Not used; not supported; off-chain coordination for MVP |
| FungibleToken module | Not used; shielded token primitives used directly |
