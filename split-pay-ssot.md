# Split-Pay — Master Single Source of Truth

**Rewritten:** 29 September 2026 (supersedes the 22 September version in full)
**Status:** Locked for build
**Event:** Monad Metropolis Hackathon — Track 02 (Consumer Products & Payments)
**Submission deadline:** October 13, 2026, 11:59 PM ET
**Builder:** RohoForge (solo), building via five parallel AI coding agents: Web3, Backend, Frontend-Entry, Frontend-Core, Frontend-Shell (frontend is split three ways — see Section 12)

**Companion documents:** `AGENTS.md` (how every agent thinks/acts — shared, all three), `SEAM.md` (every name/interface that crosses between agents — fixed, not up for reinterpretation).

---

## 1. One-Sentence Pitch

Split-Pay is a fast, bank-like app, built for Nigeria's patchy networks, that lets friend groups pool money for cinema outings and restaurant bills without anyone fronting cash or chasing repayments — group payments settle automatically the moment the pot is full, with automatic refunds if it isn't.

## 2. Named User Segment

Nigerian friend groups who go to the cinema or eat out together and currently split the bill manually via cash or bank transfer, with one person always fronting the money and chasing others to pay back.

## 3. Core Product Loop (current, step by step)

1. **Sign up / log in** — email only, Privy creates an embedded wallet silently. Zero crypto-facing UI.
2. **Home** — a few partner company tiles (mock cinema/restaurant, each with their own payout wallet and catalog), favoriting available, or "Start a custom group" with a manual receiver field.
3. **Create a group pay** — pick category, set a cap (hard limit — see Section 4 for why), set a timer (default 6h, range 1–24h), pick a receiver (a company wallet or the creator's own wallet), get a shareable link.
4. **Share the link** — no app install needed. Anyone who opens it can log in and see the pot live.
5. **Contribute, including the creator** — the creator pays in the same way as anyone else, via the same action, at any point before the deadline. No special one-time step at creation. Each contributor picks catalog items (cinema tickets/snacks, restaurant menu), their contribution = sum of items picked, capped at whatever headroom remains.
6. **Live status** — running total vs. cap, who's paid, what's outstanding, real-time (Supabase real-time subscriptions — no manual refresh ever needed).

**Solo payment** needs no separate feature or UI — it's a natural side effect of the same mechanic: a creator who contributes the full cap alone, without sharing the link, completes the pot exactly the same way a group would.
7. **Completion** — the instant a contribution causes the cap to be hit, the contract releases the full pot to the designated receiver in that same transaction. No external trigger needed for payout.
8. **Refund path** — if the deadline passes with the cap unmet, a backend-owned script (checking every 1–2 minutes) calls the contract's public refund function, returning each contributor's exact amount. No manual action, no external dependency.
9. **Receiver confirmation** — before the final "Pay" tap, the contributor sees an explicit line stating exactly who they're sending money to ("You're sending X RC to [RohoWatch / RohoEats / creator's display name]"), requiring deliberate confirmation. Not optional, not skippable.
10. **Receipts + QR (scope-cut 8 Oct 2026 — timeline risk).** Each contributor's personal QR is still **generated** and shown as proof of attendance, logged to their own history — cheap, low-risk. The camera-based **scan-side** confirmation flow (receiver scans a QR to confirm) is **cut entirely**: real device/browser camera-API risk for a step the contract doesn't depend on, since payout already fires automatically on cap-hit regardless of any scan. If time allows after the core is done, this is the first thing to add back, not before.
11. **Dashboard** — three tabs: Active, Successful, Unsuccessful, each pulled from on-chain status. Company tiles show visit frequency (not spend). Favorites available.
12. **Help** — a searchable static feature-list page (title + short explanation per feature, live-filtered by search), replacing an interactive tutorial overlay entirely (see Section 9B for why).
13. **Account controls** — sign out, and delete account (with an explicit notice of what can and can't be removed — see Section 5A). Terms of Service acceptance is required at sign-up (see Section 5C) and linked here for reference.

## 4. Why the Cap Is a Hard Limit, Not a Design Shortcut

A cinema outing or dinner is an atomic event — either the full group goes, or the plan doesn't happen. There's no meaningful "75% funded" state for a restaurant booking. A flexible/partial-release system would recreate the exact ambiguity (who's owed what, who trusts who) that the app exists to remove. Strict completion is the correct design for this use case, not a limitation to apologize for — this reasoning belongs in the pitch.

## 5. Explicitly Out of Scope (for the hackathon build)

- **Fiat on-ramp.** Users are pre-funded via testnet faucet. The README states this is a deliberate prioritization of the payment mechanic over infrastructure that's assumed/expected, and names the real path (a licensed on-ramp — see Section 5B) for a production version.
- **Real bank payout / off-ramp.** Funds stay within the contract and the app's internal display (RohoCredits); nothing sends to a real bank account.
- **Personal/group savings (ajo/esusu-style) and lending.** Different regulatory category and different product — deliberately excluded.
- **Real cinema/restaurant vendor integration.** Catalogs are mocked.
- **Chainlink CRE.** Dropped entirely (see Section 8). Refunds rely solely on the backend's own script — reliable, needs no external approval, runs today.
- **Custom-group trust enforcement.** The contract guarantees *who* gets paid and *when*, not what the receiver does with funds afterward — same trust model as handing a friend cash. Company pots carry no such risk (funds go directly to the venue's own wallet, never through the creator). Named plainly in the README, with a theoretical solution (Section 5D), not hidden.
- **Interactive tutorial overlay.** Replaced by a searchable static feature-list page (Section 9B) — simpler to build correctly and arguably better for a judge wanting a fast specific answer.
- **Admin/owner role or role-based access control.** Deliberately not built, anywhere. The platform treasury (Section 9A) is a plain address outside the account/permission system entirely — avoids a real security surface for no actual benefit. See Section 9A.

## 5A. Account Deletion — Exact Disclosure

Delete account must show, before confirming: *"Your email and profile will be removed. Your past group pay history stays on the blockchain permanently and can't be deleted — but it isn't tied to your name or email once your profile is gone."* No surprises, no soft-pedaling.

## 5B. Real-Money Path (stated honestly in the README, not built)

A production version would route fiat→crypto conversion through a licensed on-ramp (so Split-Pay never custodies fiat itself and doesn't need its own money-transmitter license) — the user leaves to a licensed widget, converts there, and only the finished stablecoin/token returns to the contract. This is the same honest architecture used for the parked Galaxy-Pay/DevoFund ideas (see separate briefing file) and should be named explicitly as the fast-follow, not glossed over.

## 5C. Terms of Service — Content and Gating

**Content:** what data is stored and why (email, wallet address, group pay history); explicit testnet/no-real-money disclosure; the account deletion notice (Section 5A, verbatim); the 1.5% platform fee disclosed plainly (Section 9A); the custom-group trust limitation (Section 5, bullet above); a basic no-warranty statement.

**Gating (not optional):** ToS agreement is an unavoidable checkbox at sign-up — no account is created without it. Frontend-Shell owns the content page; Frontend-Entry owns gating sign-up behind it.

## 5D. Theoretical Solutions for Named Limitations (README "Path to Production" section)

Every limitation in Section 5 gets a real, specific theoretical fix, not a vague "future work" note — this demonstrates production-path thinking, not just awareness of gaps:
- **Fiat on-ramp / real-money path:** a licensed on-ramp integration (Section 5B) — already the most detailed of these.
- **Custom-group trust gap:** an optional milestone/manual-confirmation release layer (same shape as the parked DevoFund idea — see separate briefing file), named as a real roadmap item, not built now.
- **Real vendor integration:** a venue onboarding flow once real partnerships exist — the company-receiver architecture (Section 11A) is already built to support this, only the partnerships themselves are missing.

Each limitation's blocker is named honestly (licensing, capital, or partnerships — not effort), since these are constraints, not unfinished work.

## 6. Money Flow (Critical Logic)

- Money enters the contract **as each person pays**, not only at completion — the pot is a running balance from the first contribution.
- **Payout** is built into the same transaction as the contribution that hits the cap — no external trigger.
- **Refund** requires an external nudge (the blockchain has no clock) — owned by the backend's script, not any sponsor protocol.
- **Double-submit protection** is guaranteed by the blockchain's own transaction sequencing — no custom idempotency system is built. The frontend disables the action button on tap as a UI courtesy only (see AGENTS.md / SEAM.md Section 10).
- **Race-condition safety:** cap enforcement is an atomic check inside the contract itself — two simultaneous contributions that together exceed the cap cannot both succeed; the second reverts and the frontend shows a friendly message, not a raw error.
- **Source of truth:** the smart contract decides all money movement. Supabase and Envio read/mirror; they never independently authorize anything.
- **Contributor cap:** 20 per pot, to keep the refund loop small enough to always succeed within gas limits.
- **Deadline enforcement:** `contribute()` rejects any call after the deadline — enforced in the contract, not only the frontend.

## 7. Tech Stack

| Layer | Choice |
|---|---|
| Frontend | SvelteKit + Tailwind CSS |
| Backend/DB | Supabase (Postgres, auth pairing, real-time subscriptions, RLS) — app data only, not a copy of on-chain data |
| Wallets | Privy (embedded wallets, buyer + creator/company side) |
| Smart contract | Solidity, Hardhat |
| Chain client | viem |
| Indexing | Envio — queried directly for on-chain facts; not synced into Supabase |
| RPC | Alchemy |
| Payment token | **Native MON** (not a stablecoin — one transaction, no second contract dependency, no bounty reason left to use a stablecoin since Agora is dropped) |
| Network | Monad testnet |
| Refund trigger | Backend's own script, checking every 1–2 minutes — no Chainlink |
| Display currency | **RohoCredits (RC)** — not ₦, not a raw crypto symbol; explained in the tutorial/README as standing in for real currency |

## 8. Sponsor Bounty Stack (Current)

| Sponsor | Bounty | Amount | How Split-Pay qualifies |
|---|---|---|---|
| Track 02 | Consumer Products & Payments | $10,000 (1 of 3) | Primary submission track |
| Privy | "Privy!" | $5,000 | Embedded wallets (buyer + company/creator side), gas sponsorship keeping the UI crypto-free |
| Envio | Best Use of Envio | $1,000 | Powers the Active/Successful/Unsuccessful dashboard and company visit-frequency stats — genuine use, not forced |
| Alchemy | Best Projects using Alchemy | $1,000 in credits (not cash) | RPC provider |

**Cash total if everything lands:** Track 02 $10K + Privy $5K + Envio $1K = **$16K**. Alchemy is credits only.

**Dropped:** Chainlink CRE ($3,000) — Early Access approval was an unresolved external dependency against a hard deadline; the backend's own script already does the real job (reliable refund triggering) without it. Agora, Aurora Intents, Nansen — all dropped earlier as forced fits (see prior reasoning, unchanged).

**Not pursued:** Kuru, Perpl, Dynamic, Mera, MetaMask Agent Wallet Plugin, Kimi/Kepler/Alibaba credits, Monad Community Team Project (solo build) — all wrong category or conflicting.

## 8A. Account Auto-Funding

Every new account is automatically funded with **500 RC** (RohoCredits, the display name for testnet MON) the instant their Privy wallet is created — no faucet interaction needed by the user. Mechanically: the builder's own treasury wallet is pre-stocked from Monad's public testnet faucet ahead of launch; a backend task sends 500 RC from that treasury to each new wallet on signup. This is only possible because it's test currency with no real value — a real-money version would require a licensed fiat on-ramp (Section 5B), which is explicitly out of scope here. README states this plainly, as something deliberately built, not skipped.

No single catalog item (cinema ticket, snack, menu item) is priced above **200 RC**, keeping the whole demo cheap and small-scale.

If a user ever needs more than their starting 500 RC, they can request additional testnet MON themselves directly from Monad's public faucet — noted once in the README and once in the Account section of the app, not required for normal use.

## 9. Privy's Role (final)

- **Embedded wallets** for every user, including company/organizer accounts — created invisibly on login.
- **Gas sponsorship** on Monad testnet, configured in the Privy Dashboard, verified working before any UI is built on top of it.
- **No server wallet.** Originally planned to handle payouts, but the contract releases funds itself (Section 6) — a server wallet moving money would contradict the rule that the contract is the sole source of truth. Dropped, not just deprioritized.
- **Platform treasury wallet** — a third, legitimate use of the same mechanism: one Privy wallet created once, at setup, tied to the builder's own login, no new system. Its address is hardcoded into the contract at deploy time. See Section 9A.

## 9A. Platform Fee and Treasury

Every completed payout automatically sends **1.5%** to a fixed treasury address and the remainder to the designated receiver, in the same transaction, inside `contribute()`'s payout branch — never off refunds, which always return exactly what was contributed. No per-company fee split; not relevant until real partnerships exist.

**The treasury is a plain Privy wallet address, not an account type.** Public (viewable on-chain by anyone, standard for every blockchain wallet) is not the same as accessible (only the builder, via their own Privy login, can move funds) — this is framed as a pitch strength ("fully auditable revenue") in the README, not a flaw. No admin/owner role or RBAC is built anywhere in the app as a result of this (see Section 5, limitations list) — the treasury sits outside the account/permission system entirely.

**Submission metrics:** a one-off script (Backend-owned, run manually before submitting, not a live dashboard feature) pulls: total accounts (Supabase `profiles` row count), successful transactions (Envio `PotCompleted` event count), and treasury revenue (on-chain balance check) — real, timestamped numbers for the README as proof of use, not estimates.

## 9B. Help System (replaces the tutorial overlay)

`<AppTutorialOverlay>` (interactive, element-highlighting walkthrough) is dropped entirely — real engineering effort (live-element positioning, interaction-blocking, step sequencing) for comparatively low payoff. Replaced by a **searchable static feature-list page**: title + short explanation per feature, live-filtered by a search input, no live-element targeting needed. Owned by Frontend-Shell. Better fit for a judge who wants a fast, specific answer rather than a forced multi-step walkthrough.

## 10. UI/UX — Locked Vocabulary, States, and Full App Layout

See `SEAM.md` Section 6 for the exact, fixed wording every agent must use (action states, pot status badges, empty states, 404 page, deletion notice), and `SEAM.md` Section 11 for the full locked screen layout and design tokens. **Nothing about layout, arrangement, or structure is left to an agent's judgment** — every screen's arrangement is specified below and in SEAM.md; agents implement, they do not design.

### Home screen layout (locked — see generated reference mockup)
- Top bar (brand yellow): logo/wordmark left, RC balance pill + tutorial icon (fixed `?` button) right.
- Greeting line + one-line subtext.
- Two company tiles side by side (RohoWatch, RohoEats — see Section 11A), each: icon circle, name, category + visit-frequency stat (Envio-sourced), favorite star (filled if favorited, outlined if not).
- "Start a Custom Group" — full-width outlined card below the company tiles.
- "Active group pays" section: card(s) showing title, status badge, progress bar, "X / cap RC • N of M paid."
- Bottom nav, 4 items: Home, Active, History, Account.

### Account section contents (locked)
Display name/email, wallet address (view-only, copyable), RC balance, "get more RC" link (faucet, optional top-up), sign out, delete account (with Section 5A disclosure), link to Terms of Service.

### Envio's full scope (recap — one list, multiple filtered views, not separate systems)
There is **one** ongoing record of group pays. Envio watches the blockchain and keeps that single record accurate and queryable in real time as pots are created, contributed to, completed, or expired. Every Envio-powered view — the Active/Successful/Unsuccessful dashboard tabs, company visit-frequency stats, personal receipt/transaction history — is a different **filtered view** over that one record, never a separate data source.

### Post-completion link behavior (locked)
Opening a group pay's link after it has already completed or been refunded takes the user to their account, then directly into that specific group pay's permanent summary/receipt view — identical to what they'd see tapping it from inside the dashboard. It never breaks and never re-shows a "pay now" state.

### Session behavior (locked)
Magic-link email login. Session persists after first login (Privy-managed) — no repeated login prompts on return visits, until explicit sign-out.

### Receiver confirmation screen (locked — new, Frontend-Core)
Inserted between item-picking and final payment: explicit statement of who the money is going to, plus total, requiring deliberate confirmation before the transaction fires.

### QR — generation only, scanning cut (8 Oct 2026)
Personal QR generation stays, built by Frontend-Core, low-risk. The camera-based scan-to-confirm flow originally planned here is cut for timeline risk — see Section 5 entry above. If added back post-core, it remains a permission check (logged-in wallet matches the pot's `receiver`), not a new account type.

## 11. Design Direction

- **Palette (locked):** yellow + white as the base, with **dark slate/charcoal (`#1E293B`)** replacing black for every dark element (buttons, nav bar) — pure black against bright yellow reads as hazard/construction tape; warm charcoal keeps the premium, trustworthy feel without clashing. Any additional color (success green, error red, body text gray) used only where it genuinely complements the base.
- **Fonts:** heading/display — Baloo 2 (rounded, warm, free on Google Fonts); body — Inter (clean, highly legible at small sizes for numbers/receipts). Pending final confirmation once the person reviews real layout references (Section 11B).
- **Spacing / corner radius / animation timing:** no custom system invented — use Tailwind CSS's built-in default scale throughout, consistently. Animation: all transitions use one standard duration/easing (150–200ms, ease-out) everywhere, no per-screen variation.
- **Feel:** warm, rounded, fun — not corporate or sharp. Soft fills over hard outlines (cream card backgrounds, not bordered boxes); pill-shaped buttons and nav over sharp rectangles.
- **Signature loading animation:** a receipt torn diagonally in half, "Split-Pay" written across the cut — ties the brand concept (splitting) to the visual identity. Built as its own independent, non-blocking task (Phase 2), dropped into the shared `<LoadingReceipt>` placeholder component once ready. Treated as a must-have, the strongest single brand moment in the demo.
- **Help:** searchable static feature-list page, not an interactive overlay — see Section 9B.
- **404 page:** simple, branded, "this page doesn't exist" + home button.
- **Error states:** never a raw error or blank screen — see SEAM.md Section 6 for exact wording.

## 11A. Mock Companies (locked)

- **RohoWatch** — cinema. Catalog: movie tickets, snacks. No single item over 200 RC.
- **RohoEats** — restaurant. Catalog: menu items. No single item over 200 RC.

Both have their own Privy-created payout wallet (SEAM.md Section 3, `companies.payout_wallet_address`).

## 11B. Layout Reference (pending)

The person is reviewing real Tailwind layout examples (tailwindcss.com/showcase, Flowbite, DaisyUI, Mobbin) and will specify a chosen reference layout to lock in. Until then, the structural layout in Section 10 stands as the locked arrangement; visual styling (exact spacing/shadows/card treatment) follows once a reference is chosen.

## 12. Build Order (Five-Agent Parallel Structure)

**Five agents total:** Web3, Backend, Frontend-Entry, Frontend-Core, Frontend-Shell. Frontend is split three ways because it has the most individual screens — the split follows user-journey boundaries, not arbitrary slicing:
- **Frontend-Entry:** landing, sign-up/login, home, create-pot screen.
- **Frontend-Core:** the full money-critical journey kept as one cohesive piece — shared pot view, catalog/item picking, payment action, completion screen with QR. Highest stakes, never split further.
- **Frontend-Shell:** dashboard (tabs + detail), tutorial overlay, account controls, 404, Terms of Service page, wiring in the final design system/SVG once ready.

**Phase 0 — Web3 agent alone.** Contract (create, contribute-with-cap-check-and-built-in-payout, refund) + tests, deployed to Monad testnet. No mock-data layer for anyone — all four other agents wait for a real deployed address + ABI before wiring real data, per SEAM.md.

**Phase 0.5 — Sponsor tool setup + isolated verification, before feature code depends on them.** Privy: confirm Monad testnet gas sponsorship actually works with a trivial test transaction (not just "wallet got created" — confirm zero gas prompt reaches the user) before Frontend-Entry builds the login flow on top of it. Envio: once the contract is deployed, point the indexer at it and confirm a real test `contribute()` call shows up in an Envio query before Backend builds dashboard queries on top of it. This is standard procedure, not a special exception — see AGENTS.md.

**Phase 1 — parallel, once Phase 0 and 0.5 land.**
- Web3: hardening, edge-case tests.
- Backend: Supabase schema, Envio-backed queries, the refund-trigger script, the Privy→profile handoff.
- Frontend-Entry, Frontend-Core, Frontend-Shell: build their named scope in parallel against real data, using SEAM.md's locked vocabulary and this document's locked layout (Section 10).
- Whichever of the three Frontend agents finishes its named scope first picks up any remaining small pieces in Frontend-Shell's scope not yet started — a static, pre-decided assignment (Shell absorbs overflow, since the overflow items are natively Shell-shaped work), not an open choice made at build time.

**Phase 1.5 — Utility templates, built in isolation (see AGENTS.md "Utility Templates" section for the full list and the verify-before-use rule).**

**Phase 2 — independent, non-blocking.** Real `<LoadingReceipt>` torn-receipt animation (optimized file size, actually wired into the component, not left orphaned), favicon, searchable help content (Section 9B), final visual design pass once a layout reference is chosen (Section 11B). Drops into already-named placeholder components without touching anyone else's code.

**Phase 3.** Run the submission-metrics script (Section 9A), real user test (minimum 5 friends), fix friction, record demo + pitch videos, finalize README (AI-use disclosure, README joke under the logo, auto-funding note, faucet note, Path to Production section from Section 5D, metrics as proof of use), submit.

**Target:** core logic (Phases 0–1) done by ~Oct 6, leaving a genuine buffer before Oct 13.

## 13. Judging Rubric Alignment (Track 02)

| Criterion | Weight | How this build answers it |
|---|---|---|
| Founder & Market Readiness | 25% | Specific named segment; real, understood pain point; speed/network-resilience as an explicit differentiator |
| Technical Execution | 20% | Real onchain cap enforcement, race-condition-safe, automated payout and refund — not simulated |
| Design & Craft | 20% | Zero crypto-facing UI; non-crypto user completes the full flow unassisted; honest, calm error states |
| Traction & Path Forward | 20% | Real test with 5+ friends before submission; distribution story = one friend group tells another |
| Originality & Track Insight | 15% | Group-pay-as-a-message mechanic (matches Monad's own suggested direction #1), not a generic wallet app |

## 14. Submission Requirements Checklist

- [ ] Public GitHub repo, shared with metropolis@hackathon.monad.xyz
- [ ] MIT/Apache/GPL license
- [ ] README: setup instructions, architecture overview, tech stack, AI-tool-use disclosure, scoped-out-features note (Section 5), real-money path note (Section 5B)
- [ ] Commit history covering the full build window
- [ ] Technical demo video (≤3 min, live product, no slides/code walkthrough)
- [ ] Pitch video (≤2 min, team + problem + why)
- [ ] Project logo/graphic (JPG/JPEG/PNG/WEBP, ≤3MB)
- [ ] Live product link on Monad testnet, with test login credentials for judges, pre-funded demo wallet
- [ ] Evidence of real user testing (minimum 5 friends) before submission
- [ ] (Optional) 30-second product ad — not judged, skip unless finished early

## 15. Resolved Decisions Log (previously "open," now closed)

- **Payment token:** native MON. Decided — simplest path, no bounty reason to use a stablecoin.
- **Chainlink Early Access / receiver design:** moot — Chainlink dropped entirely.
- **Privy server wallet:** dropped, not kept as a fallback — contract handles payout itself.
- **Privy gas credits:** still needs a dashboard check before relying on it — the one item carried forward as a real pre-build verification step, not a design decision.
- **QR system:** one group QR (payment confirmation) + individual QR per contributor (personal record) — same for both categories, no category-specific logic.
- **Currency display:** RohoCredits (RC).
- **Supabase vs. Envio boundary:** Envio queried directly for on-chain facts; Supabase never a copy of it.

## 16. Still Genuinely Open

- Final font confirmation (Baloo 2 / Inter) pending the person's chosen layout reference (Section 11B).
- Exact layout reference selection (Section 11B) — person is reviewing tailwindcss.com/showcase, Flowbite, DaisyUI, Mobbin.

## 17A. Timeline Reality Check (8 October 2026 — deadline Oct 13, 5 days)

Build hasn't started yet; pipeline hit a real blocker (API key purchase failures). Given the compressed window:
- **QR scan-side: cut entirely** (Section 5, Section 10). Generation stays.
- **Torn-receipt `<LoadingReceipt>` animation: built in isolation**, same Phase 2 non-blocking pattern already in AGENTS.md — main pipeline never waits on it, dropped in whenever ready.
- **5-person user test: not cut, correctly sequenced as the last step** (Phase 3) — trivial once the core app actually works, so it's not treated as a timeline risk in itself.
- **Everything else in this document stands as the real target**, not just the core — the person chose to keep full scope rather than cut further, understanding this requires a near-flawless remaining build.

## 17. Resolved This Session (30 September 2026)

- **QR scan-side access:** permission check on existing `receiver` data, not a new account type — see Section 10.
- **Receiver confirmation screen:** locked into the payment flow before final "Pay" — Section 10.
- **Custom-group trust gap:** named as an explicit, acknowledged limitation with a theoretical solution (Section 5D), not something to engineer past.
- **Platform fee:** 1.5%, routes to a personal treasury wallet (plain Privy wallet, no new account type) — Section 9A.
- **No admin/owner role, no RBAC:** explicit decision, anywhere in the app — Section 5, Section 9A.
- **Solo payment:** confirmed as a side effect of the existing mechanic, no separate UI — Section 3.
- **Modular/reusable builds:** reusability comes from clean SEAM-boundary code, not a separate external library effort.
- **Terms of Service:** content list and sign-up gating locked — Section 5C.
- **Tutorial overlay dropped**, replaced by a searchable static feature list — Section 9B.
- **Dark palette element:** charcoal (`#1E293B`) replacing pure black — Section 11.
- **Submission metrics tracking:** one-off script for accounts/transactions/revenue — Section 9A.
