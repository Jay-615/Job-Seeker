# Architecture

A short walk through how Job-Seeker is put together and the decisions behind the shape. For the user-facing how-to, see the [README](../README.md). For the scoring model, see [scoring-methodology.md](scoring-methodology.md). For why the tool stops where it stops, see [humans-in-the-loop.md](humans-in-the-loop.md).

---

## Runtime stack

| Component | Role |
|---|---|
| **Claude Code (CLI)** | The agent runtime. Reads `CLAUDE.md`, executes skills, edits and reads files on your filesystem. |
| **Claude in Chrome** (extension) | The browser controller. Operates LinkedIn (job search, network proximity, profile reads) and any company career page that needs JavaScript rendering — against your existing logged-in Chrome session. |
| **Pandoc** (local CLI) | Converts the markdown source for resumes and cover letters into DOCX for submission. Opt-in per draft. |
| **Git / GitHub** | Source distribution. Friends `git clone` the repo. Personal configuration stays on each user's Mac (gitignored). |

No server-side code. No central database. No accounts. Every byte of state — your criteria, target companies, connections, scored jobs, drafted applications — lives on your local filesystem.

---

## Repository shape

```
Job-Seeker/
├── CLAUDE.md                 # orients any Claude session
├── README.md                 # public-facing
├── .claude/skills/           # one file per slash command
├── config/                   # gitignored real files + committed .example.* templates
├── templates/                # resume, cover-letter, job-posting markdown skeletons
├── docs/                     # this folder — scoring, humans-in-the-loop, architecture
├── jobs/                     # GITIGNORED — inbox, scored, applied, passed, research, caches
└── applications/             # GITIGNORED — one folder per drafted application
```

The `.example.*` files in `config/` are committed and show a fictional persona (Sam Chen, a Seattle-based e-commerce PM). They serve two audiences: new users see exactly what to fill in, and portfolio viewers see the data model without needing to run anything.

The `.gitignore` keeps your real `config/*.md`, `config/*.csv`, `jobs/`, and `applications/` out of git automatically. Nothing personal ever gets pushed.

---

## Key architectural decisions

### 1. Claude Code workspace, not a custom Chrome extension or web app

A Chrome extension could automate parts of LinkedIn directly. A web app could host a hosted version of the tool. Both were considered and rejected:

- **Current paradigm.** A Claude Code workspace demonstrates the AI-native composition pattern hiring managers in 2026 actually care about — orchestrating an agent across files, browser, and shell, not building yet another single-purpose UI.
- **No bot-detection arms race.** Claude in Chrome operates your already-logged-in session at human pace. LinkedIn's anti-bot stack is built to defeat headless scrapers and automation patterns; a real browser with real cookies, paced like a human, doesn't trip those alarms.
- **Trivially debuggable.** All state is plain markdown and JSON on disk. When a score looks wrong, you read the per-job file. When a network number looks off, you read `jobs/network-cache.json`. Everything is inspectable with a text editor.
- **Modular by nature.** Each step is a skill. `/network-check` is testable standalone. `/score-jobs` calls it without duplicating browser logic. New skills slot in without touching the others.

### 2. Public repo, gitignored config

The repo is a portfolio artifact. Visibility matters — Friends-of-Phil Slack colleagues should be able to find it, and hiring managers should be able to browse the README, CLAUDE.md, and skill files as a work sample.

But job-search data is personal. The `.gitignore` strategy gives both:

- **Committed:** code, skills, templates, `.example.*` config files with a fictional persona, docs.
- **Local-only:** your real `criteria.md`, `target_companies.csv`, `connections.csv`, `settings.md`, your base resume (path-referenced, lives outside the repo), every file under `jobs/` and `applications/`.

The convention is documented in `config/README.md` so non-engineers cloning the repo understand what's safe to edit and what's safe to ignore.

### 3. Human-in-the-loop checkpoints between scan, score, and draft

The workflow could chain `/scan-jobs → /score-jobs → /draft-tailored-resume-coverletter` automatically. It doesn't. Each command is invoked manually, and each one stops with the user before the next.

The reasoning is documented in [humans-in-the-loop.md](humans-in-the-loop.md). Short version: the value of the tool is in **what the human does between agent runs** — reviewing scored postings, rating connections, editing tailored drafts. Skipping past those moments would optimize the metric (time-to-applied) by destroying the substance (a thoughtful application).

### 4. Filter-then-score, not score-everything

`/scan-jobs` applies hard filters (title category, seniority, location, comp floor, job type) and drops obvious misfires before any scoring happens. `/score-jobs` then reasons carefully about the survivors.

Two payoffs:

- **Cost.** Scoring a posting takes a few thousand tokens of careful reasoning. Filtering one takes roughly zero. Filters drop the misfires before any expensive reasoning starts.
- **Signal quality.** A dashboard with 30 highly-relevant scored jobs is more useful than one with 100 jobs of which 30 are relevant.

This is a classic operations funnel pattern. The README and `docs/scoring-methodology.md` both surface it because it's worth understanding before you tune your `criteria.md`.

### 5. Markdown as source of truth for resumes and cover letters

The tailored draft is written in markdown first. DOCX is generated from it via pandoc, opt-in per draft.

Why markdown:

