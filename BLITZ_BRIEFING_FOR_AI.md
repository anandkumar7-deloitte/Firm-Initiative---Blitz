# Blitz Briefing — Context for an AI Model Taking Over or Assisting

Audience: another AI model (or a new human teammate) that needs to understand what "Blitz" is, what the
owner's work consists of, and how to work on it safely. Read this first, then `CLAUDE.md` in the repo
root (the authoritative, more technical rule/gotcha list), then the sample workbooks.

---

## 1. Business context

- **Organisation:** Deloitte USI. The "Blitz" team runs recurring **interview drives** ("Blitzes") to recruit
  **AI & Data** interview panelists, i.e. internal SMEs who will interview candidates.
- **Each Blitz has:**
  1. A **skill-gap demand** — e.g. "need 40 R1 slots for Databricks". Skills are split into **R1** and **R2**
     interview rounds (levels).
  2. A **Master Panel** list — a company-wide roster of SMEs pre-approved per skill (`Panel Approved = Y`).
  3. A **Microsoft Forms survey** where SMEs self-declare availability and the skill they will interview for.
- **The owner's job** is to turn survey submissions into a clean, confirmed panel and to chase remaining gaps.
  The workbooks in this repo automate that so that panelists route themselves into **Panel Details**
  (confirmed) or **Yet to Confirm** (needs follow-up) with no manual triage.
- **Outreach** (Teams / email / Power Automate) is a big part of the work: gap-fill nominations, travel-booking
  reminders, RSVP broadcasts, approval-matrix requests. Templates live in `message-drafts/` with
  `<<placeholders>>` that must be filled before sending.

## 2. Two workbook variants

| | Weekday | Weekend / In-person |
|---|---|---|
| Format | Virtual, Mon–Fri | Single day, in person, in a specific city |
| Availability | Per-weekday checkbox + time windows | "Entire Duration" Yes/No + free-text custom time |
| Slot count | Computed from selected windows | Validated full-day constant (varies per event — re-derive from that event's own history, never assume a prior event's number) or a flat number if the team explicitly says so |
| Samples | `samples/weekday/` | `samples/weekend/` |

Samples have had **all real employee data stripped** (MasterPanel is kept; it is routinely refreshed
reference data). Never commit real personal availability data.

## 3. Business rules currently in force

1. **One row per person, latest submission wins, no merging.** Older submissions are tagged `Superseded`.
   An earlier "additive/merge across submissions" design was explicitly reversed — do not reintroduce it.
2. **Eligibility** = Approved in MasterPanel (email + mapped skill, `Panel Approved = Y`) **and** the
   architect check (skills containing "architect" exclude anyone below Manager).
3. **Level:** non-architect — Manager and above → R2, others → R1. Architect — inverted: Manager → R1,
   Senior Manager / Specialist Master and above → R2.
4. **Routing:** latest + eligible → Panel Details; latest + not eligible → Yet to Confirm with a short
   "approved for X instead" reason; not latest → Superseded.
5. **Many-to-many skill mapping** between MasterPanel's vocabulary and the survey's (one master → two survey
   skills; one survey → two master skills), implemented via helper columns in `Skill Mapping`
   (capped at two variants per direction).
6. **Weekend only:** check Primary skill first (approved and capacity left), else Secondary, else Followup.
   Capacity is tracked first-come-first-served against Gap Sheet's **static Ask**, not the live "Available"
   number (that would be a circular reference).

## 4. NEW direction for the November 2026 Blitz (stated by the owner, not yet implemented)

- **Only Senior Consultants (SCon) and Managers are eligible panelists.** Consultants are no longer taken
  (they had been, previously).
- Owner clarified: stick to "Senior Consultant and Managers" **only** — i.e. Analysts, Consultants,
  Senior Managers and above are all out of scope for this Blitz.
- Applies to **both** weekday and weekend workbooks.
- Out-of-scope designations are **suspended**, not routed to Yet to Confirm with a reason.
  *Working interpretation (unconfirmed):* tag the record `Suspended` in Auto Process (like `Superseded`),
  excluded from Panel Details, Yet to Confirm, Gap Sheet and Post Chat, while remaining visible in Auto
  Process for audit.
