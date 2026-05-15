# GenV — Pre-build sync (round 3, after your contract push)
Below are just the things still genuinely open.

---

## 🔴 Critical: does Lace actually track shielded coins minted by `claim_deposit`?

Your Frontend Reference §6.1 says the SDK auto-stores `ShieldedCoinInfo` and §8 says `wallet.getShieldedBalance(CONTRACT_ADDRESS)` reads the balance.

A forum thread from April ([link](https://forum.midnight.network/t/shielded-kernel-operations-receiveshielded-mintshieldedtoken-fail-with-proof-server-400/1137)) suggested Lace does NOT auto-track contract-minted shielded coins. I'll verify empirically by running through your `smoke-test.ts` and then trying to read the share balance from a fresh Lace browser session — but if you've already confirmed this works in your testing, that saves me time.

**Which way?**

---

## 🔴 Critical: demo-mode deposit shortcut for the hosted version

Your Manager Reference §5.1 + §7 require real Polygon USDC transfer before `authorize_deposit`. For the hosted version judges visit, I can't realistically require them to send real USDC.

My plan: run the agent with `DEPOSIT_MODE=demo` in the hosted environment that skips the Polygon-side check. UI POSTs intent → agent immediately calls `authorize_deposit`. Recording stays on `DEPOSIT_MODE=real` so the recorded video uses your real ordering.

**OK with that as an env-gated bypass?**

---

## 🟡 First deposit minimum (1000 USDC) — hard to hit on testnet

`helpers.compact` has `const min_deposit: Uint<128> = 1000 * 1000000;` enforced in `process_first_deposit`. For the recorded demo's cold-start narrative, getting 1000 USDC of testnet stablecoin on camera is impractical.

Two options:
- **(a) Pre-seed in `deploy.ts`**: extend your deploy script with an optional flag that does an authorize+claim with 1001 USDC after `initialize_vault`. The recorded demo then starts from "warm" vault — never hits cold-start branch.
- **(b) Drop the minimum to 10 or 100 for the hackathon**: 1-line change in `helpers.compact`.

**My preference: (a)** — keeps the inflation-attack defense honest, just shifts the demo narrative to "normal deposits" rather than "first deposit." Push back if you'd rather (b).

---

## 🟡 Network: "TestNet" or "Preview"?

Your `.env.example` says `NETWORK_ID=TestNet` and points at `indexer.testnet.midnight.network/api/v1/graphql`. Our earlier discussion was about "Preview" (faucet at `faucet.preview.midnight.network`).

Are these the same thing under different names, or are you targeting a different network than the one I was assuming?

Also: the indexer URL uses `/api/v1/graphql` rather than `/api/v4/graphql`. The midnight-js packages are at v4. Compatibility-wise, is `v1` what you've confirmed working? (Asking because if I bring up the bboard tutorial as a Lace-tracking sanity check, I want to point it at the same network you're testing against.)

---

## 🟡 Polymarket collateral — USDC.e or pUSD?

Your Manager Reference §2 references USDC.e. Polymarket migrated to pUSD in April 2026 (separate ERC-20, separate Exchange contract). If the vault holds USDC.e, the agent has to swap USDC.e ⇄ pUSD around every Polymarket trade.

**My recommendation: pUSD throughout** — one less swap step. OK?

---

## 🟢 Confirmation items (quick yes/no)

1. **Proof generation benchmarks** — can you run each demo circuit through your smoke test and report median + p99 wall-clock per proof on your machine? The 2-min video packs 7 proofs; if any p99 > 15s we need to pre-generate some. By hour 8 of the hackathon if possible.

2. **Same manager seed everywhere?** Whatever seed you put in `.env` for `npm run deploy` becomes the manager. I'll use that exact seed in the agent's `.env` and Railway. You generate it (or I do — doesn't matter who, as long as we sync). Cool?

3. **`@midnight-ntwrk/compact-js` import in `deploy.ts`** — I don't see it in package.json's dependencies. Is it transitively pulled in by `@midnight-ntwrk/midnight-js-contracts`, or do we need to add it explicitly? Just want to make sure `npm install` works clean.

---

## What I'm building (FYI)

Lifting `src/utils.ts` directly for the agent's wallet + provider setup. Adding on top:
- 10s `setInterval` loop with Polymarket polling + Claude decisions + `manager_withdraw` / `manager_deposit` calls
- `POST /authorize` HTTP endpoint that calls `authorize_deposit` (env-gated demo mode)
- Chalk-formatted logs for the recording
- Railway deployment for the hosted version

UI is a separate React scaffold (bboard pattern) calling the same contract address from `deployment.json`. Polls indexer every 12s for `Ledger` state, renders a Public Ledger View tab showing `total_share_supply`, `total_liquid`, `float_outstanding`, `deposit_authorizations`, `tickets` so the privacy claim is mechanically demonstrable on video.

---

## TL;DR — replies I most need

| # | Item | Need from you |
|---|---|---|
| 1 | Lace shielded balance tracking — works in your testing? | Yes / no / "verify yourself" |
| 2 | `DEPOSIT_MODE=demo` env-gated bypass for hosted version | ✅ / ❌ |
| 3 | First deposit: pre-seed in deploy.ts (option a)? | ✅ / option (b) / something else |
| 4 | "TestNet" — is this Midnight's preview or something else? | Clarify |
| 5 | pUSD vs USDC.e | Pick |
| 6 | Proof benchmarks by hour 8 | ✅ / ❌ |
| 7 | Same manager seed everywhere | ✅ / ❌ |


