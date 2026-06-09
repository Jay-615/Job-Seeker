---
name: setup
description: First-run guided configuration for Job-Seeker. Interviews the user through an 8-stage onboarding flow (browser check, LinkedIn login check, contact info, base resume path, job criteria, target companies, scoring weights, cost guardrails) and writes config/criteria.md, config/target_companies.csv, and config/settings.md. Re-runnable — detects existing config and asks whether to overwrite or skip.
---

# /setup — first-run guided configuration

You are walking the user through the one-time setup for Job-Seeker. By the end of this skill the user should have three populated config files and a clear next step (`/scan-jobs`).

Many users are non-engineers. Keep the language friendly. Where a default makes sense, show it and let the user accept by just pressing Enter (or by selecting the recommended option).

Use the `AskUserQuestion` tool for every multiple-choice prompt and every value you collect. Quote each user's exact input back in your acknowledgements when it's helpful — it builds trust that you wrote the right thing down.

---

## Before you start: detect existing config

Check whether any of these files already exist:

- `config/settings.md`
- `config/criteria.md`
- `config/target_companies.csv`

For each file that already exists, ask the user whether to:

- **Keep** the existing file (skip this stage)
- **Overwrite** the existing file (re-run this stage and replace contents)
- **Review** the existing contents together first, then decide

If all three exist and the user chooses Keep on all of them, tell the user "Setup is already complete. Run `/scan-jobs` whenever you want to find new postings." and stop.

Otherwise, proceed through only the stages that correspond to files the user chose to (re)create.

---

## Stage 1 — Browser connection check

Goal: confirm that Claude in Chrome is installed and connected on the user's machine. The rest of Job-Seeker depends on this for LinkedIn access.

1. Call `mcp__Claude_in_Chrome__list_connected_browsers`.
2. If the call returns at least one browser, report the browser name and OS to the user and confirm: "Looks good — I can see your Chrome." Then move on.
3. If the call returns no browsers (or errors), pause and show the user this message:

   > I can't see a connected Chrome. Job-Seeker uses the Claude in Chrome
   > extension to read LinkedIn on your behalf. Install it here:
   >
   >     https://www.anthropic.com/claude-in-chrome
   >
   > Once it's installed and connected, type "ready" and I'll re-check.

   Then ask the user (via `AskUserQuestion`) whether they're ready to re-check, want to skip the check for now (continue without LinkedIn — note that scanning and scoring will be degraded), or want to abort setup. If they say ready, call `mcp__Claude_in_Chrome__list_connected_browsers` again.

Never proceed past this stage silently if Chrome isn't connected — either confirm it is, or get an explicit "skip for now" from the user.

---

## Stage 2 — LinkedIn login check

Goal: confirm the user is logged into LinkedIn in their Chrome.

Ask the user:

> Open Chrome and go to https://www.linkedin.com. Are you logged in? (If you're not sure, look at the top-right corner — your profile photo should be visible.)

Options: **Yes, I'm logged in** / **No, I need to log in first** / **Skip**.

If the user picks "No," wait for them to log in and re-ask. Don't try to drive the login yourself.

If the user skipped Stage 1 (no Chrome connected), skip this stage too with a note that LinkedIn-dependent features will fail until the browser is connected.

---

## Stage 3 — Contact info → `config/settings.md`

Goal: collect the user's contact info for resumes and cover letters, and start writing `config/settings.md`.

Collect, one at a time using `AskUserQuestion`:

- **Full name**
- **Email address** — validate it looks like an email (contains `@` and a `.` after the `@`). If it doesn't, ask again.
- **Phone number** — accept any format the user types; don't force a specific shape.
- **Location** — "City, ST" format (e.g., "Portland, OR"). Used in cover letters and as a soft signal for some scoring.

**As soon as you have all four values, write `config/settings.md` immediately** with the contact-info section and placeholders for the rest. This is by design: if anything crashes mid-setup, the user's contact info is already saved.

