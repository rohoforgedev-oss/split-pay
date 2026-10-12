# Split-Pay — Web3 Agent SSOT

You are the **Web3 agent**. You own `/contracts` and `/abi`. Read `AGENTS.md` (shared rules for all agents) and `SEAM.md` (locked interface — your contract must match it exactly) before writing anything. This document gives you the full picture plus your specific scope.

---

## The App, in Brief (context, not your job to build)

Split-Pay lets Nigerian friend groups pool money for cinema outings and restaurant bills without anyone fronting cash. A creator sets a spending cap and deadline; friends join via a link, pick items, and pay their share; the pot pays out automatically the instant the cap is hit, or refunds everyone automatically if the deadline passes unfunded. Target: Monad Metropolis Hackathon, Track 02, deadline Oct 13, 2026. Full product detail: `split-pay-ssot.md` (master).

**You go first, alone.** Backend and Frontend wait on your deployed contract + ABI before wiring real data — there is no mock-data layer for anyone to fall back on. Prioritize shipping a correct, tested, deployed draft fast over a perfect one slowly.

## Your Scope

Build, test, and deploy the Split-Pay smart contract to Monad testnet, per the exact interface in `SEAM.md` Section 2. Do not invent function names, event names, or constants — they're fixed. Everything else about the contract's internal implementation is your judgment.

### What the contract must do (plain description — SEAM.md has the exact signatures)

1. **Create a group pay:** store a cap, a deadline, and a designated receiver address (could be a company wallet or the creator's own wallet — generalized, not hardcoded).
2. **Accept contributions:** anyone, including the creator, can pay in at any time before the deadline, via the same function. Reject any attempt after the deadline. Enforce the cap atomically — if two contributions arrive that together would exceed the cap, the first succeeds and the second reverts cleanly. Payment token is **native MON**, so this is a `payable` function using `msg.value`, not an ERC-20 transfer.
3. **Auto-payout on cap hit:** the moment a contribution causes the running total to reach the cap, split the pot **98.5% to the designated receiver, 1.5% to `TREASURY_ADDRESS`** (the platform fee), both transfers **in that same transaction**. No external trigger, no second call needed. Fee never applies to refunds — refunds always return exactly what was contributed, untouched.
4. **Public refund function:** callable by *anyone*, with no authorization check, once the deadline has passed on a pot that's still unfunded. Loops through all contributors and returns each their exact amount. This will be called by a backend-owned script every 1–2 minutes — not by Chainlink, not by any sponsor automation (dropped entirely, see master SSOT Section 8).
5. **Read function:** expose a way to fetch a pot's full current state in one call (cap, total so far, deadline, receiver, status, contributor list) — Backend and Frontend both need this.
6. **Emit the four events** listed in SEAM.md Section 2 at the right moments — this is what Envio indexes, so get the field names and timing right. Note `PotCompleted` carries both `receiverAmount` and `feeAmount` separately, not a combined total — this keeps the platform fee independently auditable on-chain, which is part of the project's pitch (transparent revenue, not hidden).
7. **`TREASURY_ADDRESS` and `PLATFORM_FEE_BPS` are fixed constants**, set once at deploy time — not configurable per-pot, not settable by any caller. There is no function anywhere in this contract that changes who the treasury is or what the fee is — that's intentional; don't add one even if it seems like a reasonable admin convenience.

### Non-Negotiable Invariants (from AGENTS.md, restated for your context)

- Cap enforcement must be atomic — never "check then act" in a way that isn't safe against simultaneous calls.
- The contract is the sole source of truth for money. Nothing in Backend or Frontend can override or bypass what you enforce here.
- Every contributor cap is **20 per pot, maximum** — enforce this in `createGroupPay` or `contribute`, whichever is cleaner, so the refund loop can never run out of gas.
- Timer range: 1–24 hours, enforced in the contract (not just the frontend) — a direct contract call with a 0-second deadline must still be rejected.
- No idempotency system needed — the chain's own transaction sequencing already prevents double-processing. Don't build anything extra for this.
- No admin/owner role, no privileged function of any kind — the treasury address and fee percentage are fixed constants, period.

### Testing Requirements (per AGENTS.md Section 2 — write these before or alongside the code)

- Exact-cap-hit: a contribution that lands exactly on the cap triggers the payout split correctly (98.5%/1.5%, both transfers succeed, amounts are exact with no rounding loss beyond what basis-points math allows).
- Fee never applies to a refund path — confirm a refunded contributor gets back their full original amount, not 98.5% of it.
- Race condition: two simultaneous contributions that together exceed the cap — one succeeds, one reverts, no partial state.
- Timer boundary: a contribution at exactly the deadline, one second before, one second after.
- Refund: a pot with multiple contributors, expired, unfunded — confirm every contributor gets back their exact amount, not an equal split.
- Refund with 20 contributors (the max) — confirm it doesn't run out of gas.
- Zero-contributor edge case: what happens if a pot is created and never contributed to, then expires? (Should be a safe no-op, not a revert that locks anything.)
- Deadline enforcement: a contribution attempt after the deadline is rejected, not silently accepted.

### Deliverable / Handoff

- Deployed contract address on Monad testnet.
- The ABI file, written to `/abi`, as the one artifact Backend and Frontend both depend on.
- A short note (commit message or README in `/contracts`) on any implementation decision that isn't obvious from the code — per AGENTS.md Section 5.

### Explicitly Not Your Job

- No gas sponsorship configuration (that's a Privy dashboard / Backend-adjacent task).
- No frontend error messaging (Frontend's job, though your revert reasons should be clear enough for them to map to friendly messages).
- No Chainlink integration of any kind — dropped from the project entirely.
