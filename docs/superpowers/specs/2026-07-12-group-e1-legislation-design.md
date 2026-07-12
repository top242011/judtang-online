# Group E1: Legislation System (เสนอ-แก้ไข-โหวตกฎหมาย) — Design

**Execution mode:** Inline execution in this session (no subagent-per-task dispatch). Spec and plan are kept lean accordingly.

## Goal

Add a persistent legislation subsystem to the governing phase: any party can propose a bill, other parties can compete to amend it, the chamber votes on both the amendment and final passage, and passed bills apply a continuing per-turn effect until replaced. Ties into region ideology drift and a public-debt carryover, per explicit user request during brainstorming.

## Scope decisions from brainstorming (recap)

- Full concurrent multi-bill system: up to 6 bills/turn (1 per party), amendment competition, seat-weighted votes — accepted at full scope, longer build time explicitly OK'd by user.
- **E1b (political capital unifying all decision types — events + laws + policies + civil-service orders) is deferred**, not part of this spec.
- Items from the same conversation that are **separate, standalone Groups** (not part of E1): campaign-platform reveal + prior-term comparison (lobby feature), numeric slider input (UX), and the AI-automation race-condition bug (already fixed and shipped in commit `32885d2`, along with the optional-coalition-on-majority feature).
- Item "ระบบงบประมาณแบบละเอียด + หนี้ภาครัฐ" is **partially folded into E1**: a persistent public-debt figure tied to the fiscal category, interacting with `activeLaws`. Full per-ministry budget-allocation sliders are explicitly **out of scope** here — that is the Democracy-4-style mechanic already earmarked for E2, not duplicated in E1.

## 1. Data model

**New party field** (hand-written per party in `PARTY_TEMPLATES`, alongside existing `policy`):
```js
billStance:{fiscal:-100..100, social:-100..100, security:-100..100}
```
6 values written by hand, roughly consistent with each party's existing `policy` (e.g. a party with `policy.economy` very negative gets `billStance.fiscal` very negative — populist/high-spend).

**New room fields** (added to room-creation defaults, `resetRoomToLobby()`, and `startNewTerm()` per the reset matrix in §7):
```js
pendingBills: []          // in-flight bills this turn, cleared every turn
activeLaws: { fiscal:null, social:null, security:null }   // persists across terms
publicDebt: 0             // persists across terms (only resetRoomToLobby clears it)
```

**Bill shape** (one entry in `pendingBills`):
```js
{
  id,                       // random per-bill id
  proposerId,               // slot id
  category: 'fiscal'|'social'|'security',
  direction: 1|-1,
  stage: 'amend'|'select'|'pass'|'done',
  amendments: [ {proposerId, intensity} ],  // intensity != original's
  selectedIntensity,        // set once 'select' stage resolves
  selectVotes: {},          // slotId -> intensity voted for
  passVotes: {},            // slotId -> 'yes'|'no'
}
```

Intensity is one of `'mild'|'moderate'|'aggressive'`. Amendments may only change intensity, never category or direction. Duplicate amendment intensities collapse to one candidate in the selection vote.

## 2. Magnitude table

Per category, per intensity, applied every turn while the law is active (direction multiplies the sign):

