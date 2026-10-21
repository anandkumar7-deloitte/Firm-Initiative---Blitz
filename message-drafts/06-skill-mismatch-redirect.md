# Skill-mismatch redirect notice

**When to use:** someone marked availability for a skill they're not currently approved for in
MasterPanel, but they do have a different approved skill with an open gap — redirect them instead
of just rejecting the submission.

---

Hi **<<Name>>**,

Reaching out on behalf of the Blitz Team for the upcoming <<Weekday/Weekend>> Blitz
(<<date range>>).

We noticed you've marked your availability for **<<requested skill>>** — however, that isn't
listed as an approved primary skill in our system just yet.

That said, we can see that you have **<<approved skill with a gap>>** as a skill, which is
approved, and we're currently short on panelists for it. Would you be willing to take up
interviews under **<<approved skill>>** for this Blitz instead?

Do let us know — we'd really appreciate your support!

Thank you!

---

### Follow-up: pointing them toward getting the originally-requested skill approved

> To move forward with **<<requested skill>>** in future, we would request you to please raise an
> approval for it as your skill. Once the approval comes through, we can append the Skill Master
> List accordingly.

### A real edge case worth knowing
If the person pushes back saying they're already approved and don't know why it was removed —
don't assume it's a data error. Check whether the skill they believe they hold is actually an
**architect-level variant** of the skill (e.g. "GCP Architect" vs "GCP DE") and whether their
designation is below Manager — the architect rule (see CLAUDE.md) blocks below-Manager designations
from architect roles regardless of how many people approve it. In that case the fix isn't more
approvals, it's re-registering them under the non-architect variant of the same skill.
