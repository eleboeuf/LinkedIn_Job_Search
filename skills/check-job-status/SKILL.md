---
name: check-job-status
description: "Exhaustively checks the application, rejection, and NOGO status of every job in a LinkedIn job report. Combines three sources: Gmail \"Jobs\" label (for applied/rejected), inputs/job_feedback.json (for NOGO/skip/closed overrides), and report segmentation rules. Use this skill whenever producing or updating a daily job report, whenever asked to check which jobs have been applied to or rejected, or whenever updating status flags (🟠 Applied, 🔴 Rejected, ⛔ NOGO, 🔒 Closed) in any job tracking document. This skill prevents the common mistake of only checking status for a subset of jobs — it enforces checking every single company across all sources before finalising the report."
---

# Check Job Status — Exhaustive Status Verification

## Why this matters

Missing a status makes a job report misleading. The user may re-apply to a
role they were already rejected from, waste time on a NOGO role, or miss that
a position has closed. There are three status sources that must ALL be checked:

1. **Gmail "Jobs" label** — application confirmations and rejection emails
2. **inputs/job_feedback.json** — manual NOGO/skip/closed overrides from the user
3. **Report segmentation rules** — partnership roles must never be excluded

## Protocol

### Step 1 — Build the full company + title list

Before checking any source, list every company AND job title that appears in
the current report or email batch. Write them down as pairs (as a Python list
of tuples, a text file, or inline comments). Do not start checking until the
list is complete. You need both fields because status matching requires
**company + title**, not company alone.

### Step 2 — Load inputs/job_feedback.json

Read `inputs/job_feedback.json` from the workspace folder (same folder as
TASK_prompt.txt). This file contains the user's manual status overrides:

```json
[
  {
    "company": "Company Name",
    "title": "Job Title",
    "date_added": "2026-06-23",
    "comment": "NOGO — reason here",
    "status_override": "skip"
  }
]
```

**status_override values:**
- `"skip"` → Mark as ⛔ NOGO. Show the comment as the reason. Job stays
  visible in the report but greyed out and not actionable.
- `"closed"` → Mark as 🔒 Closed. The position is no longer accepting
  applications.
- `"interview"` → Show in the 🎯 Open Interviews section at the top of the
  report. These are roles the user is actively interviewing for.
- `"follow-up"` → Show in the 📌 Follow-up table. These are roles the user
  has applied to and wants to track.
- `"failed_interview"` → Show in the 💔 Failed Interviews collapsible section,
  displayed just before the Follow-up table. Sorted by date descending.
  Do NOT count these as active interviews. Do NOT show in Open Interviews.
