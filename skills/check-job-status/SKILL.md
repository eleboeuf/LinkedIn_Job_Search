---
name: check-job-status
description: "Exhaustively checks the application, rejection, and NOGO status of every job in a LinkedIn job report. Combines three sources: Gmail \"Jobs\" label (for applied/rejected), inputs/job_feedback.json (for NOGO/skip/closed overrides), and report segmentation rules. Use this skill whenever producing or updating a daily job report, whenever asked to check which jobs have been applied to or rejected, or whenever updating status flags (🟠 Applied, 🔴 Rejected, ⛔ NOGO, 🔒 Closed) in any job tracking document. This skill prevents the common mistake of only checking status for a subset of jobs — it enforces checking every single company across all sources before finalising the report."
---

# Check Job Status — Exhaustive Status Verification

> **Consolidated 7 July 2026:** this skill used to restate rules already spelled out in
> TASK_prompt.txt (badge placement, report segmentation, status text format, the
> company-name matching-key normalisation). Those are now stated ONCE, in TASK_prompt.txt,
> under the canonical rule names referenced below. This skill only holds the protocol that
> isn't stated there: how to search Gmail, how to classify a thread, and the priority order
> for merging results. If a canonical rule and this file ever seem to disagree, TASK_prompt.txt
> wins.

## Why this matters

Missing a status makes a job report misleading — the user may re-apply to a role they were
already rejected from, or waste time on a NOGO role. Three sources must ALL be checked:
Gmail "Jobs" label, inputs/job_feedback.json overrides, and the report segmentation rules
(see TASK_prompt.txt step 9 for segmentation; that part is not repeated here).

## Protocol

### Step 1 — Build the full company + title + location list (or reuse the master job list)

If TASK_prompt.txt's "MASTER JOB LIST — BUILD ONCE, REUSE FOR EVERY CHECK" already exists for
this run, use it directly — just add the resolved status from Steps 2-5 below onto each
existing entry. Only build a fresh company+title+location list here if this skill is run
standalone, outside the full daily-report pipeline. See TASK_prompt.txt's "CANONICAL RULE —
UNIQUE ROLE IDENTIFIER" section for why location is now part of this key (added 8 July 2026)
— the same company+title posted for two different locations are two different roles, not one.

### Step 2 — Load inputs/job_feedback.json

Read `inputs/job_feedback.json`. See TASK_prompt.txt step 8 for the full status_override
values (skip/closed/interview/follow-up/failed_interview/null) and where each shows up in the
report — not repeated here.

