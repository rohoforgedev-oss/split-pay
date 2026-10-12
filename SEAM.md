# SEAM.md — Split-Pay Shared Interface

This is the single list of everything that crosses between the five agents
(Web3, Backend, Frontend-Entry, Frontend-Core, Frontend-Shell). If it's in
this file, nobody renames it, restructures it, or invents an alternative —
it's fixed. Anything NOT in this file is each agent's own internal choice
and is not governed here.

Build order: Web3 agent builds and deploys a working contract to Monad testnet
FIRST, alone. Backend and Frontend do not start wiring real data until a
deployed address + ABI exist. No mock-data layer, no switch-over step.

---

## 1. File Ownership (no shared bucket)

| Path | Owner | Others |
|---|---|---|
| `/contracts` | Web3 agent | read-only |
| `/backend` | Backend agent | read-only |
| `/frontend/entry` | Frontend-Entry agent | read-only |
| `/frontend/core` | Frontend-Core agent | read-only |
| `/frontend/shell` | Frontend-Shell agent | read-only |
| `/abi` | Web3 agent writes, generated on compile | all others read-only |
| `/fonts` | Shared, written once during project setup | all Frontend agents read-only |
| `/icons` | Shared, written once during project setup | all Frontend agents read-only |

Frontend-Entry, Frontend-Core, and Frontend-Shell share a routing shell, an
auth/session read pattern, the design tokens in Section 11A, and the
`/fonts` + `/icons` folders — these are the seam between the three Frontend
sub-agents, same no-shared-bucket principle applied one level down. Fonts
are self-hosted `.woff2` files (not loaded from an external CDN, e.g. Google
Fonts) — this removes a network dependency on every page load, consistent
with the app's reliability pitch; state this explicitly in the README.
Icons are local SVG files from a free icon set (Lucide, Heroicons, or
Phosphor — pick one, use it consistently), not an icon font and not loaded
from a CDN. No agent hand-writes icon SVG code — files are sourced from the
chosen set and placed in `/icons`, except the one brand-specific exception
below.

If a change touches two of these, say so explicitly in the commit — don't let
it look like a single-agent change.

## 2. Smart Contract Interface (Web3 agent builds exactly this; others call it exactly this)

**Functions:**
- `createGroupPay(uint256 cap, uint256 deadlineTimestamp, address receiver) returns (uint256 potId)`
- `contribute(uint256 potId) payable` — amount is `msg.value`, no separate amount parameter
- `refund(uint256 potId)` — public, no auth, callable by anyone once the pot is expired and unfunded
- `getPot(uint256 potId) view returns (...)` — full pot state in one read

**Events (what Envio indexes):**
- `PotCreated(potId, creator, cap, deadline, receiver)`
- `ContributionMade(potId, contributor, amount, newTotal)`
- `PotCompleted(potId, receiver, receiverAmount, feeAmount)` — split into the receiver's share and the treasury fee, so each is independently auditable
- `RefundIssued(potId, contributor, amount)` — emitted once per contributor

**Status enum (exact values, exact order):** `Open`, `Completed`, `Refunded`

**Constants:**
- `MAX_CONTRIBUTORS = 20`
- `MIN_TIMER = 1 hour`
- `MAX_TIMER = 24 hours`
- `PLATFORM_FEE_BPS = 150` (1.5%, in basis points — standard fixed-point convention for percentage fees in Solidity)
- `TREASURY_ADDRESS` — a fixed constant, the builder's own Privy wallet, hardcoded at deploy time
- Payment token: **native MON** (not a stablecoin — decided for simplicity, one transaction, no second contract dependency)

**Conventions:**
- Amounts: always wei on-chain. Convert to a display value only in the frontend.
- Timestamps: always unix seconds.
- Payout logic: built into `contribute()` itself — the function checks after
  adding a payment whether the cap is now met, and if so splits the pot
  between `receiver` (98.5%) and `TREASURY_ADDRESS` (1.5%, `PLATFORM_FEE_BPS`)
  in that same transaction. No external trigger for payout. Fee never applies
  to refunds.
- Refund logic: contract never triggers itself (blockchains have no clock).
  A separate backend-owned script checks every 1–2 minutes for expired,
  unfunded pots and calls `refund()` on them. This is the only refund
  trigger — no Chainlink dependency.
- Deadline enforcement: `contribute()` must reject any call after the
  deadline has passed — enforced in the contract, not only the frontend.

## 3. Supabase Schema (Backend agent owns creation; Frontend reads/writes only these exact names)

- **`profiles`**: id, privy_user_id, email, display_name, wallet_address, created_at
- **`group_pays`**: id (= on-chain potId), category, title, catalog_snapshot, link_slug, creator_profile_id, receiver_type, company_id, created_at
- **`companies`**: id, name, logo_url, category, payout_wallet_address, catalog
- **`contributions`**: group_pay_id, contributor_profile_id, amount, items_picked, paid_at
- **`favorites`**: profile_id, company_id

**Rule:** Supabase holds only things that were never on-chain in the first
place (identity, catalog, favorites, display metadata). It is not a copy of
Envio's data.

## 4. Envio (queried directly — not synced into Supabase)

Envio indexes the four contract events above and is queried directly by the
backend agent for anything that's a blockchain fact: live pot progress,
payment history, completed/refunded status, company visit frequency.
Supabase is never used as a middle hop for this data — that would weaken
the genuine-use case for the bounty and add an unnecessary sync step.

## 5. Login → Profile Creation (the exact handoff)

Privy handles identity and wallet creation only. It does not know about
Split-Pay's data model. The flow:

1. User logs in → Privy confirms identity, returns a user ID + wallet address.
2. Frontend calls one backend-owned handler, passing that ID and address.
   The frontend never writes to `profiles` directly.
