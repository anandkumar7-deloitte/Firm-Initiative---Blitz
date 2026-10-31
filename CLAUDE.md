# AI & Data Blitz — Panel Gap Sheet Automation

## What this project is

Deloitte USI's "Blitz" team runs recurring interview drives ("Blitzes") to recruit AI & Data
panelists. Each Blitz has a skill-gap demand (e.g. "need 40 R1 slots for Databricks"), a pool of
pre-approved SMEs (the **Master Panel** list), and a self-service Microsoft Forms survey where
SMEs declare their availability. This repo's spreadsheets automate the matching of survey
submissions against the master-approval list and the skill-gap demand, so panelists route
themselves into **Panel Details** (confirmed) or **Yet to Confirm** (needs follow-up) with no
manual triage.

There are **two workbook variants**:

| | Weekday | Weekend / In-Person |
|---|---|---|
| Format | Virtual, Mon–Fri | Single day, in-person, at a specific city |
| Availability model | Per-weekday checkbox + time windows | "Entire Duration" Yes/No + free-text custom time |
| Slot count | Computed from selected windows | Either a validated full-day constant (varies by event — **do not assume a prior event's number carries over**; re-derive from that event's own historical data) or a flat fixed number if the team has explicitly said so |
| Sample | `samples/weekday/` | `samples/weekend/` |

**Samples in this repo have had all real employee data stripped** (Survey Response / Panel
Details / Yet to Confirm rows cleared; MasterPanel is kept since approvals are routinely-refreshed
reference data, not personal availability). If you need a copy with real data for local debugging,
pull it from the actual delivered file, don't commit it here.

---

## Current classification rules (as of the last correction)

