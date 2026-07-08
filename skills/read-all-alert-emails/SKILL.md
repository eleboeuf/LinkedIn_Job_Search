---
name: read-all-alert-emails
description: "Ensures every LinkedIn Alert email is fully opened and read when building a job report — including \"your job alert has been created\" confirmation emails, which LinkedIn uses to deliver the first batch of matching jobs. Use this skill whenever processing LinkedIn job alerts, fetching job listings from email, building or updating a daily job report, or any task that involves extracting job postings from LinkedIn Alert emails. This skill prevents the common mistake of skipping new-alert confirmation emails that look like setup notifications but contain real job listings inside."
---

# Read All LinkedIn Alert Emails — Exhaustive Extraction

> **Consolidated 7 July 2026:** the Gmail label-ID-vs-display-name warning used to be
> restated here (and in check-job-status/SKILL.md) nearly verbatim. It now lives once, in
> TASK_prompt.txt's canonical "GMAIL LABEL SEARCH" rule — see that instead of looking for it
> here. Everything below is protocol specific to this skill (recognising both alert email
> types, parsing job blocks out of the body) that isn't stated anywhere else.

## Why this matters

LinkedIn sends two types of alert emails, and both contain job listings:

1. **Ongoing alert emails** — subject like `"director of strategic partnerships": Acme - Role
   posted on 6/19/26`. Easy to spot: a single job or small digest, clearly labelled.
2. **New alert confirmation emails** — subject like `Eric: your job alert for ("CPaaS" OR
   "CCaaS") has been created`. These look like setup notifications, but LinkedIn also
   includes the **first batch of matching jobs** inside the body — indistinguishable from an
   ongoing alert in content, just a different subject line.

Skipping type 2 misses entire batches of postings — this is exactly what happened when
Cresta (99), Google (94), Zendesk (89), and Guidewire (83) were all omitted because they
lived inside "has been created" emails.

## Protocol

### Step 1 — Fetch all alert emails without filtering by subject

Search only the Gmail label defined by gmail_job_alerts_label in config.json (fallback:
"Linkedin Alerts"), using the canonical display-name rule in TASK_prompt.txt (never the
internal label ID). Use a broad query that captures everything:

```
label:<gmail_job_alerts_label from config.json, spaces→hyphens> newer_than:<alerts_lookback_days from config.json>d
```

Do **not** add filters like `-subject:"has been created"` or `subject:"posted on"` — both
alert types must be included.

### Step 2 — Open every thread and read the body

For every thread returned, call `get_thread` and extract the `plaintextBody`. Do not rely on
the subject line snippet alone — it only shows the first line ("See your latest job matches")
with no job details.

Job listings in the plain-text body follow this pattern:

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

**First job appears before the first separator**: the first listing in each email sits
between the "Your job alert for..." header and the first line of dashes. If you split the
body on dash separators, you must also process the content *before* the first separator —
that block zero contains a real job. Skipping it caused production misses of high-scoring
roles like Cresta (95) and Attentive (81).

### Step 3 — Extract all fields per job

For every job block, extract: job title, company, location, salary range (from the email
body if present; also check the LinkedIn job page via Chrome tools for salary not in the
email — leave blank only if genuinely not published anywhere), apply link (the "View job:"
URL), a 2-3 line summary, seniority level (infer from title if not explicit), and keywords
detected (CPaaS, CCaaS, CX, AI, partnerships, channel, ecosystem, alliances, GTM, etc.).

### Step 4 — Deduplicate across emails

The same job may appear in multiple alert emails because it matched several different
keyword searches. Track companies already extracted and skip duplicates within this step,
keeping the first occurrence's apply link. (Cross-run dedup by job ID, and the
company+title+location repost check for jobs reposted under a new job ID, are handled
separately in TASK_prompt.txt's DEDUPLICATION step — not repeated here. Note it's
company+title+location, not company+title alone: the same title reposted for a different
location is a different role, not a repost — see TASK_prompt.txt's CANONICAL RULE — UNIQUE
ROLE IDENTIFIER section.)

### Step 5 — Build the complete company list before scoring

Only after extracting jobs from **all** emails should scoring and ranking start. A company
that appears in a "has been created" email may score very high (Cresta 99, Google 94) and
would change the entire ranking if added after the fact.

## Quick reference — recognising the two email types

| Email type | Subject pattern | Contains jobs? |
|------------|----------------|---------------|
| Ongoing alert | `"keyword": Company – Role posted on date` | Yes — usually 1 job |
| New alert confirmation | `Eric: your job alert for ("X" OR "Y") has been created` | **Yes — first batch of matches** |
| Application confirmation | `Eric, your application was sent to Company` | No — skip for extraction |
| Saved jobs reminder | (informational) | No — skip for extraction |
