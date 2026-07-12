# Group F: Campaign Transparency + Numeric Input — Design

**Execution mode:** Inline execution in this session (no subagent dispatch), matching Group E1.

## Goal

Three small, related UX additions to the campaign phase: reveal every party's policy stance + campaign budget once all human players have submitted, show the previous term's national stats and seat count for comparison from term 2 onward, and pair every policy/budget slider with a linked numeric input.

## 1. Reveal policy + budget after everyone submits

**Trigger:** `claimedSlots().every(s=>s.submitted)` — the same condition `maybeRunElection()` already checks before running the election, so this reveal and the election trigger fire at the same moment (the reveal is purely additive UI, not a new state transition).

**Content:** A new table in `PHASE_TEMPLATES.campaign`, rendered only when the trigger condition is true, listing all 6 `slots`: party name, the three policy axes (`economy`/`reform`/`security`), and the 5-region budget allocation. All of this data already exists on each `slot` — no new room fields.

**Placement:** Below the player's own policy/budget editing UI, so a player still sees their own controls until the moment they've all submitted (this table can only appear once nobody can change their own submission anyway).

## 2. Prior-term comparison card (term ≥ 2)

Confirmed by reading `startNewTerm()`: it does **not** reset `r.national`, `r.treasury`, or `r.lastElection` — those fields still hold the previous term's final values when the room enters the new `campaign` phase. No new room fields needed.

**Content:** A card shown only when `r.term>1`, with:
- `r.national.economy`/`.trust`/`.unrest` and `r.treasury` (last term's final figures)
- A small table from `r.lastElection.results` (party name, seats won last election)

## 3. Numeric input paired with every slider

Every `<input type="range">` in the campaign phase (3 policy axes + 5 region budget steppers) gets a sibling `<input type="number">` bound to the same underlying value, two-way:
- Dragging the slider updates the number.
- Typing in the number updates the slider (clamped to the same range: `-100..100` for policy axes; `0..remaining` for budget, same as the existing `adjustDraftBudget` clamp behavior).

No new mutation functions — this only changes the `oninput` wiring already present on these inputs (`ui.draft.policy.economy=Number(this.value)` etc.), adding a matching numeric sibling that calls the same setter.

## Explicitly out of scope

- No changes to the election calculation, room state shape, or any phase other than `campaign`.
- No historical record beyond the immediately preceding term (no multi-term history log).

## Self-review

- **Placeholder scan:** none found.
- **Scope check:** single phase (`campaign`), no data-model changes — appropriately small for one plan, matching the "small, quick" framing this Group was chosen for.
- **Ambiguity check:** "prior-term comparison" scoped explicitly to the immediately preceding term only (not full history), per design above — this was implicit in the brainstorm and made explicit here to avoid an implementer over-building a history log.
