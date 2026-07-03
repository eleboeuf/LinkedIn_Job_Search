---
name: read-all-alert-emails
description: "Ensures every LinkedIn Alert email is fully opened and read when building a job report — including \"your job alert has been created\" confirmation emails, which LinkedIn uses to deliver the first batch of matching jobs. Use this skill whenever processing LinkedIn job alerts, fetching job listings from email, building or updating a daily job report, or any task that involves extracting job postings from LinkedIn Alert emails. This skill prevents the common mistake of skipping new-alert confirmation emails that look like setup notifications but contain real job listings inside."
---

# Read All LinkedIn Alert Emails — Exhaustive Extraction

## Why this matters

LinkedIn sends two types of alert emails, and both contain job listings:

1. **Ongoing alert emails** — subject like `"director of strategic partnerships": Acme - Role posted on 6/19/26`
   These are easy to spot: a single job or small digest, clearly labelled.

2. **New alert confirmation emails** — subject like `Eric: your job alert for ("CPaaS" OR "CCaaS") has been created`
   These look like setup notifications, but LinkedIn also includes the **first batch of matching jobs** inside the body. They are indistinguishable from ongoing alerts in terms of content — they just have a different subject line.

If you skip type 2, you will miss entire batches of job postings. This is exactly what happened when Cresta (score 99), Google (94), Zendesk (89), and Guidewire (83) were all omitted from a report because they lived inside "has been created" emails.

## Critical — Gmail label naming in search queries

When searching Gmail with the `search_threads` tool, always use the **human-readable
label name** in the query string — never the internal label ID.

- **Correct:** `label:LinkedIn-Alerts` (display name, spaces become hyphens)
- **Wrong:** `label:Label_229824024361616313` (internal ID — returns empty results)

Gmail search syntax expects the display name with hyphens replacing spaces.
The `list_labels` tool returns internal IDs like `Label_29` or `Label_229824024361616313`,
but these IDs **do not work** in search queries and silently return zero results.

This applies to every label used in this workflow:

| Label display name | Search query syntax | Internal ID (do NOT use) |
|---|---|---|
| LinkedIn Alerts | `label:LinkedIn-Alerts` | `Label_229824024361616313` |
| Jobs | `label:Jobs` | `Label_29` |

The reason this matters: using internal IDs caused an entire daily report to miss
all label-filtered emails, forcing a fallback to `from:` queries that lost the
label-based filtering entirely.

## Protocol

### Step 1 — Fetch all alert emails without filtering by subject

Search only the Gmail label defined by gmail_job_alerts_label in config.json (fallback: "Linkedin Alerts"). Replace spaces with hyphens when using it in the Gmail search query. Ignore all other folders or labels.

Use a broad query that captures everything:

```
label:<gmail_job_alerts_label from config.json, spaces→hyphens> newer_than:<alerts_lookback_days from config.json>d
```

Do **not** use the internal label ID (e.g. `label:Label_229824024361616313`) — it will return empty results. Use the display name as shown above.

Do **not** add filters like `-subject:"has been created"` or `subject:"posted on"`. Both alert types must be included.

### Step 2 — Open every thread and read the body

For every thread returned, call `get_thread` and extract the `plaintextBody`. Do not rely on the subject line snippet alone — the snippet only shows the first line ("See your latest job matches") which gives no job details.

Process each email body to extract all job blocks. In the plain-text body, job listings appear in this pattern:

```
[Job Title]
[Company Name]
[Location]
[Salary range — if published]
[Seniority level — if shown]

[Optional: "This company is actively hiring" / "X connections"]
Apply with resume & profile  OR  View job: [URL]
```

Job blocks are separated by a line of 20+ dashes: `-------------------------`

**First job appears before the first separator**: The first job listing in
each email sits between the "Your job alert for..." header and the first
line of dashes. If you split the body on dash separators, you must also
process the content *before* the first separator — that is block zero and
contains a real job. Skipping it caused production misses of high-scoring
roles like Cresta (95) and Attentive (81).

### Step 3 — Extract all fields per job

For every job block, extract:

- **Job title**
- **Company**
- **Location**
- **Salary range** — extract from the email body if present. Also check the LinkedIn job page (via Chrome browser tools) for salary data not included in the email. Display in cards and as a dedicated Salary column in tables. Leave blank only if genuinely not published anywhere.
- **Apply link** (the "View job:" URL)
- **Summary** (2–3 lines describing the role)
- **Seniority level** (e.g. Director, VP, Senior Manager — infer from title if not explicit)
- **Keywords detected** (match against target keywords: CPaaS, CCaaS, CX, AI, partnerships, channel, ecosystem, alliances, GTM, etc.)

### Step 4 — Deduplicate across emails

The same job (e.g. "Scout Global — Director of Strategic Partnerships") may appear in multiple alert emails because it matched several different keyword searches. Track companies already extracted and skip duplicates. Keep the first occurrence's apply link.

### Step 5 — Build the complete company list before scoring

Only after extracting jobs from **all** emails should you start scoring and ranking. The reason: a company that appears in a "has been created" email may have a high score (like Cresta at 99 or Google at 94) and would change the entire ranking if added after the fact.

## Quick reference — recognising the two email types

| Email type | Subject pattern | Contains jobs? |
|------------|----------------|---------------|
| Ongoing alert | `"keyword": Company – Role posted on date` | Yes — usually 1 job |
| New alert confirmation | `Eric: your job alert for ("X" OR "Y") has been created` | **Yes — first batch of matches** |
| Application confirmation | `Eric, your application was sent to Company` | No — skip for extraction |
| Saved jobs reminder 