# BlockBite Smart Contract — Week 8 Status Report

**Date:** 2026-05-20  
**Program ID (Devnet):** `9UipodjT55vBd8zZmEPvcFc8dVCveV1CMzYW2zsDHceX`  
**Network:** Solana Devnet  
**Framework:** Anchor 0.32.1  

---

## Deployment Status

| Environment | Status | Explorer |
|---|---|---|
| Localnet (CI) | ✅ Live — 25/25 tests green | GitHub Actions |
| Devnet | ✅ Deployed | [Solana Explorer](https://explorer.solana.com/address/9UipodjT55vBd8zZmEPvcFc8dVCveV1CMzYW2zsDHceX?cluster=devnet) |
| Mainnet | ⏳ Pending audit | — |

---

## Feature Completion

| Feature | Description | Status |
|---|---|---|
| `create_stream` | Linear vesting with cliff + dev fee (1%) | ✅ |
| `withdraw` | Pro-rata unlock, VGPV anti-bot, MIN_CLAIM_AMOUNT dust filter | ✅ |
| `cancel` | Creator reclaims unvested tokens; recipient gets vested share | ✅ |
| `set_milestone` | Creator unlocks cliff-gated vesting after milestone event | ✅ |
| VGPV | Velocity Guard Penalty Valve: 3 strikes within 2s → BotDetected | ✅ |
| Dev Fee | 1% of stream amount transferred to protocol treasury at creation | ✅ |

---

## Test Coverage

| Suite | Count | Result |
|---|---|---|
| Rust unit tests (`calculate_unlocked`) | 13 | ✅ Pass |
| Integration tests (TypeScript/Mocha) | 25 | ✅ Pass |
| **Total** | **38** | **✅ All green** |

### Integration test breakdown

**Happy paths:**
- Stream creation with dev fee validation
- Partial withdraw (50% elapsed)
- Full withdraw (100% elapsed, after end time)
- Cancel mid-stream (creator recovers unvested, recipient gets vested)
- Milestone unlock: cliff → set_milestone → linear vest → withdraw

**Error paths / edge cases:**
- Double withdraw → NothingToWithdraw
- Withdraw by non-recipient → Unauthorized
- Cancel by non-creator → Unauthorized
- Withdraw from cancelled stream → StreamCancelled
- Double cancel → AlreadyCancelled
- Cancel after full vest → FullyVested
- Zero amount create → InvalidAmount
- Same creator/recipient → InvalidRecipient
- Invalid timestamp (end ≤ start) → InvalidTimestamp
- Invalid cliff (cliff > end) → InvalidTimestamp
- Withdraw before stream start → StreamNotStarted
- Claimable below MIN_CLAIM_AMOUNT → ClaimTooSmall
- set_milestone by non-creator → Unauthorized/ConstraintSeeds
- set_milestone already reached → MilestoneAlreadyReached
- set_milestone before cliff → CliffNotReached
- VGPV: 4th rapid withdraw triggers BotDetected

---

## Bug Fixes (Weeks 5–7)

1. **Double-counted discriminator in `space`** — `StreamAccount::LEN` already includes 8 bytes; adding +8 wasted rent
2. **Borrow checker conflict in VGPV** — snapshotted immutable fields before `&mut ctx.accounts.stream`
3. **Wrong `set_milestone` discriminator in tests** — recomputed to `sha256("global:set_milestone")[0..8]`
4. **CI: missing `.anchor` parent directory** — added `mkdir -p .anchor` before `solana-test-validator`
5. **CI: `--bind-address 0.0.0.0` panic** — gossip layer rejects unspecified IP in newer agave; removed flag
6. **CI: deployer had 0 SOL** — added `solana airdrop 100` before `anchor deploy`
7. **CI: `ANCHOR_PROVIDER_URL` undefined** — added env vars for `ts-mocha` outside `anchor test`

---

## Performance Notes

| Metric | Observation |
|---|---|
| `create_stream` cost | ~0.01 SOL rent (StreamAccount: 196 bytes + escrow token account: 165 bytes) |
| `withdraw` cost | ~0.000005 SOL (single CPI call, no account init) |
| `cancel` cost | ~0.000005–0.00001 SOL (1–2 CPI calls depending on vested amount) |
| `set_milestone` cost | ~0.000005 SOL (state write only, no token transfers) |
| Compute budget | All instructions well within 200K CU limit; no `compute_budget` instruction needed |
| PDA derivation | O(1) — 2 PDAs per stream (stream + escrow), deterministic |

**Bottleneck:** Stream creation is the most expensive due to two account initializations (stream state + escrow token account). This is a one-time cost. Subsequent operations (withdraw, cancel) are cheap.

---

## Known Limitations / Future Work

- **No `close_stream` instruction:** Closed accounts could recover rent. Currently, stream and escrow accounts persist on-chain forever. Planned for next milestone.
- **Fixed DEV_FEE_BPS:** Fee is hardcoded at 1%. A governance mechanism for adjusting fee is planned.
- **No referral tracking on-chain:** The referral system described in the FAQ is currently off-chain. On-chain referral PDA is planned.
- **Formal audit:** Scheduled before Mainnet launch (noted in SECURITY_CHECKLIST.md).

---

## Security Checklist

See [`SECURITY_CHECKLIST.md`](./SECURITY_CHECKLIST.md) for the full security audit covering:
- Signer validation ✅
- PDA seed correctness ✅  
- Integer overflow protection ✅
- Account ownership ✅
- Reentrancy (CEI pattern) ✅
- Frontrunning/MEV resistance ✅