These are the rules actually agreed and intended to be implemented. If a delivered `.xlsx` in this
repo or in production doesn't match a rule below, that's a known gap — see
[Open / pending items](#open--pending-items).

### 1. One row per person — latest submission wins, **no merging**
If someone submits the survey more than once, only their **most recent submission** counts.
Everything about them — skill, days/times, designation — comes from that one submission only.

> **Correction (supersedes earlier design):** an earlier version of this engine *merged* day-level
> availability across submissions, so that if someone's latest submission dropped a day they'd
> previously offered, that day was retained from the older submission ("additive" model). **This
> was explicitly reversed.** The current, correct rule is a clean replacement: whatever the latest
> submission doesn't mention is simply not in the final record, even if an earlier submission
> offered it. Do not re-introduce the merge/retention behavior.

Older (non-latest) submissions are tagged `Superseded` and excluded from everything downstream.

### 2. Eligibility
A submission is eligible only if **both** hold:
- **Approved**: MasterPanel has a row for that email + their submitted skill (after mapping to
  the master-skill name via Skill Mapping) with `Panel Approved = Y`.
- **Architect check**: if the skill name contains "architect", anyone below Manager (Consultant,
  Senior Consultant, Analyst — see Config's designation lists) is **not eligible**, regardless of
  MasterPanel approval. This is a hard business rule, not a data quality issue.

### 3. Level (R1 / R2)
- **Non-architect skills**: Manager and above → R2, everyone else → R1.
- **Architect skills**: inverted — Manager → R1, Senior Manager / Specialist Master and above → R2.

### 4. Routing
- Latest submission + eligible → **Panel Details**.
- Latest submission + not eligible → **Yet to Confirm**, with a short reason naming what they
  *are* approved for instead (via the SkillCheck matrix — see below).
- Not the latest submission for that email → **Superseded**.

### 5. Many-to-many skill mapping
MasterPanel's skill vocabulary doesn't always match the survey's skill names 1:1. Two patterns
recur and must be supported:
- **One master skill → two survey skills** (e.g. Informatica's "MDM + CDQ" combo approval should
  count toward both the "CDQ" and "Informatica MDM" survey buckets).
- **One survey skill → two master skills** (e.g. "Azure DE" could be approved under either of two
  slightly different master-skill spellings; "CDI/CAI" similarly has two master variants).

A single `MATCH()` only finds the *first* occurrence of a repeated key, so this is **not**
achievable with a plain one-column lookup — see the Skill Mapping helper-column design below.

### 6. Weekend-specific: primary-then-secondary fallback with FCFS capacity
For the simplified weekend/in-person model (flat slot count, no R1/R2-specific gap-matching):
check the submitter's **Primary skill** first — if approved *and* that skill+level still has Ask
capacity remaining (checked first-come-first-served by submission order) — place them there.
Otherwise fall back to checking their **Secondary skill** the same way. Otherwise → Followup
(simple, no deep reason-explanation sheet needed for this variant).

**Why this isn't computed from the live Gap Sheet "Available" number**: doing so would create a
genuine circular reference (Panel Details ← routing decision ← Gap Sheet "Available" ← Panel
Details). Instead, the engine tracks its own running tally against Gap Sheet's **static Ask**
numbers, processed in physical row order (which matches submission order in a normal Microsoft
Forms export). See the Technical gotchas section — this is also where a real circular-reference
bug was caught and fixed.

---

## Workbook structure (weekday variant)

| Sheet | Purpose |
|---|---|
| `Config` | One cell controls the week: Monday's date. Day-token text, below-Manager list, and R2-designation list all live here and drive everything else. |
| `MasterPanel` | Company-wide approved-panelist roster, copied in fresh each time. **Column layout is not guaranteed stable between exports** — verify positions every time (see gotchas). |
| `Skill Mapping` | One row per distinct *approved* master skill. Columns: Master Skill, Survey Skill, Active (Y/N), plus 4 auto-computed helper columns (`ActiveKey`, `ActiveMasterKey`, `ActiveKey2`, `ActiveMasterKey2`) that support the many-to-many case (1st and 2nd occurrence, both directions). Only `Active=Y` rows are live; others are kept as historical/available-for-future reference, not deleted. |
| `Survey Response` | Raw Microsoft Forms export. Email lives in "Deloitte Email ID" (not the "Email" column, which can be anonymous/blank), name in "Full Name". |
| `Auto Process` | The engine. One row per survey submission; computes Check/Route/Level/Eligible?/day columns/comments, and feeds the `Panel Details` / `Yet to Confirm` live views below. |
| `SkillCheck` | One column per active Skill Mapping entry; per-person approved-skill matrix, feeds the Yet-to-Confirm "approved for: …" reason text. Must be rebuilt whenever Skill Mapping's active list changes, or the reasons go stale. |
| `Panel Details` / `Yet to Confirm` | Live views pulling from Auto Process via an `AGGREGATE`-based "find the Nth matching row" formula. `srcRow` helper column is intentionally **visible**, not hidden — this was a specific, repeated instruction. |
| `Gap Sheet` / `Pivot` | Live `SUMPRODUCT`/`SUMIFS`-based skill-demand summary, reading straight from Panel Details. **Never a native Excel PivotTable** — see gotchas. |
| `Post Chat Engine` / `Post Chat` | Finds everyone in MasterPanel who's approved for a still-gapped skill (checked at *their own* R1/R2 level) and hasn't already submitted the survey or been added to Do Not Contact. Outputs one row per person with all their qualifying skills combined, ready for a Teams/Power-Automate outreach flow. |
| `Do Not Contact` | Simple Email + Reason (dropdown: NA - Not available / Onsite / Other) list. Anyone here is excluded from Post Chat. |

The weekend variant drops `Post Chat`'s MasterPanel-approval-check complexity in its simplest
form and adds a `Final Panels` stage (manually-curated subset of Panel Details, with `Panel to
Travel` and `Booking confirmation` columns — these are genuine human judgment calls, **not**
derivable from Location or any other field; don't try to automate them).

---

## Technical gotchas (hard-won — read before editing formulas)

1. **Never use `insert_rows()`/`insert_cols()` on a sheet that already has formulas.** It moves
   cell *positions* but does not rewrite formula *text* elsewhere that references the old column
   letters — this silently corrupts every downstream reference. Rebuild the sheet fresh instead.
2. **Never build a native Excel PivotTable as Gap Sheet's data source.** Its cached source range
   freezes at creation time and silently goes stale as Panel Details grows — this has broken the
   same way (and needed the same fix) at least three separate times in this project's history.
   Always use live `SUMPRODUCT`/`SUMIFS` formulas instead.
3. **Strip external workbook links** (`Get Data`/Power Query connections to a local file path) —
   check `xl/externalLinks/`, `xl/connections.xml`, `xl/queries/` at the zip level, not just
   `wb._pivots`. One recurring real example pointed at a personal `C:\Users\...\Downloads\` path.
4. **MasterPanel's column layout can change between exports.** It has appeared both with and
   without a leading `id` column, which shifts every other column by one. Always verify the header
   row's actual positions before hardcoding column letters.
5. **Trailing whitespace / invisible non-ASCII characters in skill names and emails are a
   recurring real issue** — e.g. `"Databricks "`, `"Informatica MDM "`, a stray `\xa0` in one
   person's email that silently split them into two "people" in Post Chat. Use
   `IF(CODE(RIGHT(text,1))>127, strip-last-char, text)` for emails, and `TRIM()` everywhere skill
   names are compared.
6. **`AGGREGATE`-based "find the matching row" formulas must never scan a range that includes the
   current row's own (still-being-computed) cell**, even when the final condition would multiply
   that term to zero — Excel still has to resolve the dependency to evaluate the array, which
   creates a genuine circular reference. Caught this exact bug building the weekend FCFS-capacity
   engine; fixed by scanning only physically-prior rows instead of the full range with an
   in-formula exclusion condition.
7. **LibreOffice (used for automated recalculation/testing in this environment) and genuine
   Microsoft Excel can disagree on array-formula edge cases** — e.g. comparing a text `""` against
   a number with `<` inside `SUMPRODUCT` threw `#VALUE!` in LibreOffice specifically. Don't treat a
   clean LibreOffice recalc as proof the user's real Excel will behave identically; keep formulas
   as type-consistent as possible (numeric sentinels instead of `""`, etc.).
8. **Near-identical skill-name spellings are often genuinely different population pools, not
   typos** — e.g. "Informatica MDM SAS" (2 people), "Informatica SAS" (2 people), and "Informatica
   MDM SaaS" (10 people) are three distinct strings in MasterPanel. Always check actual headcounts
   before assuming two similar-looking names are the same thing or can be safely merged.
9. **Always cross-check a new/changed engine with an independent from-scratch Python
   re-implementation** before delivering — reading the formulas back and asserting they're correct
   is not sufficient; every major bug in this project's history was caught this way, not by
   inspection.
10. **Size dynamic ranges with realistic headroom, not maximal padding.** Oversized ranges combined
    with O(N) or O(N²) formulas (the dedup/day-merge chains, Post Chat's running tally) can make
    recalculation slow or time out. Gate expensive per-row lookups behind cheap short-circuiting
    conditions (e.g. only compute a backward scan `IF(Include="Y", ..., "")`) wherever possible.

---

## Reusable design patterns

- **Dedup to latest**: `SUMPRODUCT((EmailRange=thisEmail)*((CompletionRange>thisCompletion)+(CompletionRange=thisCompletion)*(SurveyIDRange>thisSurveyID)))=0` → "Is Latest". Survey ID is the tiebreaker for same-timestamp submissions.
- **Config-driven day tokens**: `TEXT(monday+k,"dddd")&", "&DAY(...)&ordinal-suffix-logic&" "&TEXT(...,"mmm")&", "&TEXT(...,"yyyy")` reproduces Microsoft Forms' `"Monday, 17th Aug, 2026"`-style date text. **Re-validate this against real survey data every time** — Forms' exact date-text format has changed between weeks before.
- **Skill Mapping helper columns** for many-to-many support: `ActiveKey`/`ActiveMasterKey` catch the first occurrence of a repeated key; `ActiveKey2`/`ActiveMasterKey2` catch the second. Current fan-out support is capped at 2 variants per direction — if a third genuinely distinct variant ever appears, this needs extending, not just another duplicate row (the third would be silently unreachable).
- **"Find the Nth matching row" live views**: `AGGREGATE(15,6,ROW(routeRange)/(routeRange="Panel Details"),ROW()-1)` gives the row offset of the k-th match, then `INDEX()` pulls the value — this is how Panel Details/Yet to Confirm/Post Chat all stay live without a helper table.

---

## Open / pending items

- **The day-retention/merge behavior needs to be removed from the last delivered weekday
  workbook** (`Auto Panel Gap Sheet Weekday - 2nd November to 6th November 2026.xlsx`) — it still
  has the old additive logic. Rebuilding Auto Process's day-column formulas to the simple
  "latest-only" rule (rule #1 above) is outstanding.
- **"Python AI - Healthcare"** has no corresponding entry anywhere in MasterPanel's approved-skill
  vocabulary. Nobody can be auto-matched to it until that skill exists in the master data — this
  isn't a mapping bug, there's genuinely nothing to map it to yet.
- **Informatica MDM SAS mapping was incomplete** — only mapped to a 2-person pool
  ("Informatica SAS") when a 10-person pool ("Informatica MDM SaaS") was the real intended match.
  Fix is scoped but not yet applied to the live Skill Mapping tab.

---

## Git / branch housekeeping

- **Deleting branches (local or remote) must be done manually by the repo owner** (e.g. from GitHub's
  Branches page, or `git push origin --delete <branch>` from their own machine). The Claude Code cloud
  environment cannot delete remote branches — the push is rejected/disconnected. Don't retry it; merge
  into `main`, verify the branch is fully contained (`git merge-base --is-ancestor <branch> main`), and
  ask the owner to delete it.

---

## Folder guide

- `samples/weekday/` — reference weekday workbook (real data stripped).
- `samples/weekend/` — reference weekend/in-person workbook (real data stripped).
- `message-drafts/` — reusable Teams/email templates for the recurring outreach patterns (gap-fill
  nominations, travel-booking reminders, RSVP broadcasts, approval-matrix requests). Each has
  placeholders marked `<<like this>>` — fill in before sending, don't paste verbatim.