| Category | mild | moderate | aggressive |
|---|---|---|---|
| fiscal | treasury ±5, economy ±1 | treasury ±10, economy ±2 | treasury ±18, economy ±4, trust ∓2 |
| social | trust ±3, unrest ∓2 | trust ±6, unrest ∓4 | trust ±10, unrest ∓7, satisfactionAll ±3 |
| security | unrest ∓3 | unrest ∓6, trust ±2 | unrest ∓10, trust ±4, regions (party's strongest-fit region) ±5 |

(Sign convention: `direction:1` applies the effect as listed; `direction:-1` flips every sign.)

Note: the `regions ±5` entry for security/aggressive is a **popularity** effect on the law's sponsoring party (same mechanism as existing event `effect.regions`, writing into `r.popularity[proposerId][regionId]`) — distinct from the permanent `REGIONS.leaning` drift described in §4, which is a separate, slower, party-independent effect of the law simply being active.

## 3. Lifecycle (all within one governing turn, alongside the existing random event)

1. **Propose** — each party (human via UI, AI via automation) may submit at most one bill this turn. Up to 6 concurrent entries in `pendingBills`.
2. **Amend** — every party other than the proposer may submit one amendment (a different intensity, same category/direction). AI submits an amendment only when the bill's direction opposes its own `billStance` sign, picking the mildest available softer intensity.
3. **Select** (only if ≥1 amendment exists) — seat-weighted plurality across the distinct intensities in play (original + collapsed amendments); PM tie-break on exact ties. AI votes for the intensity closest to its own `billStance`.
4. **Pass** — seat-weighted yes/no on the winning intensity; >50% to pass, PM tie-break at exact 50/50. AI votes based on stance-gap between its `billStance` and the passed version's effective direction/intensity.
5. **Resolve:**
   - Pass → `activeLaws[category]` replaced with the new law (auto-replace, no separate repeal action); proposer's party gets no trust change.
   - Fail → proposer loses a small trust amount (`trust -2`); bill discarded, nothing recorded.
6. `pendingBills` is cleared at the end of every turn regardless of stage reached (a bill that didn't finish its lifecycle within the turn is simply dropped — no carryover across turns).

## 4. Ongoing effects, region drift, and public debt

- Every turn in `nextTurn()`, before drawing the next event: apply each non-null `activeLaws[category]`'s per-turn stat effect (from §2) to `treasury`/`national.economy`/`national.trust`/`national.unrest`.
- **Region drift:** while a `fiscal` or `security` law is active, each turn nudge the `leaning` axis it targets for every region by a small fixed amount (`±0.3` per turn) in the law's direction, clamped to `[-100,100]`. This is the only mechanism that permanently shifts `REGIONS.leaning` — it never resets between terms.
- **Public debt:** each turn, if `treasury` would go negative after all effects, move the shortfall into `publicDebt` instead of letting treasury go negative (`publicDebt += -treasury; treasury = 0`). Additionally, every turn, `treasury -= Math.round(publicDebt*0.02)` (interest drag), floored at 0. `publicDebt` is never reduced automatically — only a `fiscal` law with `direction:-1` (fiscal tightening) at `aggressive` intensity also pays down debt (`publicDebt = Math.max(0, publicDebt - 8)` per turn while active). `publicDebt` carries across elections and governments (only `resetRoomToLobby()` clears it) — this is what makes it "consequences for whoever governs next," per the original request.

## 5. UI

New card section in the governing-phase template, alongside the existing event card: "ร่างกฎหมายที่เสนอเทิร์นนี้" listing each `pendingBills` entry with its current stage and the action button relevant to the player's own party (propose/amend/vote), read-only status for everyone else. A small persistent panel shows active laws per category and the current public-debt figure.

## 6. AI automation

Extends `runHostAutomation()`'s governing-phase branch with sub-steps for: proposing (probabilistic per turn, category chosen from whichever of the 3 the AI's `billStance` is most extreme in and doesn't already have an active law), amending, select-voting, and pass-voting — each mirroring the existing no-confidence AI-vote pattern (small `setTimeout` delay, busy-flag guarded, reusing the race-condition fix already shipped).

## 7. Reset matrix (state-leak prevention — this class of bug has recurred 3 times across prior Groups)

| Field | Room creation | `resetRoomToLobby()` | `startNewTerm()` | Mid-term PM reshuffle |
|---|---|---|---|---|
| `pendingBills` | `[]` | `[]` | `[]` | `[]` (any in-flight bill is abandoned) |
| `activeLaws` | `{fiscal:null,social:null,security:null}` | reset to nulls | **unchanged (persists)** | **unchanged (persists)** |
| `publicDebt` | `0` | `0` | **unchanged (persists)** | **unchanged (persists)** |

A party disabled/kicked mid-process has its pending bill discarded and is excluded from seat totals in any in-flight vote for that turn (same treatment as existing disabled-party handling elsewhere).

## Explicitly out of scope for E1

- E1b: unified "political capital" resource gating events/laws/policies/civil-service orders together.
- Per-ministry budget-allocation sliders (belongs to E2).
- Campaign-platform reveal + prior-term comparison UI.
- Numeric slider input for campaign policy screen.
