# CLAUDE.md — Governance Engine

Context for whoever (human or Claude Code) rebuilds the governance module in this repo. This file specifies *what* to build and *why*, not a step-by-step implementation script — use judgment on file layout, but keep the contracts below.

## Why this module exists

Arenium's governance is built on one hard rule and everything else configurable. The hard rule: **protection of members' fundamental rights is never optional, in any community, ever.** Everything else — voting method, approval threshold, quorum, fine escalation — is a per-community choice, stored as data, not branched in code.

If you find yourself writing `if (community.type === 'condominio') { ... }` anywhere in this module, stop. That's exactly the pattern this architecture exists to avoid. A community's behavior comes entirely from its ruleset, not from special-casing the kind of community it is.

## Core data shape: the ruleset

A ruleset is the full, explicit configuration for how one community governs itself. It is **versioned and append-only** — a community never edits its ruleset in place. Changing a rule creates a new version; old versions stay in the table forever, because a proposal must always be evaluated against whatever ruleset was active *when that proposal opened*, not whatever is active now.

```ts
// src/lib/governance/ruleset.ts
import { z } from 'zod';

export const RulesetSchema = z.object({
  votingMethod: z.enum(['one_member_one_vote', 'proportional_to_contribution']),
  approval: z.object({
    threshold: z.enum(['simple_majority', 'two_thirds', 'unanimity']),
    quorumPercent: z.number().min(0).max(100),
  }),
  cancellation: z.object({
    threshold: z.enum(['simple_majority', 'two_thirds', 'unanimity']),
  }),
  fineEscalation: z.object({
    noticeDays: z.number().int().positive(),
    accrualPercent: z.number().min(0),
    legalCollectionAfterDays: z.number().int().positive(),
    governedExclusionAfterDays: z.number().int().positive(),
  }),
});

export type Ruleset = z.infer<typeof RulesetSchema>;

export const DEFAULT_RULESET: Ruleset = {
  votingMethod: 'one_member_one_vote',
  approval: { threshold: 'simple_majority', quorumPercent: 50 },
  cancellation: { threshold: 'simple_majority' },
  fineEscalation: { noticeDays: 7, accrualPercent: 2, legalCollectionAfterDays: 30, governedExclusionAfterDays: 90 },
};
```

Table: `community_rulesets` — append-only. Each row is `{ id, community_id, version, ruleset (jsonb), created_at, created_by }`. No `UPDATE` statements against this table, ever — only `INSERT`. "Editing a rule" in the UI means: validate the new object against `RulesetSchema`, insert it as `version + 1`.

**Don't hardcode a solution for things the team hasn't decided yet** — tie-breaking mechanism (voto de minerva), the Specialist role, and unequal vote weighting are all still open design questions. Leave room in the schema for them (e.g. an optional field, or a documented TODO), but don't invent a default that looks authoritative.

## The engine: pure functions, no side effects

```ts
// src/lib/governance/engine.ts
function computeProposalStatus(proposal: Proposal, votes: Vote[], ruleset: Ruleset): ProposalStatus
function computeCancellationStatus(proposal: Proposal, votes: Vote[], ruleset: Ruleset): ProposalStatus
```

Contract:
- **Pure.** Same inputs → same output, every time. No DB calls, no `Date.now()`, no I/O inside these functions — pass in whatever time-dependent values they need as arguments.
- **The ruleset passed in is whatever version was active when the proposal opened** — the caller resolves this by joining on `community_rulesets` at the correct version, not the engine.
- Output is a status enum (`pending`, `approved`, `rejected`, `expired` — whatever the actual states are) plus enough detail to explain *why* (vote counts, quorum met/not met) for the UI and for audit.

DB operations (fetching the community, resolving the right ruleset version, writing the resulting status) live in a separate layer that *calls* the engine — never inside it. If you're tempted to add a DB query inside `computeProposalStatus`, that's a sign the function is doing too much.

## The bug that already happened once — don't reintroduce it

A previous implementation computed "simple majority" as `votesFor >= totalVotes * 0.5`. That's wrong — it approves exact ties, which is not a majority. It must be strictly greater than: `votesFor > totalVotes * 0.5`. This was caught in code review, not by a test, which is exactly backwards. **Write the boundary-condition test first**: exactly 50/50 must resolve to *not approved* under `simple_majority`. Cover the same boundary for `two_thirds` (exactly 2/3 either way) and `unanimity` (one dissenting vote).

## Testing

Unit-test the engine with `bun:test`, testing pure input/output — no DB, no mocks needed since the functions don't touch I/O. At minimum, cover:
- Each `threshold` type at, above, and below its boundary.
- Quorum not met (even with unanimous support among those who did vote).
- Zero votes cast.
- A proposal evaluated against an *old* ruleset version after the community has since changed its rules — confirm it uses the version that was active at proposal-open time, not the current one.

## Explicitly out of scope for this module

Don't pull these in — they're separate concerns with their own modules:
- Nostr event signing (governance events aren't signed yet — this module works on plain DB rows for now)
- Chat-first UI
- Payment execution / Pix
- KYC / identity verification

Keep this module's surface area to: ruleset schema + validation, versioned storage, and the two pure decision functions.
