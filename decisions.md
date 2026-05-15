# GenV — Pre-build sync (round 2, after reading your specs)

---

## 🔴 Critical: does Lace actually track contract-minted shielded coins?

Your Frontend Reference §6.1 says `claim_deposit` returns `ShieldedCoinInfo` and "the SDK stores this automatically in the wallet's private state. The frontend does not need to handle the return value directly." And §8 says I can read user balance via `wallet.getShieldedBalance(CONTRACT_ADDRESS)`.

A forum thread from April ([link](https://forum.midnight.network/t/shielded-kernel-operations-receiveshielded-mintshieldedtoken-fail-with-proof-server-400/1137) and related discussions) suggested Lace does NOT auto-track contract-minted shielded coins from `mintShieldedToken` — that the DApp has to persist them to IndexedDB itself. This would mean I'd need to build a manual persistence layer.

**Which is true?** If Lace tracks them, my UI is much simpler. If not, I have meaningful additional work. I can verify empirically by running the bboard tutorial on Preview and checking if balance persists across browser reloads — happy to do that on Day 1. But if you've already confirmed it one way or the other, that'd save me the check.

---

## 🔴 Critical: deposit flow ordering for the demo

Your Manager Reference §5.1 + §7 (ordering rules) requires: real USDC arrives at the manager's Polygon wallet → confirm → call `authorize_deposit`. Hard rule: never authorize without receiving funds first.

For the hackathon hosted demo (judges visit a URL on Preview), I can't realistically require judges to send real USDC to a manager wallet. My plan was to run the agent with a `DEPOSIT_MODE=demo` flag in the hosted environment that skips the Polygon-side check — the user clicks "Deposit $50 (demo)" in the UI, the UI POSTs the intent to the agent, the agent calls `authorize_deposit` immediately.

For the recorded demo and any production code path, we'd keep your real ordering.

**Are you OK with the demo-mode shortcut for the hosted version?** Same code path, env-gated. Without it, the hosted version doesn't work for judges who can't fund a Polygon wallet on the spot.

---

## 🟡 Polymarket collateral — USDC.e or pUSD?

Your Manager Reference §2 uses USDC.e (`0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`).

Polymarket [migrated collateral to pUSD in April 2026](https://help.polymarket.com/en/articles/14762452-polymarket-exchange-upgrade-april-28-2026). pUSD is 1:1-backed by USDC but a separate ERC-20 with a different Exchange contract.

If the vault holds USDC.e, the agent has to swap USDC.e ⇄ pUSD around every Polymarket trade. Friction.

**My recommendation:** use pUSD throughout. Vault accepts pUSD deposits, holds pUSD, sends pUSD payouts. One less moving part. OK with that?

---

## 🟡 First deposit minimum — 1000 USDC is a lot for the demo

§9 of the contract design: `MINIMUM_FIRST_DEPOSIT = 1_000` (base units), scaled by 6 decimals = 1000 USDC. That's a meaningful amount for a hackathon demo (1000 pUSD is real money on Polygon mainnet, and Preview faucet may not dispense that much in one go).

**My plan:** I generate the manager seed, deploy the vault, and pre-seed it with a 1000+ USDC first deposit during the init script — so the demo never hits the cold-start branch. The recorded demo storyboard would just show normal deposits from depositor 2 and 3, not the cold start.

**Alternative:** drop `MINIMUM_FIRST_DEPOSIT` to something like 10 or 100 for the hackathon. ~1 line change.

**Which do you prefer?**

---

## 🟢 Confirmation items (likely just yes)

1. **Is `cancel_withdraw` still in your scope?** You included it in the Frontend Reference, but we'd planned to cut it from the demo. If you've already started it, we keep it (just won't demo it). If not, we cut it. Either is fine for me — the frontend reference makes it trivially easy to add a UI button if it exists.

2. **Proof generation benchmarks** — can you run each of the demo's circuits on the recording machine and report median + p99 by hour 8 of the hackathon? Our 2-min storyboard packs 7 proofs; if any one exceeds 15s p99, we have to pre-generate some of them rather than run them live.

3. **Network: Preview as default?** Mainnet is not required (Jay confirmed in Discord). I'm waiting on Jay's follow-up about whether preprod's shielded-op bug is fixed. Until that resolves, defaulting to Preview (faucet works, shielded ops apparently work per the bboard tutorial).

4. **You deploy the contract; I generate the manager seed and run `initialize_vault`** from a script. You don't need to touch the seed. After init, the deployer wallet IS the manager — no separate `update_manager` ceremony needed. OK?

---

## What I'm building (no action needed, just FYI)

- **UI**: React DApp scaffolded from your Frontend Reference. Wallet via Lace. Calls `claim_deposit` / `request_withdraw` / (optional `cancel_withdraw`). Polls indexer every 12s for `Ledger` state. Has a "Public Ledger View" tab that renders the full on-chain state to make the privacy claim demonstrable on video. Deployed to Vercel.
- **Agent**: Node.js process using `WalletBuilder.buildFromSeed` with the manager seed. Implements your four loops (deposit / withdrawal processing / capital deployment / capital return) from Manager Reference §5. Also runs an HTTP endpoint `POST /authorize` for deposit intents from the UI. Polymarket via `@polymarket/clob-client-v2`. LLM decisions via Claude. Deployed to Railway.

Both target the same network (Preview, pending). Both runnable locally for dev iteration.

---

## TL;DR — replies I most need from you

| #   | Item                                                 | Need from you                 |
| --- | ---------------------------------------------------- | ----------------------------- |
| 1   | Does Lace track contract-minted shielded coins?      | Yes / no / "test it on Day 1" |
| 2   | Demo-mode deposit shortcut (no real USDC for hosted) | ✅ / ❌                       |
| 3   | pUSD vs USDC.e                                       | Pick one                      |
| 4   | First deposit: pre-seed or lower MINIMUM             | Pick one                      |
| 5   | Cancellation in scope?                               | ✅ / ❌                       |
| 6   | Proof benchmarks by hour 8                           | ✅ / ❌                       |
| 7   | Preview as default network                           | ✅ / ❌                       |
| 8   | You deploy, I init                                   | ✅ / ❌                       |

Thanks for shipping the specs in advance — saved both of us a lot of round-trips.