Use this template for the partial write (you'll update it again at the end of Stage 4 and Stage 7/8):

```markdown
# Job-Seeker Settings

## User Profile
- Name: <name>
- Email: <email>
- Phone: <phone>
- Location: <city, state>
- Base resume path: <PENDING — filled in Stage 4>

## Scoring Weights
- Experience match: 0.5
- Network proximity: 0.3
- Target company: 0.2

## Threshold
- Draft-ready threshold: 70

## Sources (turn off any to skip)
- linkedin: true
- indeed: true
- dice: true
- company_career_pages: true

## Cost Guardrails (company career pages only)
- priority_filter: High
- max_pages_per_run: 10
- cache_results_days: 3

## Network Proximity
- network_cache_days: 30
- include_deep_research_default: false

## Output
- Resume formats: markdown, docx
- Cover letter formats: markdown, docx
- Pandoc path: /usr/local/bin/pandoc
```

Tell the user: "I saved your contact info to `config/settings.md` so you don't lose it if we get interrupted."

---

## Stage 4 — Base resume path → `config/settings.md`

Goal: capture the absolute path to the user's existing base resume file (Markdown, DOCX, or PDF — the user keeps editing the source themselves; `/draft-tailored-resume-coverletter` reads from it).

Ask:

> What's the absolute path to your existing base resume on this Mac?
> (Example: `/Users/yourname/Documents/resume.md`)

**Validation:**

- The path must start with `/`. If the user gives a relative path or a `~/...` path, expand it (`~` → the user's home directory) and re-ask if the expansion isn't obvious.
- Use the `Read` tool (or `Bash` `ls -l "<path>"`) to confirm the file exists and is readable. If not, tell the user clearly what went wrong and ask again. Common cases:
  - File doesn't exist → "I can't find a file at that path. Did you mean a different filename?"
  - Path is a directory → "That's a folder, not a file. Which resume file inside it?"
  - Permission denied → "I can see the path but can't open the file. Check the permissions or pick a different copy."
- Once validated, update `config/settings.md` — replace the `<PENDING — filled in Stage 4>` placeholder with the absolute path.

---

## Stage 5 — Job criteria → `config/criteria.md`

Goal: build the user's `config/criteria.md`. Use `config/criteria.example.md` (the Sam Chen persona) as a structural reference — same headings, but with the user's answers.

Collect:

1. **Roles** — "Which job titles should we search for?" Show common patterns from the example (Senior Product Manager, Product Manager, Program Manager, Project Manager). The user enters a comma-separated list or accepts a suggested set.
2. **Locations** — "Which locations work for you?" Multi-select with sensible options drawn from the user's Stage 3 location, plus "Remote (US-based)" and "Hybrid with <user's city> HQ." The user can also add custom entries.
3. **Seniority** — Two sub-questions:
   - "Which seniority levels do you accept?" (multi-select: IC / Senior IC / Lead / Principal / Manager / Director)
   - "Which do you want to reject outright?" (multi-select: VP+ / Head of / Chief / Associate / Junior / Intern)
4. **Compensation floor** — "What's your minimum acceptable compensation?" Ask for hourly and annual figures, or accept a single one. Default phrasing: "$X/hour OR $Y annual (only applied if comp is disclosed in posting)."
5. **Job types** — Multi-select: Full-time / Contract / Contract-to-hire / Part-time.
6. **Industries** — "Which industries are a strong fit for you?" Free text or comma-separated list. If the user has multiple role types (e.g., PM and Program Manager), ask whether the industries differ per role and capture that distinction.

Write `config/criteria.md` using this structure:

```markdown
# Job Search Criteria

## Roles (job titles to search for)
- <role 1>
- <role 2>
...

## Locations
- <location 1>
- <location 2>
...

## Seniority
- Accept: <levels, comma-separated>
- Reject: <levels, comma-separated>

## Compensation floor
- $<hourly>/hour OR $<annual> annual (only applied if comp is disclosed in posting)

## Job types
- <type 1>
- <type 2>
...

## Industries (used as soft signal during scoring)

For <role type A>:
- <industry 1>
- <industry 2>
...

For <role type B>:
- <industry 1>
...
```

After writing, show the user the file contents and ask: "Look good, or want to tweak anything?"

---

## Stage 6 — Target companies → `config/target_companies.csv`

Goal: produce `config/target_companies.csv` per the schema:

```
id,company,linkedin_company_id,location,city,contacts,priority,status,roles_of_interest,notes
```

Ask the user which path they want:

- **A. I already have a target companies file** — give me the path. (Then read it, detect whether it matches the schema, and migrate it — see migration notes below.)
- **B. Start from the example file** — seed `target_companies.csv` from `config/target_companies.example.csv` (the Sam Chen fictional companies) so the user has a structural reference. Make clear the seeded companies are fictional and the user should replace them over time.
- **C. Start empty** — write a CSV with just the header row.

**Migration (path A):**

If the user supplies a path, read it and look for the columns in the schema. If columns are missing — most likely `linkedin_company_id` — tell the user explicitly:

> Your file is missing these columns: `<list>`. I'll add them with blank values; future skills (like `/network-check`) will look them up on first use and write them back to the CSV. OK to proceed?

If the existing file has unknown columns the schema doesn't include, preserve them at the end of each row (don't silently drop user data). Note this in the summary.

Write the result to `config/target_companies.csv` and report the row count to the user.

---

## Stage 7 — Scoring weights & threshold → `config/settings.md`

Goal: confirm or adjust the scoring weights and threshold in `config/settings.md`.

Show the defaults:

- Experience match weight: **0.5**
- Network proximity weight: **0.3**
- Target company weight: **0.2**
- Draft-ready threshold: **70**

Ask: "These are the defaults. Want to keep them, or adjust?" — options: **Keep defaults (recommended)** / **Adjust weights** / **Adjust threshold** / **Adjust both**.

If the user adjusts weights, validate they sum to 1.0 (allow rounding to two decimals — e.g., 0.5 + 0.3 + 0.2 is fine; 0.4 + 0.4 + 0.3 is not). If the sum is off, show the math and ask again.

If the user adjusts threshold, validate it's an integer in [0, 100].

Update `config/settings.md` with the values.

---

## Stage 8 — Cost guardrails → `config/settings.md`

Goal: confirm or adjust the cost guardrails for company-career-page scanning in `config/settings.md`.

Show the defaults:

- `priority_filter`: **High** — only scan companies at this priority or above
- `max_pages_per_run`: **10** — cap total companies scanned per `/scan-jobs` run
- `cache_results_days`: **3** — don't re-scan a company within N days

Briefly explain: "Company career pages can be expensive (a lot of tokens), so these guardrails keep `/scan-jobs` cheap by default. You can raise them later in `config/settings.md`."

Ask: **Keep defaults (recommended)** / **Adjust**.

If the user adjusts:
- `priority_filter` ∈ { High, Medium, Low } (Low = scan everything)
- `max_pages_per_run` must be a positive integer
- `cache_results_days` must be a non-negative integer

Update `config/settings.md` with the final values.

---

## On completion

1. List the three files you wrote with their full paths.
2. Show a one-line summary of each key choice (e.g., "Roles: Product Manager, Program Manager, Project Manager / Threshold: 70 / Companies seeded: 15").
3. Tell the user: *"You're set. Run `/scan-jobs` whenever you want to find new postings."*

## If the user aborts mid-setup

Any file you've already written stays in place — that's intentional. Tell the user which files exist and which stages haven't been run yet. They can re-run `/setup` later; the existing-config detection at the top will pick up where they stopped.
