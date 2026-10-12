# AGENTS.md — Split-Pay Build Contract

This file governs how any AI coding agent (any model, any session) works on this repo. It does not describe *what* to build — that's `split-pay-ssot.md`. It does not describe the shared names/interfaces between agents — that's `SEAM.md`. This file describes *how to think and act* while building, so behavior stays consistent regardless of which model is working or what part of the codebase it's touching.

**Five agents work this repo: Web3, Backend, Frontend-Entry, Frontend-Core, Frontend-Shell.** Frontend is split three ways along user-journey boundaries (see master SSOT Section 12) — never further split, since smaller slices add coordination overhead that outweighs the time saved. Each agent owns its own folder (see SEAM.md Section 1 for exact paths — no shared bucket). Anything that crosses between agents is named in SEAM.md and is not up for reinterpretation. Anything inside one agent's own files is that agent's judgment, governed by the rules below.

**Standard applies equally regardless of task size.** Changing a button's color and writing the payment contract are held to the identical bar of rigor and rule-following in this file. Task size is never a reason to relax verification discipline.

If SSOT and this file ever conflict, SSOT wins on product scope; this file wins on process and correctness discipline; SEAM.md wins on any naming/interface question.

---

## 1. Non-Negotiable Invariants

These rules apply to every feature, every file, every session. Violating one is a build-blocking error, not a style note.

- **This is a money system.** Every payment-path function must be idempotent, race-condition-safe, and must fail loudly (revert/error), never silently.
- **The smart contract is the source of truth for money.** Cap enforcement, contribution totals, and refund eligibility are decided onchain. Supabase/Envio mirror that state for fast reads — they never independently authorize a payment, cap check, or refund.
- **No crypto-facing UI leaks.** No wallet popups, gas prompts, seed phrases, "connect wallet" buttons, or chain-selector dropdowns are ever shown to a buyer or organizer. If an implementation path would surface one, stop and flag it — don't ship it and note it as a known issue.
- **Double-submit protection is already guaranteed by the chain** (transaction sequencing prevents a resend from processing twice) — no custom idempotency system is built. The Frontend agent's only job is disabling the action button on tap until a result returns (SEAM.md Section 10) — a UI courtesy, not a safety mechanism.
- **Every capped contribution check must assume simultaneous requests.** Never write "check then act" logic that isn't atomic. Prefer `require()`-style atomic checks over "read balance, then update" patterns.
- **Refunds are never self-triggered by the contract** (blockchains have no clock). The contract exposes a permissionless `refund()` per SEAM.md Section 2. The backend agent owns a script that calls it every 1–2 minutes for expired, unfunded pots. This is the only refund trigger — no third-party automation dependency.
- **No admin/owner role, no role-based access control, anywhere in the app.** The platform treasury (SEAM.md Section 2 `TREASURY_ADDRESS`, Section 8) is a plain, pre-set address outside the account/permission system entirely — never build a privileged role to manage it. This is a deliberate decision, not an oversight — flag, don't build, if a task seems to call for one.

## 2. Verification Discipline

- **Don't self-report success. Prove it.** A task isn't "done" because the agent says so — it's done when a deterministic check (a test that runs and passes, a transaction that confirms onchain, a build that compiles) demonstrates it.
- **Test the edge case, not just the happy path**, for every money-related function: exact-cap-hit, double-submit of the same idempotency key, two simultaneous contributions that together exceed the cap, timer boundary (exactly at deadline), zero-contributor refund.
- **When touching the smart contract, write the test before or alongside the code**, not after. Contract bugs are the most expensive kind here (see Section 4 of the SSOT — code is final once deployed).
- **If a library, SDK, or sponsor integration behaves unexpectedly, verify against its actual current docs before assuming your usage is wrong.** Training data may be stale for fast-moving tools like Privy and Envio.
- **When a verification check fails:** stop. Do not paper over it, ship a workaround, or proceed on the assumption that a partial fix solved it. Isolate exactly what failed, fix that specific thing, then re-run the *same* check that failed — not a weaker or different one. If genuinely stuck after a real attempt, flag it plainly rather than pushing forward with something fragile (per Section 4, ambiguity handling) — naming the risk is always correct; silently shipping uncertainty is never acceptable.

## 3. Build Order Priority

**Phase 0 (Web3 agent alone):** contract core logic (create, contribute with cap enforcement + built-in payout, refund) + tests, deployed to Monad testnet. Nobody else starts wiring real data until a deployed address + ABI exist in `/abi`. No mock-data layer — real contract from day one for everyone else.

