# pi-bar TLDR backtest — v3 (post-0.3.28, true)

**Important:** the v2 report measured 0.3.27, not 0.3.28. The patcher's commit
synced `backtest/tldr-logic.ts` with the v1 changes only; v2 helpers
(`stripDanglingPrepositions`, branched `checkpointSystemPrompt`) never made
it into the vendored copy. Re-synced manually, then re-ran. This report
reflects the real 0.3.28 behavior.

## Wins from 0.3.28 (R2 + R3)

### R2 — Final-priority past-tense — ✅ fully fixed
Every `final` TLDR across the 3 sessions now uses past tense:
- `Removed outdated image references from README`
- `Added new screenshots to README and removed old asset`
- `Published package with latest updates`
- `Updated footer implementation with extension status support`
- `Bumped package for publication`
- `Implemented status filtering commands for extension`
- `Implemented interactive status visibility picker`
- `Identified five small-surface fixes for backtest logic`
- `Updated backtest deliverables with findings and traces`

0/15 finals were past-tense before v2 → 15/15 after. Big win.

### R3 — Dangling preposition — ✅ fixed
- `Bumping version to for publishing` → now `Bumped package for publication`
- `Publishing pi-bar version` → now `Published updated package`
- No dangling `to`/`version` at end of any TLDR observed.

## Residual issues

### R1 — Banned verbs still slip (Verifying / Validating / Confirming)
Expanded ban list (`Verifying`, `Validating`, `Checking`, `Confirming`,
`Searching`, `Finding`, `Capturing`) doesn't fully hold. Live samples:

- `Verifying image references in README content`
- `Verifying repository status after commit` (×N)
- `Verifying TypeScript compiler availability`
- `Verifying package and repository status`
- `Validating TLDR logic against past transcripts`
- `Validating session message structure and content types`
- `Confirming package for publication`

Hypothesis: ban list is buried in a long instruction wall. Model treats it
as soft. Both `Verifying ✓` and `Reviewing ✓` are reasonable English; with
no positive reason to pick one over the other, it drifts.

**Fix candidates:**
- Move the ban list to a **structured** section near the END of the prompt
  (recency bias on instruction-following models). e.g.:
  ```
  HARD CONSTRAINTS (most important, do not violate):
  - The first word of your output must be one of: Reviewing, Investigating,
    Exploring, Updating, Refining, Fixing, Implementing, Wrapping up,
    Bumping, Releasing, Preparing, Updated, Investigated, Refined,
    Wrapped up, Released, Bumped, Implemented, Fixed.
  - Never start with: Verifying, Validating, Checking, Confirming, Reading,
    Reading, Editing, Writing, Running, Publishing, Capturing, Grep,
    Listing, Counting, Extracting, Displaying, Searching, Finding.
  ```
- Or: sanitizer-side fallback — if first word is in the ban list, rewrite
  to a generic neutral verb (`Reviewing`/`Reviewed`).
  ```ts
  const VERB_REPLACEMENTS = {
    Verifying: "Reviewing", Validating: "Reviewing", Checking: "Reviewing",
    Confirming: "Reviewing", Reading: "Reviewing", ...
  };
  ```

### R5 — "Continuing with task progression" still emitted verbatim
The immediate-priority prompt explicitly lists this string as a bad example,
but the model produced it verbatim once:
- `Continuing with task progression`

Other immediate-priority filler still seen:
- `Exploring user input responses`
- `Engaging with user input responses`
- `Reviewing user inquiries about recent activities`

All on second/third user messages where the model has no context yet about
the new request. (Backtest's `before_agent_start` event captures only the
new prompt text; no tool/result has happened.)

**Fix candidates:**
- Bad examples are normally interpreted as "things to AVOID writing".
  Having `Continuing with task progression.` in the bad list may
  paradoxically prime the model. Consider removing that specific example
  and instead saying:
  ```
  If the user's request is opaque (e.g. "continue", "go", "ok"), summarize
  the carry-over task using a noun, not a generic verb. Example: a "continue"
  reply during a refactor should yield "Resuming refactor work", not
  "Continuing with task progression".
  ```

### New (cosmetic) — file-path strip leaves orphaned preposition
Strip removes the path target but leaves the function preposition:
- `Reviewing for image update instructions` (was "Reviewing README for ...")
- `Reviewing for implementation insights` (was "Reviewing X.ts for ...")

`stripDanglingPrepositions` only catches `to/at/as/of/by/...` at the END.
Mid-sentence dangling `for/in/on/with` after a stripped object is not
caught.

**Fix candidate:** after `stripIdentifierLeaks`, if a stripped span was
followed by `for|in|on|with|after` and preceded by `Reviewing|Investigating
|Updating`, drop the orphan preposition too:
```ts
// "Reviewing for X" -> "Reviewing X"
.replace(/\b((?:Reviewing|Investigating|Updating|Refining|Exploring))\s+(for|in|on|with|after)\s+/gi, "$1 ")
```

### New (cosmetic) — repeated content with light variation
Render-side near-dup check (`isNearDuplicateTldr`) catches verbatim
collisions, but ~3-way repeats with synonym variation still slip:
- `Updating footer implementation in extension code` (×4 consecutive)
- `Updating status footer configuration logic` (×2)
- `Running live backtest with session data and limit parameter` (×3)

Render UX cost: minimal (each varies enough to register as fresh). API
cost: ~4× per burst. R4 from v2 (generation-side burst dedupe) still open.

## Score card

| Issue | Pre-0.3.27 | 0.3.27 | 0.3.28 |
|-------|-----------|--------|--------|
| "with success" suffix | 84 | 0 | 0 |
| File-path leak | 56 | 0 | 0 |
| Backticks | 12 | 0 | 0 |
| Premature tool_call fire | 6 | 0 | 0 |
| Final tense correct | 0/15 | 0/15 | 15/15 |
| Verb-leak first-word | 78 | 9 | 11 |
| Dangling prep | 4 | 4 | 0 |
| Aborted-final fabrication | yes | no | no |
| User-msg checkpoint carryover | yes | no | no |

Net: 0.3.28 closes R2 and R3 cleanly. R1 needs harder prompt structure or
sanitizer-side verb rewrite. R5 is mostly fixed; cosmetic one-shot slip.

## Files

- Traces: `backtest/out/*.live.md` (overwritten with 0.3.28 outputs)
- Vendored logic (post-resync): `backtest/tldr-logic.ts`
- Harness: `backtest/run.ts`