- **Consequences to keep in mind:**
  - Non-architect: SCon → R1, Manager → R2 (unchanged mapping).
  - Architect: SCon already excluded; Manager → R1; R2 normally needs Senior Manager+, who are now excluded,
    so **architect R2 has no eligible panelists** this Blitz unless the owner decides otherwise (open question).
  - Apply the designation check to the **latest** submission only; do not resurrect an older submission
    because the latest is suspended.
  - Drive the allowed-designation list from `Config` (like the existing designation lists), never hardcode.
  - Post Chat must also exclude out-of-scope designations from MasterPanel outreach.
- **Status:** the owner said "do not create anything" until they ask. Do not edit workbooks for this until
  explicitly told to build it.

## 5. Workbook structure (weekday)

`Config` (Monday date, day tokens, designation lists) → `MasterPanel` (roster; layout not stable between
exports) → `Skill Mapping` (Master Skill, Survey Skill, Active Y/N + 4 helper key columns) →
`Survey Response` (raw Forms export; email is in "Deloitte Email ID", name in "Full Name") →
`Auto Process` (the engine, one row per submission) → `SkillCheck` (approved-skill matrix for reason text) →
`Panel Details` / `Yet to Confirm` (live views via AGGREGATE "Nth match" formulas; `srcRow` column is
intentionally visible) → `Gap Sheet` / `Pivot` (live SUMPRODUCT/SUMIFS, never a native PivotTable) →
`Post Chat Engine` / `Post Chat` (MasterPanel people approved for a still-gapped skill who haven't
submitted or been marked Do Not Contact) → `Do Not Contact`.

Weekend adds a `Final Panels` stage (manually curated subset of Panel Details with `Panel to Travel` and
`Booking confirmation` — human judgement calls; do not try to automate them).

## 6. Hard-won gotchas (summary; full list in CLAUDE.md)

- Never `insert_rows`/`insert_cols` on a sheet with formulas — rebuild the sheet.
- Never use a native PivotTable as Gap Sheet source (cached range goes stale).
- Strip external links / Power Query connections at zip level.
- Verify MasterPanel column positions every export (leading `id` column sometimes appears).
- Trim whitespace and non-ASCII junk in skill names and emails (e.g. `\xa0`).
- AGGREGATE formulas must not scan a range containing their own cell (circular reference) — scan prior rows only.
- LibreOffice and Excel can disagree on array edge cases; use type-consistent formulas (numeric sentinels).
- Near-identical skill names are often distinct populations (e.g. Informatica MDM SAS vs SaaS) — check headcounts.
- **Always cross-check a changed engine with an independent from-scratch Python re-implementation** before
  delivering; every major past bug was found that way.
- Keep dynamic ranges sized with realistic headroom; gate expensive scans behind cheap conditions.

## 7. Open items

- Remove the old day-merge/retention logic from the delivered weekday workbook
  (`Auto Panel Gap Sheet Weekday - 2nd November to 6th November 2026.xlsx`).
- "Python AI - Healthcare" has no MasterPanel vocabulary entry; nothing to map to yet.
- Informatica MDM SAS mapping points at the 2-person pool; the intended 10-person "Informatica MDM SaaS"
  fix is scoped but not applied.
- November 2026 designation restriction (section 4) — awaiting a go-ahead and two confirmations
  (meaning of "suspend"; architect R2 treatment).

## 8. How to work with this owner

- Do exactly what is asked; if told "do not create anything", do not edit files or produce deliverables.
- Confirm ambiguous business rules before building; summarise your understanding first.
- Prefer config-driven changes; report outcomes faithfully, including anything unverified.
- Do not commit real employee data. Do not open PRs unless asked. Develop on the designated branch only.
- Past specific instructions to respect: `srcRow` stays visible; Gap Sheet stays formula-based; no merge
  behaviour for multi-submission people.
