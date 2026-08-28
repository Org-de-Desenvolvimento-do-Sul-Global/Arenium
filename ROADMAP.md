# Arenium — Technical Roadmap

Engineering roadmap for Arenium's governance and financial infrastructure. Focused on architecture, data model, and protocol decisions — not business or institutional milestones. For product context, see [README](./README.md).

Stack baseline: SvelteKit 2 / Svelte 5, TypeScript, Bun, Tailwind 4 + shadcn-svelte, Better Auth, Drizzle ORM + PostgreSQL, Vitest + Playwright, Docker Compose.

## Table of contents

1. [Core architecture decisions](#1-core-architecture-decisions)
2. [Nostr integration design](#2-nostr-integration-design)
3. [Now — Pre-Phase 1 (0–3 months)](#3-now--pre-phase-1-0-3-months)
4. [Phase 1 — First real community](#4-phase-1--first-real-community)
5. [Phase 2 — Multi-community scaling](#5-phase-2--multi-community-scaling)
6. [Phase 3 — Banco Ubuntu](#6-phase-3--banco-ubuntu)
7. [Testing & security baseline](#7-testing--security-baseline)
8. [Open technical questions](#8-open-technical-questions)

## 1. Core architecture decisions

- **Event-sourced governance state.** The signed event log is the source of truth. Community state (membership, active proposals, vote tallies, balances) is a materialized view derived by replaying events against a rules engine — never written directly.
- **Ledger-first, payments as an adapter.** Pix integration (and any future rail) sits behind a `PaymentProvider` interface. No payment-provider types or SDKs leak into governance/ledger modules.
- **Non-custodial by construction.** Arenium's database never holds a balance as a source of truth for a community's funds — only a projection of ledger events. Actual fund custody lives with the payment rail / bank partner.
- **Two separate identity planes.** Nostr keypairs (signing) and KYC (verified identity) are modeled as separate tables with a nullable link, never merged into one `users` record. A signing key can exist without KYC (pseudonymous participation in lower-stakes actions); KYC can exist without an active session key (pending verification).
- **Deterministic dispute path.** Default state transitions require no arbitration. Disputes trigger a separate resolution flow (Kleros-style, crowdsourced/game-theoretic) that produces its own signed event rather than mutating history.

## 2. Nostr integration design

This is the concrete design for "Nostr as the signing layer," not identity verification.

### 2.1 Event kinds

Define custom **parameterized replaceable / addressable events** (kind range `30000–39999`, per NIP-01/NIP-33) for governance objects that need a stable identity across edits, and regular signed events for immutable actions:

| Purpose | Kind range | Notes |
|---|---|---|
| Community definition | addressable (`3xxxx`) | `d` tag = community ID; holds ruleset reference, not full rules inline |
| Proposal | addressable (`3xxxx`) | editable while in draft, immutable once opened for voting |
| Vote | regular, immutable | one event per vote; delegation modeled as a distinct event referencing the delegate's pubkey |
| Membership addendum | regular, immutable | references community event ID + new member pubkey |
| Ledger entry (fine, due, receivable) | regular, immutable | references community + member pubkeys; amount and status only, never payment credentials |

- [ ] Register and document Arenium's custom kind numbers (avoid collision with existing NIPs — check [nostr-protocol/nips](https://github.com/nostr-protocol/nips) kind registry before finalizing)
- [ ] Write JSON Schema / Zod validators for each event's `content` and `tags` shape
- [ ] Implement event → Drizzle row mapping for the materialized view (idempotent upsert keyed by event ID)

### 2.2 Signing & key management

Non-technical, WhatsApp-like UX conflicts with raw `nsec` handling. Decision path:

- [ ] Evaluate **NIP-46 (Nostr Connect / remote signer, "bunker")** so the app never touches raw private keys directly — a remote signer service holds keys, the client requests signatures
- [ ] Evaluate in-app custodial key generation (keys encrypted at rest, tied to the Better Auth session) as a fallback for users without an external signer, with explicit non-custodial disclosure in the UI
- [ ] Support **NIP-07**-style browser extension signing for power users / auditors, even if not the primary path
- [ ] Decide on key derivation/backup UX (social recovery vs. seed phrase vs. custodial recovery) — flag as open question if unresolved by build time

### 2.3 Relay strategy

- [ ] Stand up a self-hosted relay (candidates: `strfry`, `nostr-rs-relay`) as the canonical relay for governance events — do not depend solely on public relays for availability
- [ ] Define relay auth via **NIP-42** to restrict write access to verified community members
- [ ] Decide on optional mirroring to public relays for auditability/transparency vs. keeping governance events relay-restricted
- [ ] Implement relay reconnect/backoff and event resubscription logic client-side (WebSocket layer)

### 2.4 Privacy layer

Nostr events are public by default; governance events reference identities and outcomes that may carry regulatory or privacy weight (LGPD).

- [ ] Keep KYC data entirely off the event log — Postgres only, never referenced beyond an opaque hash/commitment if needed for audit
- [ ] Evaluate **NIP-44** (versioned encrypted payloads) for any event content that must stay confidential between specific parties (e.g. dispute evidence) while still being signed and timestamped
- [ ] Define what is public-by-default (proposals, votes, tallies) vs. what requires encryption or stays purely in Postgres (KYC, payment references)

### 2.5 Verification pipeline

- [ ] Signature verification on ingest (reject malformed/invalid-signature events before they reach the rules engine)
- [ ] Ruleset evaluation as a pure function: `(currentState, event, ruleset) => newState | rejection`
- [ ] Replay/reindex capability: full community state must be reconstructable from the event log alone, for audits and for recovering from materialized-view corruption

## 3. Now — Pre-Phase 1 (0–3 months)

### Month 1 — Foundations
- [ ] Define Drizzle schema: `events`, `communities`, `members`, `proposals`, `votes`, `ledger_entries`, `kyc_records` (linked, not merged, to identity)
- [ ] Set up local relay in Docker Compose alongside `app` and `database` services
- [ ] Implement event signing/verification utility library (wraps `nostr-tools`), covered by unit tests
- [ ] Build event validators (Zod) for community, proposal, vote, membership-addendum kinds
- [ ] Scaffold WhatsApp-like chat/thread UI shell in SvelteKit (shadcn-svelte + Tailwind), no live data yet
- [ ] Implement login flow: Better Auth session + Nostr key association (custodial or NIP-46, per §2.2 decision)

### Month 2 — Prototype & rules engine
- [ ] Implement the deterministic rules engine: pure function evaluating events against a community's published ruleset
- [ ] Implement vote + delegation event handling, including quorum calculation logic
- [ ] Build materialized-view sync worker: subscribe to relay, validate, upsert into Postgres
- [ ] Implement community & member search against the materialized view (indexed queries, not relay queries)
- [ ] Add database-level protections: row-level security or equivalent scoping so one community's data isn't queryable by another without authorization
- [ ] Write integration tests: publish event → relay → sync worker → expected Postgres state
- [ ] Prototype the `PaymentProvider` interface with a mock adapter (no live Pix integration yet)

### Month 3 — Integration & on-chain research spike
- [ ] Wire proposal creation → voting → tally → membership-addendum generation end-to-end in the UI
- [ ] Finish user dashboard backed by the materialized view (live data, not static wireframes)
- [ ] Spike: smart-contract feasibility for a future on-chain settlement/anchoring layer (does not need to ship — output is a written recommendation, not code)
- [ ] Load-test the relay + sync worker path with synthetic event volume representative of a small community

## 4. Phase 1 — First real community

- [ ] Cut over from mock `PaymentProvider` to a real adapter (Pix), still isolated behind the interface boundary
- [ ] Implement the unpaid-receivable escalation ladder as a state machine: `notification → fine accrual → legal collection → governed exclusion`, each transition a signed event
- [ ] Harden relay auth (NIP-42) for production community access
- [ ] Add monitoring/alerting on relay uptime and sync-worker lag (event backlog, replay drift from canonical log)
- [ ] Run a full audit: reconstruct community state from the event log independently of the materialized view and diff against production data

## 5. Phase 2 — Multi-community scaling

- [ ] Partition materialized-view queries and relay subscriptions per community to keep query and sync performance flat as community count grows
- [ ] Implement the subsidy/cross-community support model as its own event-linked ledger relationship, not a special case bolted onto single-community ledger logic
- [ ] Evaluate horizontal scaling of the relay (or multi-relay federation) if a single self-hosted relay becomes a bottleneck
- [ ] Revisit BaaS integration behind the existing `PaymentProvider` interface — should require no changes to governance/ledger code, only a new adapter implementation

## 6. Phase 3 — Bank

- [ ] Design a credit-scoring computation as a read-only projection over historical governance/ledger events — never a mutation of the event log itself
- [ ] Prototype anonymized cross-community reputation exposure (e.g. commitment schemes or aggregate scores instead of raw event references) — treat re-identifiability as a hard constraint to engineer against, not just a policy concern
- [ ] Design custody/lending event kinds and their interaction with the existing non-custodial ledger model (this phase may require revisiting the non-custodial constraint explicitly, since lending implies some custody)
- [ ] Evaluate migrating parts of the governance/settlement layer toward on-chain anchoring, informed by the Month 3 research spike

## 7. Testing & security baseline

- [ ] Unit tests (Vitest) for: event validators, rules-engine transitions, signature verification, ledger state machine
- [ ] Integration tests for the relay → sync-worker → Postgres pipeline
- [ ] E2E tests (Playwright) for the core governance loop: create community → propose → vote → tally → addendum
- [ ] Fuzz/property-based tests against the rules engine (malformed events, out-of-order delivery, replayed/duplicate events)
- [ ] Dependency and container image scanning in CI
- [ ] Secrets management review (relay keys, Better Auth secrets, any custodial key material) — no secrets in the repo or plain env files in production
- [ ] Rate limiting and abuse protection on the relay and public API endpoints

## 8. Open technical questions

- Final key-management model: NIP-46 remote signer vs. in-app custodial keys vs. hybrid, and backup/recovery UX
- Whether governance events mirror to public relays for third-party auditability, or stay relay-restricted
- Self-hosted relay software choice (`strfry` vs. `nostr-rs-relay` vs. alternatives) and its HA/backup strategy
- Anonymization mechanism for cross-community reputation scoring (Phase 3) that survives LGPD and Central Bank credit-bureau scrutiny
- Whether/when to introduce on-chain anchoring, based on the Month 3 spike outcome

---

*Update this document directly as architecture decisions are made; don't let it drift from the actual implementation.*