Match feedback entries to jobs by company name (fuzzy — partial match is OK, e.g. "AWS"
matches "Amazon Web Services (AWS)") AND title; when the entry's optional "location" field is
also populated (see TASK_prompt.txt's 10c LOCATION FIELD section), require location to match
too — the same company+title in two different locations are different roles, and a NOGO/skip
entry scoped to one location must not silently apply to the other. If either side lacks a
location, fall back to company+title matching (do not treat the match as failed just because
location is unknown). Before matching, normalise the company name using TASK_prompt.txt's
canonical normalisation (strip any trailing parenthetical annotation, e.g. "Grey Matter
Recruitment (AI-Native MarTech client)" → "Grey Matter Recruitment", then lowercase/strip
punctuation/whitespace) — see the MATCHING KEY BUG section there. Normalise location the same
way (treating "London" / "Greater London" / "London, UK" as equivalent). Do not re-derive a
second normalisation rule here.

NOGO status takes priority over Gmail status — if a job is marked NOGO in
job_feedback.json AND has an application in Gmail, show it as ⛔ NOGO (the user deliberately
flagged it after applying). Closed status is DIFFERENT (updated 8 July 2026, per user
request — APPLIED VS CLOSED PRECEDENCE, see TASK_prompt.txt): a job marked "closed" that
ALSO has an application in Gmail shows its 🟠 Applied/🔴 Rejected status instead of 🔒 Closed,
and moves to the 📂 Already Applied section rather than 🚫 Excluded — you already applied
before it closed, so that's what matters. Closed only wins (job stays 🔒 Closed in Excluded)
when there is no application on record at all.

### Step 3 — Search Gmail per company

Read gmail_job_applications_label from config.json (fallback "Jobs"). Per TASK_prompt.txt's
canonical Gmail-label rule, always search using the label's display name, never its internal
ID.

For each company in the list, run an **individual** search: `label:<label name> "CompanyName"`.
Search the last N days per application_lookback_days (fallback 120).

Do **not** substitute one large OR query (`label:Jobs (CompanyA OR CompanyB OR ...)`) — large
OR queries silently truncate results when there are many matches. Individual per-company
searches take a little longer but are far more reliable.

### Step 4 — Classify each Gmail result (company + title matching)

For each thread found, determine application-confirmation vs. rejection — **and verify the
job title matches, not just the company**. Large companies (Google, Amazon, Salesforce,
Pegasystems, etc.) post many roles simultaneously; an application to "Partner Sales Director
at Google" six months ago does not mean the user applied to a different Google role that
appeared in today's alerts.

**Matching rules:**
- Gmail title matches report title (exact or close semantic match) → assign the status.
- Gmail title is clearly a **different role** at the same company → do NOT assign the status;
  note "Applied to different role: [title]" for context, leave the report job unflagged.
- Not enough detail to determine the specific role → note as ambiguous, leave unflagged. Do
  not guess.

**Application confirmation signals:** "your application was sent", "thank you for applying",
"application received", "we received your application", "applied to", "application
submitted", LinkedIn "Easy Apply" confirmation.

**Rejection signals:** "not move forward", "decided not to", "unfortunately", "regret to
inform", "not selected", "other candidates", "role has been filled", "position has been
filled", "we will not be moving", "chosen not to proceed".

If both types appear for the same company AND role, record both — most recent wins for the
status flag, but note the history (e.g. "Applied May 11, Rejected Jun 3").

**ATS platform caution:** third-party ATS platforms (HiBob, Greenhouse, Workday, Lever, etc.)
send recruiting emails on behalf of the actual employer. An email from `no-reply@hibob.com`
does not mean the user applied to HiBob — read the body to identify the real employer.

### Step 5 — Merge all statuses and record

Combine Gmail results with job_feedback.json overrides, in this priority order:
1. ⛔ NOGO / 🔒 Closed (from job_feedback.json) — highest priority
2. 🔴 Rejected (from Gmail, title-matched)
3. 🟠 Applied (from Gmail, title-matched)
4. No flag — nothing found in either source

Keep a running tally with a "Gmail Title" column so it's explicit which role was matched:

| Company | Report Title | Gmail Title | Gmail Status | Feedback | Final Status |
|---------|-------------|-------------|-------------|----------|-------------|
| Acme | VP Partnerships | VP Partnerships | Applied May 14 | — | 🟠 Applied |
| Google | AI Partner Mgr | Partner Sales Dir | Applied Jun 19 | — | — (different role) |
| Garuda | Dir Partnerships | — | — | skip | ⛔ NOGO |

Complete the table for every company before writing any status flags into the report.

### Step 5b — Visit every LinkedIn job page

See TASK_prompt.txt step 8b for the full skip/recheck/parallel-batching logic — not repeated
here. The one thing worth restating: this is historically the #1 source of errors in this
report (a past run visited only 5 of 57 pages, causing 7 closed jobs to show as active) —
"visited a few and moved on" is not the same as "visited all of them per step 8b's rules."

### Step 6 — Apply flags and segmentation to the report

See TASK_prompt.txt step 9 for the report segmentation rules (High Priority / Other Roles /
Excluded) and the NOGO/CLOSED BADGE PLACEMENT section for exactly where the ⛔/🔒 indicator
is allowed to appear. Apply those rules as written — this skill does not restate them.

## Common mistakes to avoid (execution discipline only — see TASK_prompt.txt for rule content)

1. **Forgetting inputs/job_feedback.json.** Always load it before finalising the report.
2. **Stopping after the first few finds.** Keep going through the entire company list even if
   the first few all resolve cleanly.
3. **Assuming silence means "no application."** A missing result might mean the confirmation
   email was labelled differently, went to spam, or was never sent — note "not found," don't
   assume.
4. **Confusing the ATS sender with the employer.** Always read the body to identify the real
   hiring company.
5. **Skipping companies with low scores.** Status checking isn't about score — a low-scoring
   job the user applied to should still be flagged 🟠.
6. **Using a single search for all companies.** Always search individually, per company.
7. **Matching on company name alone.** A Gmail hit for "Google" doesn't mean the user applied
   to every Google role in the report — always confirm the specific title (Step 4).