3. Backend checks: does a `profiles` row already exist for this Privy ID?
   If not, create it. This is the only place a profile is ever created.

## 6. Shared UI/UX Vocabulary (every agent uses these exact words — no synonyms)

**Action states (any button/async action):**
`Idle` → `Loading` → (if >~5s) `Still working on it...` → `Success` or `Failure`

**Failure messages:**
- Generic failure: plain language + retry button, never raw error text/codes.
- Cap-exceeded specifically: "Someone just filled the last spot" (not a
  generic error).

**Pot status badges:**
`Open`, `Completed`, `Expired — Refunding`, `Refunded`

**Empty states:**
- No active pots: "You don't have any group pays yet. Start one."
- No favorites: "Star a company to see it here."

**404 page:** "This page doesn't exist." + button home.

**Account deletion notice (exact substance, wording can be lightly adapted):**
"Your email and profile will be removed. Your past group pay history stays
on the blockchain permanently and can't be deleted — but it isn't tied to
your name or email once your profile is gone."

## 7. Shared Component Names (placeholder-now, real-later — design is a separate swappable track)

- `<LoadingReceipt>` — the loading indicator used everywhere a loading state
  appears. Day-one version: plain spinner/text. Later, swapped in-place for
  the animated torn-receipt brand animation (`/icons/receipt-loading.svg` —
  the one hand-designed icon in the project, built in isolation per AGENTS.md
  Phase 2, not sourced from the icon set). No agent builds its own inline
  spinner — everyone calls this one component.
- `<HelpSearch>` — the searchable static feature-list page (replaces an
  earlier tutorial-overlay plan entirely — not built). Owned by
  Frontend-Shell, reachable from a fixed dashboard menu item.
- `<ReceiverConfirm>` — the explicit "you're sending X RC to [name]" step
  shown before final payment. Owned by Frontend-Core, sits between item
  selection and the `contribute()` call.

## 8. Currency Display

Internally: native MON, stored in wei. Displayed to users as **RohoCredits
(RC)** — not ₦, not a crypto symbol. State plainly in the README/tutorial
that a real launch would map RC to real currency via a licensed on-ramp.

**Auto-funding (locked):** every new account receives 500 RC automatically
on signup, sent from a builder-controlled treasury wallet (pre-stocked from
Monad's public testnet faucet) the moment a user's Privy wallet is created —
owned by the Backend agent, part of the same handler described in Section 5.
No single catalog item may be priced above 200 RC.

**Platform fee (locked):** 1.5% taken off completed payouts only, never off
refunds — refunds always return exactly what was contributed. Routes to
`TREASURY_ADDRESS` (Section 2), a plain Privy wallet owned by the builder —
not a new account type, not an admin role. No RBAC or owner-role concept
exists anywhere in this app as a result — see master SSOT Section 9A.

## 9. Config Names (each agent keeps actual values in its own `.env` — never committed; these are the shared NAMES only)

`CHAIN_ID`, `ALCHEMY_RPC_URL`, `CONTRACT_ADDRESS`, `ENVIO_GRAPHQL_ENDPOINT`, `PRIVY_APP_ID`

## 10. Idempotency / Double-Tap (ownership clarified)

The blockchain itself already prevents a transaction from being processed
twice (built-in transaction sequencing). No custom idempotency system is
built. The only thing owned here, by the Frontend agent, is a UI courtesy:
disable the action button the instant it's tapped, until a result (success
or failure) comes back. This is not a safety-critical mechanism — the real
safety is already guaranteed by the chain.

## 11. QR System

- **Group QR** (one per pot): scanned once by the receiver side, confirms the
  whole payment/booking is settled. This is the only QR tied to the money/
  completion logic.
- **Individual QR** (one per contributor): self-scan or shown as personal
  proof of attendance, logged into that person's own transaction history via
  Envio. Not tied to venue/payment confirmation.
- **Scan-side: cut from scope (8 Oct 2026, timeline risk).** The
  camera-based "Scan to confirm" action is removed entirely — real
  device/browser camera-API risk for a step the contract doesn't depend on.
  Do not build it. Individual QR generation is unaffected and stays in scope.
- Same system for both cinema and restaurant — no category-specific QR logic.

## 11A. Design Tokens and Locked Layout (full detail: master SSOT Sections 10–11)

- **Colors:** yellow + white base, with dark slate/charcoal (`#1E293B`) replacing black for every dark element (buttons, nav) — never pure black. Any other color (success green, error red, body gray) only where it genuinely complements the base.
- **Fonts:** Baloo 2 (headings/display), Inter (body) — pending final confirmation once a layout reference is chosen.
- **Spacing / radius / animation:** Tailwind CSS defaults throughout, no custom scale invented. All transitions: 150–200ms, ease-out, no per-screen variation.
- **Home screen arrangement (fixed, not an agent's design choice):** top bar (logo left, RC balance pill + help `?` icon right) → greeting + subtext → two company tiles side by side (icon, name, category + visit-frequency, favorite star) → full-width "Start a Custom Group" card → "Active group pays" section (progress bar cards) → bottom nav (Home, Active, History, Account).
- **Account section (fixed contents):** display name/email, wallet address (view-only, copyable), RC balance, "get more RC" link, sign out, delete account (Section 6 disclosure text), Terms of Service link.
- **`<LoadingReceipt>`, `<HelpSearch>`, `<ReceiverConfirm>`** — see Section 7. `<LoadingReceipt>` is not complete until the real torn-receipt SVG is optimized for size and actually wired in everywhere it's used — an unused or oversized asset does not count as done.

---

Anything not listed here is the owning agent's own implementation choice,
governed only by the general rules in AGENTS.md — not by this file.
