# GenV — Pre-build sync (for Yemi's review)

I'd like you to **approve, push back, or flag what won't work** — ideally before either of us writes serious code, so we don't compound mismatches over 48 hours.

Each section is short. Just react with ✅ / ❌ / "need to talk."

---

## 1. The demo storyboard — confirm this is what we're building toward

The 2-minute video has four beats. Everything else is out of scope.

| Time | Beat | Contract calls |
|---|---|---|
| 0:00–0:30 | Two users deposit (different amounts). Public Ledger View panel shows only aggregate numbers — no per-user data. | `authorize_deposit` × 2, `claim_deposit` × 2 |
| 0:30–1:00 | Agent decides on a 5-minute Polymarket BTC market, places a real ~$1 trade. `float_outstanding` rises on Midnight, no on-chain link to the Polygon side. | `manager_withdraw` |
| 1:00–1:30 | Market resolves via Chainlink in seconds. Profit returns. Share price ticks up. | `manager_deposit` |
| 1:30–2:00 | User withdraws, receives more than they deposited. | `request_withdraw`, `process_withdraw` |

**❓ Sound right? Anything you'd cut or add?**

---

## 2. Contract scope — these are the 7 circuits we actually need

In-scope (must work end-to-end for the demo):
- `initialize_vault`
- `authorize_deposit`
- `claim_deposit` (including first-deposit branch + dead-share burn)
- `manager_withdraw`
- `manager_deposit`
- `request_withdraw`
- `process_withdraw`

Plus the supporting helpers in your design doc (`share_domain`, `total_assets`, `effective_supply`, `assert_manager`, `mint_shares_to_user`, `burn_shares`).

Out of scope for the 48-hour build???:
- `cancel_withdraw`
- `update_manager` / `accept_manager`
- `set_float_cap` (set once at init, never changed)
- Float-cap-stress logic from §15 of your design doc
- Multi-ticket queue handling beyond one ticket at a time

**❓ OK to cut the above? Are these 7 enough on your side? Have you already started implementing any of the cut circuits — if yes, sunk cost rules apply and we'll keep them, just won't demo them.**

---

## 3. Required design tweak — drop plaintext fields from `DepositAuthorization`

In §4.1 of your design doc, the struct is:

```compact
struct DepositAuthorization {
  user_key:   Bytes<32>;
  amount:     Uint<128>;
  nonce:      Bytes<32>;
  is_claimed: Boolean;
}
```

I'd like to ask you to **drop `user_key`, `amount`, `nonce` from the struct and keep only `is_claimed`.**

Why: the `auth_key` is already `persistentHash([user_key, amount, nonce])`, so all three are cryptographically committed into the key. Storing them again in the struct's value means they end up plaintext in the public ledger — which contradicts the privacy claim in your own §3 ("Individual deposit amount → Private → Never disclosed in circuit execution").

Why it matters for the demo: I'm building a "Public Ledger View" tab in the UI that shows the chain state live during beat 1. If the struct keeps plaintext, the on-screen view would show per-user deposit amounts and the privacy claim collapses. With only `is_claimed`, the view shows opaque hashes — which is exactly the story we want.

~10 minute change in the struct definition + the `authorize_deposit` and `claim_deposit` circuits (delete the redundant assertions).

**❓ Can you confirm you'll make this change?**

---

## 4. `claim_deposit` return value — clarification

Does `claim_deposit` return `ShieldedCoinInfo` directly to the caller, so my UI can grab it and write it to IndexedDB? Or is there a separate `receiveShielded` ceremony after the circuit call?

This affects my deposit UI shape — Lace doesn't auto-track contract-minted shielded coins, so I have to persist them myself in the browser. The forum threads I read suggest the contract has to either `sendImmediateShielded` it or persist it itself.