**Phase 0.5 (sponsor protocol setup + isolated verification — standard procedure, not a special exception):** every sponsor protocol gets set up and verified in isolation *before* any feature code is built on top of it.
- **Privy:** confirm Monad testnet gas sponsorship actually works via a trivial test transaction — verify zero gas prompt reaches the user, not just "a wallet got created" — before Frontend-Entry builds the login flow on it.
- **Envio:** once the contract is deployed, point the indexer at it and confirm a real test `contribute()` call shows up in a query, before Backend builds dashboard queries on top of it.
- **Alchemy:** no special verification needed — it's one RPC URL every agent references in their own `.env` (SEAM.md Section 9).

**Phase 1 (parallel, once Phase 0 and 0.5 land):**
- Web3 agent: hardening, edge-case tests.
- Backend agent: Supabase schema, Envio-backed queries, the refund-trigger script, the Privy→profile handoff (SEAM.md Section 5).
- Frontend-Entry, Frontend-Core, Frontend-Shell agents: build their named scope in parallel (see each agent's own SSOT) against real data, using the shared states/vocabulary and locked layout in SEAM.md and the master SSOT Section 10.
- Whichever Frontend agent finishes its own scope first picks up any remaining Frontend-Shell-scope work not yet started — a fixed, pre-decided assignment, not a choice made at build time.

**Phase 1.5 (utility templates):** see "Utility Templates" section below.

**Phase 2 (independent, non-blocking track):** visual polish — the real `<LoadingReceipt>` torn-receipt animation, favicon, `<HelpSearch>` content, final design pass. Built separately, drops into the already-named placeholder components (SEAM.md Section 7) without anyone else's code changing.

**Phase 3:** real user test (5+ friends), demo prep, video, README.

If asked to build something out of this order, flag it rather than silently complying — the person may have a reason, but the risk should be named.

## 3A. Utility Templates

Before the main build starts, a set of genuinely generic, reusable pieces are built in isolation, outside any single agent's scope, specifically to save time once the five agents begin. "Generic" means: contains no Split-Pay-specific logic and doesn't depend on anything not yet finalized (so no contract-specific or Envio-query-specific code here).

**In scope for pre-building:** a Supabase realtime subscription helper; a retry/backoff wrapper for network calls (the "taking longer than expected" pattern); the "disable button on tap until result returns" hook; a QR code generator (string in, QR out); the shared toast/banner shells for Loading/Success/Failure (structure only, no Split-Pay copy baked in); a Hardhat test-scaffold setup.

**Explicitly out of scope for pre-building:** the smart contract itself; anything touching real Envio query shapes (schema is locked, but the actual queries aren't proven until written against a real deployed contract).

**The rule for using any pre-built template is standard procedure, stated plainly, not a special warning:** before using a template, the agent runs a check to confirm it actually works as intended, same as it would for any dependency or library it didn't write itself, then proceeds. This is not optional and not a one-time caveat — it's simply how every agent treats code it didn't just write, every time.

## 4. Ambiguity Handling

- If the SSOT doesn't specify something the current task needs, don't invent product scope. Flag the gap and propose the smallest reasonable default, consistent with what's already locked (e.g. "SSOT doesn't say what happens on a zero-contributor group pay expiring — I'll treat it as a no-op close, confirm?").
- Never silently expand scope (e.g. adding a feature "while I was in there") even if it seems helpful. Section 4 of the SSOT lists what's deliberately excluded — respect it.
- Never silently narrow scope either. If something in the SSOT looks hard to implement as specified, say so and propose an alternative — don't quietly simplify it.

## 5. Handoffs

- Every unit of work ends in a real artifact (code, a passing test, a committed file) — not a description of what was done.
- Leave a short note (commit message or comment) explaining *why* a non-obvious decision was made, especially around money logic, so the next session (human or agent) doesn't have to reverse-engineer intent.
- If a change touches both contract and frontend, make that explicit rather than letting it look like a frontend-only change.

## 5A. Build Logs

Each agent (Web3, Backend, Frontend-Entry, Frontend-Core, Frontend-Shell) creates and maintains its own `buildlog-<agent>.md`. Every agent may **read** any build log, but may only **edit** its own — this follows the same no-shared-bucket principle as code ownership (SEAM.md Section 1), applied to logs.

Each entry should capture: major decisions made and a one-line why, files completed, files in progress, files not yet started, and any open question flagged for the person. This exists so project status is checkable in seconds without reading every commit or asking the agent directly.

## 6. What "Done" Means for This Project

A feature is done when:
- It has a deterministic test/verification step that passes.
- Its edge cases (per Section 2) are covered, not just the happy path.
- It matches the SSOT's stated scope — no more, no less.
- It doesn't violate any invariant in Section 1.

If any of these isn't true, the feature is not done, regardless of whether the demo "looks like it works."