- `null` → No override. The comment is informational only (e.g., "Already
  applied 10 May 2026" — still check Gmail to confirm).

Match feedback entries to jobs by company name (fuzzy — partial match is OK,
e.g., "AWS" matches "Amazon Web Services (AWS)"). When both company AND title
match, that's a strong match. When only company matches, still apply it but
note the title difference.

NOGO/closed status takes priority over Gmail status. If a job is marked NOGO
in inputs/job_feedback.json AND has an application in Gmail, show it as ⛔ NOGO (the
user deliberately flagged it after applying).

### Step 3 — Search Gmail per company

Read the gmail_job_applications_label value from config.json to determine the correct label name (fallback: "Jobs").

For each company in the list, run an **individual** Gmail search:

```
label:<gmail_job_applications_label from config.json> "CompanyName"
```

**Gmail label syntax — critical**: Always use the label's **display name**
(e.g., `label:Jobs`), never its internal ID (e.g., `label:Label_29`). The
Gmail MCP `list_labels` tool returns both a `name` (display name) and an `id`
(internal ID like `Label_29`). Using the ID in a search query silently returns
zero results — it does not produce an error, so the failure is invisible. If
you are unsure which is the display name, call `list_labels` and use the
`name` field.

Search the last N days as defined by application_lookback_days in config.json (fallback: 120 days).  Retrieve at least
the first few matching threads and read the subject lines and body snippets.

Do **not** substitute a single large OR query like:
```
label:Jobs (CompanyA OR CompanyB OR CompanyC ...)
```
Large OR queries silently truncate results when there are many matches and are
harder to reason about per-company. Individual searches take a little longer
but are far more reliable — the cost is worth it.

### Step 4 — Classify each Gmail result (company + title matching)

For each thread found, determine whether it represents an application
confirmation or a rejection — **and verify the job title matches**.

**This is critical: match on BOTH company AND job title, not company alone.**

Large companies (Google, Amazon, Salesforce, Pegasystems, etc.) post many
roles simultaneously. The user may have applied to "Partner Sales Director at
Google" six months ago — that does NOT mean they applied to "Strategic Partner
Development Manager, AI Pureplay, EMEA at Google" which appeared in today's
alerts. These are different roles and must be treated independently.

When a Gmail thread matches the company name, read the email subject and body
to extract the **specific job title** that was applied to or rejected from.
LinkedIn application confirmation emails typically include the role title in
the subject line (e.g., "Your application to Director, Partner Sales UKI at
Pegasystems" or "Eric, your application was sent to Google" with the role
title in the body).

**Matching rules:**
- If the Gmail job title matches the report job title (exact or close
  semantic match — e.g., "Director of Partnerships" ≈ "Director,
  Partnerships"), assign the status to that job.
- If the Gmail job title is clearly a **different role** at the same company,
  do NOT assign the status. Note it in the tally as "Applied to different
  role: [title]" for context, but leave the report job unflagged.
- If the email does not contain enough detail to determine the specific role
  (rare — usually the title is in the subject), note it as ambiguous and
  leave the job unflagged. Do not guess.

**Application confirmation signals** (any of these in subject or body):
- "your application was sent"
- "thank you for applying"
- "application received"
- "we received your application"
- "applied to"
- "application submitted"
- LinkedIn "Easy Apply" confirmation

**Rejection signals** (any of these in subject or body):
- "not move forward"
- "decided not to"
- "unfortunately"
- "regret to inform"
- "not selected"
- "other candidates"
- "role has been filled"
- "position has been filled"
- "we will not be moving"
- "chosen not to proceed"

If both types are present in different threads for the same company AND same
role, record **both** — the most recent one wins for the status flag, but note
the history (e.g., "Applied May 11, Rejected Jun 3").

**ATS platform caution**: Some companies use third-party ATS platforms
(HiBob, Greenhouse, Workday, Lever, etc.) to send recruiting emails. An email
from `no-reply@hibob.com` does **not** mean the user applied to HiBob — it
means a company *using* HiBob sent the email. Read the body to identify the
actual employer before assigning a status.

### Step 5 — Merge all statuses and record

Combine Gmail results with inputs/job_feedback.json overrides. Priority order:

1. **⛔ NOGO / 🔒 Closed** (from inputs/job_feedback.json) — highest priority
2. **🔴 Rejected** (from Gmail, title-matched) — rejection found
3. **🟠 Applied** (from Gmail, title-matched) — application sent, no rejection
4. No flag — nothing found in either source

Keep a running tally. The "Gmail Title" column makes it explicit which role
was matched:

| Company | Report Title | Gmail Title | Gmail Status | Feedback | Final Status |
|---------|-------------|-------------|-------------|----------|-------------|
| Acme | VP Partnerships | VP Partnerships | Applied May 14 | — | 🟠 Applied |
| Google | AI Partner Mgr | Partner Sales Dir | Applied Jun 19 | — | — (different role) |
| Garuda | Dir Partnerships | — | — | skip | ⛔ NOGO |
| Beta | Channel Lead | Channel Lead | Rejected Jun 2 | — | 🔴 Rejected |
| T3 | Growth Partner | — | — | closed | 🔒 Closed |
| Gamma | Alliances Dir | — | — | — | — |

Complete the table for every company before writing any status flags into the
report.

### Step 5b — Visit every LinkedIn job page

This step is the **#1 source of errors** in the report. In past runs, only
5 out of 57 pages were visited, causing 7 closed jobs to appear as active.
The root cause: visiting a few pages and moving on feels "done" but leaves
most jobs unchecked. The fix is systematic batch-checking.

**Execution method — use a subagent.** Spawn a subagent (Agent tool) with the
complete list of job URLs and instruct it to visit every one. The subagent
should navigate to each URL, read the page text, and return a table of
company | status (OPEN/CLOSED) | salary. This is faster and more reliable
than visiting pages one-by-one in the main conversation. If the subagent
times out before finishing, spawn another to complete the remaining URLs.

**What to check on each page:**

1. **Is it still accepting applications?** Look for "No longer accepting
   applications", "This job is no longer available", or similar closed
   indicators. If closed:
   - Flag as 🔒 Closed in the status table
   - Add to inputs/job_feedback.json with `status_override: "closed"`
   - Move to the Excluded section (not Other Roles)

2. **Is a salary range published?** Extract it (e.g., "80K EUR/yr -
   140K EUR/yr", "£175,000 - £210,000") and record it for display in the
   report — both in High Priority cards and as a dedicated Salary column
   in tables. Leave blank only if genuinely not published.

**Which jobs to visit:** ALL jobs from the extracted list, except those
already marked `status_override: "closed"` in inputs/job_feedback.json (they were
confirmed closed in a prior run). Jobs marked NOGO/skip MUST still be
visited — the listing may have closed since the NOGO was set, and the
report needs accurate closed counts.

**Do NOT proceed to Step 6** until every job has been visited and results
logged. Verify the count: if you extracted N jobs and K are already marked
closed, you must visit N − K pages. If your visit count is lower, you
missed some.

### Step 6 — Apply flags and segmentation to the report

Once the full status table is complete, apply flags to each job card:

- **⛔ NOGO** — purple badge, greyed-out row, show the comment/reason
- **🔒 Closed** — grey badge, greyed-out row
- **🟠 Applied [date]** — orange badge
- **🔴 Rejected [date]** — red badge
- No flag — no matching status found

**Report segmentation rules** (important — these override score thresholds):

The report has three sections:

1. **⭐ High Priority** — score ≥ 75 AND not NOGO/closed
2. **📋 Other Roles** — ALL partnership/channel/ecosystem/alliance/GTM roles
   regardless of score, PLUS any other role scoring ≥ 50. NOGO rows appear
   here but greyed out.
3. **🚫 Excluded** — 🔒 Closed roles (no longer accepting applications),
   PLUS non-partnership roles (pre-sales, account executive, technical,
   generic account manager) scoring < 50.

A job is considered "partnership-related" if its title contains any of:
partnership, partners, partner, alliances, alliance, ecosystem, channel, GTM,
go-to-market, go to market.

The reason for this rule: partnership roles are always relevant to the user's
search even at lower scores — they should never be hidden in the excluded
section. Non-partnership roles (AE, pre-sales, technical) below 50 are noise
and belong in excluded.

Update the stats counters in the report header to reflect final counts
including NOGO/Closed totals.

## Common mistakes to avoid

1. **Forgetting inputs/job_feedback.json.** This file contains critical NOGO flags
   that the user has manually set. Always load it before finalising the report.

2. **Stopping after the first few finds.** Even if the first three companies
   you check all have statuses, keep going through the entire list.

3. **Assuming silence means "no application".** A missing result might mean
   the confirmation email was labelled differently, went to spam, or was never
   sent. Note it as "not found" rather than assuming it wasn't applied to.

4. **Confusing the ATS sender with the employer.** Always read the email body
   to identify who the actual hiring company is.

5. **Skipping companies with low scores.** Status checking is not about score —
   even a low-scoring job that the user applied to should be flagged 🟠.

6. **Using a single search for all companies.** Always search individually,
   per company, by name.

7. **Using the label ID instead of display name.** `label:Label_29` silently
   returns nothing. Always use the human-readable label name from config.json (gmail_job_applications_label key). Fallback: \label:Jobs`.`

8. **Putting partnership roles in Excluded.** Any role with partnership,
   alliances, ecosystem, channel, or GTM in the title belongs in Other Roles,
   never in Excluded, regardless of score.

9. **Matching on company name alone.** A Gmail hit for "Google" does not mean
   the user applied to every Google role in the report. The user applies to
   specific positions — "Partner Sales Director" is a completely different job
   from "Strategic Partner Development Manager, AI Pureplay". Always read the
   email subject/body to extract the specific job title and only flag the
   report row whose title matches. Flagging the wrong role as "Applied" is
   misleading and wastes the user's time.

10. **Visiting only a few LinkedIn pages instead of all of them.** This is
    the single most common and damaging mistake. In the 26 June 2026 run,
    only 5 of 57 pages were visited, causing 7 closed jobs (Jack & Jill,
    Mayflower, S&P Global, Resonate CX, WAX, Human Resource Quantum, SRM
    Recruitment) to appear as active in the report. The user had to catch
    the error 