- **Easy to edit by hand.** Most drafts get final-pass edits from the user. Markdown is friendlier to that loop than DOCX.
- **Easy for the agent to manipulate.** Reordering bullets, tightening phrasing, swapping in JD-relevant emphasis is a plain text transformation, not a docx-XML mutation.
- **Version-controllable.** Your application folder is a small git-able repo of its own if you want it to be.

Why DOCX is opt-in:

- Most applications go through several iteration rounds on the markdown source before the user actually submits. Generating DOCX every pass wastes pandoc invocations and clutters the folder with stale binaries.
- When you're ready to submit, re-run the draft skill and pick the DOCX option, or run pandoc directly. The skill tells you exactly how.

### 6. Two-tier network proximity (cheap default + expensive on-demand)

`/network-check` is the cheap, automatic Tier 1 — one People-search load per company, cached for 30 days. Runs on every `/score-jobs` invocation. Produces the network sub-score that goes into the weighted summary.

`/research-network <rank>` is the expensive, manual Tier 2 — visits up to 10 individual 2nd-degree profiles, reads each person's actual role, judges area relevance honestly. Produces an outreach planning document for jobs you're actively pursuing.

The split matters because the cost gradient is steep. Tier 1 is ~5–15K tokens per company. Tier 2 is ~5–15K tokens **per profile** × 10 profiles. Running Tier 2 for the whole inbox would burn budget on jobs the user isn't pursuing. Running Tier 1 only would miss the substantive area-fit reasoning that decides which intro to ask for.

The cache in `jobs/network-cache.json` is what makes the weekly workflow affordable. Once Nike's been Tier-1-scanned, the next 4 Nike postings on the dashboard score for free.

### 7. `connections.csv` as a separate, user-rated data source

The Tier 1 network score has two parts: a base score from raw 1st-/2nd-degree counts, and a bonus from explicitly-rated connections (`+25` per High mutual, `+10` per Medium mutual). The ratings live in `config/connections.csv`.

Two skills feed this:

- `/build-connections` does a one-shot bulk pull of all your 1st-degrees from LinkedIn into the CSV, defaulting them all to `Unrated`.
- `/rate-connections` walks you through Unrated rows one at a time. Or you skip the skill entirely and bulk-edit the CSV in Excel — both paths are first-class and the skills say so.

The rating data is the single highest-leverage piece of personal configuration in the tool. In the smoke run, every Nike mutual was `Unrated`, so the Nike network score sat at base-only `70`. Rating just three frequent mutuals as `High` would have lifted it to `100` and re-ranked the dashboard. The per-job narratives surface this so the user knows which ratings to fill in first.

### 8. Loud failure beats quiet wrongness

This is a posture, not a feature — but it's in every skill file. When LinkedIn changes its UI, when a CAPTCHA appears, when a network call returns 0 results that should plausibly have returned dozens, every skill is written to **stop and tell the user honestly** rather than guess.

A confidently wrong score wastes the user's time on a bad application. An honest "I couldn't reach LinkedIn for this company — the network sub-score is `—` for this run; `/rescore` after reconnecting" wastes 30 seconds and is recoverable.

Each browser-driven skill has a "LinkedIn UI dependencies" block at the bottom listing exactly what page-structure assumptions it carries, so a future maintainer can update those assumptions in one place when LinkedIn changes things. (Several of those blocks were updated as part of M7's polish pass, after a field-tested smoke run exposed parser edges the original draft missed.)

---

## Cost and performance, briefly

The most expensive operations, descending order:

1. **Company career page scans** — 5K–50K tokens per company. Controlled by `priority_filter`, `max_pages_per_run`, and `cache_results_days` in `settings.md`.
2. **LinkedIn job search results pages** — moderate, predictable. One page per role × location pair per source.
3. **Network proximity lookups** — ~5–15K tokens per company. Cached 30 days.
4. **Experience scoring** — a few thousand tokens per posting. Text-only, no browser.
5. **Tailored draft generation** — ~5–10K tokens per draft. Text-only, no browser.

End-to-end weekly run estimate (for a Pro-tier user with ~20 inbox postings and ~10 unique companies needing network scans): **~200K–500K tokens**. The defaults in `settings.example.md` keep this comfortable on Pro. Max users have headroom to raise `max_pages_per_run` and `research_profile_cap`.

The README is honest about subscription tier vs. realistic usage so users can choose with eyes open.

---

## What's next

The skills are stable and the smoke run validated the end-to-end pipeline. The natural next directions, if the tool gets real weekly use:

- **`/mark-applied` and `/mark-passed` implementations.** Stubbed today; should be straightforward — move the posting file, update `seen.json`, optionally update `target_companies.csv` status.
- **Application tracking summary.** A `/status` skill that reads `jobs/applied/` and shows the user a clean view of where each application sits.
- **Cross-company network reachability index.** A periodic computation that builds a `connections × companies` matrix from cached Tier 1 data — answers "given who I know, which target companies am I best-positioned to break into?"
- **Optional Apply-link JD recovery.** For the "Promoted by hirer · Responses managed off LinkedIn" pattern, follow the Apply button's `href` to the hirer's ATS and try to extract the JD there. Currently flagged in the inbox file as "not on LinkedIn page."

None of these were in M0–M9 scope. They're the obvious follow-ons after a few weeks of real use surfaces which next-step is most worth building.