**❓ What's the exact return shape you're planning for `claim_deposit`? Same question for `cancel_withdraw` (though that's out of scope) and `process_withdraw`.**

---

## 5. Toolchain pinning

I'm planning to pin on the UI/agent side:
- `compactc` 0.31.0
- `@midnight-ntwrk/compact-runtime` ^0.16.0

These are the versions our research found compatible. Misalignment here apparently breaks everything in subtle ways.

**❓ What versions are you using? If different, let's pick one set together.**

---

## 6. Network — Preview, pending Jay's follow-up

Jay (Midnight team) confirmed mainnet is not required. He recommended **preprod**, but I asked a follow-up about the April thread reporting `mintShieldedToken` / `receiveShielded` / `sendShielded` failing on preprod with proof-server HTTP 400 ([forum link](https://forum.midnight.network/t/shielded-kernel-operations-receiveshielded-mintshieldedtoken-fail-with-proof-server-400/1137)).

Plan:
- **If preprod bug confirmed fixed** → deploy to preprod.
- **If still broken** → deploy to Preview (faucet available, working shielded ops per the bboard tutorial).

We'll know once Jay replies.

**❓ Any objection to Preview as the default until that resolves?**

---

## 7. Manager seed + who deploys what

Proposal:
- **You** deploy the contract bytecode to the network and give me the contract address.
- **I** generate the manager seed (32 bytes, lives in my `.env` + Railway secrets), then call `initialize_vault(max_float_bps=8000)` from a script.

This way you don't have to handle the seed (and don't accidentally commit it). The agent (which signs everything) and the seed live in the same place.

For the hosted version, the vault will be pre-seeded with one cycle of activity (first deposit + dead shares processed) so judges who visit land on a "warm" vault and experience the normal deposit path, not the cold-start branch with its 1:1 mint quirks. For the recorded demo, I'll use a separate fresh deployment so the cold-start narrative plays cleanly.

**❓ OK with this split?**

---

## 8. Proof generation benchmarks — the highest-risk unknown

The 2-minute video packs **7 ZK proofs** (2× `claim_deposit`, 1× `manager_withdraw`, 1× `manager_deposit`, 1× `request_withdraw`, 1× `process_withdraw`, plus 2× `authorize_deposit` from the agent during beat 1).

If proof generation is slow (15+ seconds per proof), the demo doesn't fit in 120 seconds and we have to redesign — probably by pre-generating proofs in advance and submitting them at the right beat.

I need you to run each in-scope circuit on the recording machine and report??:
- Median proof generation time per circuit
- p99 (worst-case observed)

By **hour 8 of the hackathon**, ideally. This gates whether we proceed with the storyboard as-is or restructure.

**❓ Can you commit to having benchmarks by hour 8?**

---

## 9. What I'm building (FYI, no action needed)

- **UI**: React DApp scaffolded from the bboard template, deployed to Vercel. Connects to Lace, calls your circuits via the generated TS client, persists shielded coins to IndexedDB, has a "Public Ledger View" tab.
- **Agent**: Node.js process on Railway. Wears three hats — (1) signs all manager circuits, (2) Polymarket trader loop, (3) HTTP endpoint at `POST /authorize` that turns user deposit intents into `authorize_deposit` calls.
- Both target the same network. Both also runnable locally against Docker for dev iteration.
- Anything that depends on the contract shape, I'll mock against your generated TS interface until your version is ready.

---

## 10. Things to flag back to me

If you disagree with anything above, push back. Particularly:
- Anything you've already started that's outside the in-scope list
- Anything in the in-scope list you don't think you can deliver in 48h
- The `DepositAuthorization` privacy fix (§3) — this is the one I most need you to commit to or refuse
- Different opinions on network choice (§6), toolchain pinning (§5), or who-deploys-what (§7)

---

**TL;DR for fast scan:**

| # | Item | Need from you |
|---|---|---|
| 1 | Storyboard | ✅ / ❌ |
| 2 | 7 in-scope circuits, cut the rest | ✅ / ❌, + "have I already started cut work?" |
| 3 | Drop plaintext from `DepositAuthorization` | ✅ / ❌ — this is the load-bearing one |
| 4 | `claim_deposit` return shape | Clarify |
| 5 | compactc 0.31.0 + runtime 0.16.x | Match versions or propose alternatives |
| 6 | Preview by default | ✅ / ❌ |
| 7 | I generate seed + init; you deploy contract | ✅ / ❌ |
| 8 | Proof benchmarks by hour 8 | ✅ / ❌ |

Reply on any of these and we sync from there. Thanks 🙏
