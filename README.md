# Job-Seeker

A personal job-search workflow built as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) workspace. You clone the repo, run `claude` in the folder, walk through a one-time setup, and then twice a week run a small set of slash commands to find new postings, score them on three dimensions, generate tailored resumes and cover letters for the best matches, and track outcomes.

It's a tool, not a service. Everything runs locally on your Mac. Nothing about your job search ever leaves the machine.

> **Demo video:** *Coming soon — TODO: record a 3-minute walk-through after the first real weekly run.*

---

## What this is

Job-Seeker turns a hand-driven job search into a partnered one. You stay in the loop for every judgment call. Claude does the slow, systematic work — scanning four sources, applying filters, scoring postings against your resume and your network, drafting tailored applications, and tracking what you've already seen.

The headline weekly flow:

```
/scan-jobs    # find new postings, filter, dedupe
/score-jobs   # rank the inbox on three dimensions
(read)        # review the dashboard, dig into the ⭐ jobs
/research-network <rank>           # deep network research on a job you're pursuing
/draft-tailored-resume-coverletter <rank>   # tailored resume + cover letter
(read and edit)                    # iterate on the markdown
/mark-applied <rank>               # or /mark-passed
```

That's the whole product. The rest of this README explains what each step does, why it was built this way, and how to set it up.

---

## Who this is for

Two audiences, by design.

**You're an ex-Nike colleague (or anyone job-searching).** You want a tool that actually helps you find and apply to jobs without grinding through search results manually. You don't want a tool that auto-submits applications on your behalf — that's not where the value is, and it's the part you wouldn't trust an agent with anyway. Setup is ~10 minutes; the weekly run is 30–60 minutes of focused work. See the [10-minute setup](#10-minute-setup) and [Weekly usage](#weekly-usage) sections.

**You're a PM, PgM, or hiring manager browsing the repo as a work sample.** The interesting reading is:

- [`docs/humans-in-the-loop.md`](docs/humans-in-the-loop.md) — the PM choices about where the agent stops and the human shows up
- [`docs/architecture.md`](docs/architecture.md) — the runtime stack and the eight architectural decisions behind the shape
- [`docs/scoring-methodology.md`](docs/scoring-methodology.md) — the filter-then-score funnel, the three sub-scores, and why
- [`.claude/skills/`](.claude/skills/) — the actual skill files; each one is its own short product spec
- [`docs/sample-network-research.md`](docs/sample-network-research.md) — a representative `/research-network` output, anonymized

I'm Jay Zambelli, 11 years at Nike on bot mitigation, identity, and digital ops. Laid off April 2026; built this in the weeks after. The tool was simultaneously my actual job-search workflow and a deliberate work sample. If you'd like to talk, my LinkedIn is in the cover letter of any application this tool produced.

---

## Prerequisites

| Component | What you need | Why |
|---|---|---|
| **macOS** | Recent macOS (Apple Silicon or Intel). | Everything is local; the skills assume a Unix-y shell. |
| **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** | Installed and signed in. | The agent runtime — reads `CLAUDE.md`, runs the skills, edits files. |
| **[Claude in Chrome](https://www.anthropic.com/claude-in-chrome)** extension | Installed and connected, with you logged into LinkedIn in that same Chrome. | Operates LinkedIn (job search, network proximity, profile reads) on your behalf, against your own logged-in session at human pace. |
| **Claude Pro or Max subscription** | Pro works for occasional use. Max is recommended for full weekly runs. | A weekly end-to-end run is roughly 200K–500K tokens. Pro is fine if you tighten the cost guardrails in `settings.md`; Max gives material headroom. |
| **Pandoc** | `brew install pandoc` (or update the path in `settings.md`). | Converts the markdown resume + cover letter source into the DOCX most employers expect. Opt-in per draft. |
| **Git** | Already on macOS. | For cloning the repo and (optionally) keeping your own application materials version-controlled locally. |

That's it. No accounts to make, no servers to configure, no APIs to enable.

---

## 10-minute setup

1. **Clone the repo.**

   ```sh
   git clone https://github.com/<your-or-jay's-fork>/Job-Seeker.git ~/job-seeker
   cd ~/job-seeker
   ```

2. **Launch Claude Code in the folder.**

   ```sh
   claude
   ```

3. **Run the guided setup.**

   ```
   /setup
   ```

   The `/setup` skill interviews you through eight short stages:

   1. **Browser connection check** — confirms Claude in Chrome is installed and connected. If not, it pauses with an install link.
   2. **LinkedIn login check** — asks you to confirm you're logged into LinkedIn in that Chrome.
   3. **Contact info** — your name, email, phone, and city. Written to `config/settings.md` immediately so you don't lose it if anything interrupts.
   4. **Base resume path** — the absolute path to your existing resume file on your Mac (markdown, DOCX, or PDF). The tool reads this when tailoring drafts; you keep editing the source yourself.
   5. **Job criteria** — roles, locations, seniority, compensation floor, job types, industries. Written to `config/criteria.md`.
   6. **Target companies** — either point at an existing CSV you have, seed from `target_companies.example.csv`, or start empty.
   7. **Scoring weights and threshold** — defaults are `0.5 / 0.3 / 0.2` and threshold `70`; you can adjust.
   8. **Cost guardrails** — defaults are `priority_filter: High`, `max_pages_per_run: 10`, `cache_results_days: 3`. The skill explains each one.

   `/setup` is re-runnable any time — it detects existing config and asks before overwriting.

4. **(Recommended) Build your connections file.**

   ```
   /build-connections
   ```

   This is a one-shot bulk pull of your 1st-degree LinkedIn connections into `config/connections.csv`, all defaulted to `Unrated`. Expect 3–5 minutes for a ~500-connection network. The point of this step is to enable the network-proximity *bonus* score later: when LinkedIn surfaces your 1st-degree connections as mutuals on a target-company page, the rated ones (`High` or `Medium`) add bonus points.

   You don't have to rate everyone. Run `/rate-connections` for an interactive walk-through, or open `config/connections.csv` in Excel or Numbers and bulk-edit the `rating` column — both paths are first-class.

5. **You're ready.** Run `/scan-jobs` whenever you want to find new postings.

---

## Weekly usage

The expected pattern: two ~30–60 minute sessions a week. Both follow the same shape.

### Step 1: Find new postings

```
/scan-jobs
```

The tool scans LinkedIn, Indeed, Dice, and your priority-High company career pages (per your `settings.md` toggles), applies the hard filters from [`docs/scoring-methodology.md`](docs/scoring-methodology.md), dedupes against `jobs/seen.json`, and writes one markdown file per new posting to `jobs/inbox/`.

If Indeed pushes a CAPTCHA, the skill pauses and asks via `AskUserQuestion` whether to clear it manually (a 30-second flip to your Chrome window) or skip the source. The same pattern handles "log in to Dice for better results" style soft suggestions.

Sample summary at the end of a run:

```
Scanned 4 sources.
- LinkedIn: 31 postings → 18 after filters
- Indeed: 12 → 7
- Dice: 4 → 1
- Career pages (8 priority-High companies): 6 → 4

30 total survivors. 12 are dedupes from prior runs — skipped.
18 new postings written to jobs/inbox/.

Filter drops: 8 too senior (VP+), 4 wrong location, 2 too junior, 11 dedupes.

Ready when you are: /score-jobs
```

### Step 2: Score the inbox

```
/score-jobs
```

For every posting in `jobs/inbox/`, the agent computes three sub-scores and a weighted summary:

- **Experience match (50% weight)** — reads the JD against your base resume, scores title/level alignment, years, industry, skills, and education, and writes a per-sub-component narrative.
- **Network proximity (30% weight)** — uses `/network-check` to count 1st- and 2nd-degree connections at the company, cross-references your `connections.csv` ratings for bonuses (High = +25, Medium = +10 per distinct mutual). Cached 30 days per company.
- **Target company alignment (20% weight)** — looks up the company in `target_companies.csv` (High = 100, Medium = 75, Low = 50, not on list = 25).

The output is a ranked dashboard at `jobs/scored/dashboard-<date>.md` plus one detailed file per posting at `jobs/scored/<id>.md`. The dashboard renders like this (real example from the smoke run):

```
# Scored Jobs — 2026-06-09

5 postings scored. Threshold: 70. Weights: Experience 0.5 / Network 0.3 / Target 0.2.

| Rank | Company   | Title                          | Type      | Location      | Mode   | Comp                  | Exp | Net | Tgt | Sum | Status |
|------|-----------|--------------------------------|-----------|---------------|--------|-----------------------|-----|-----|-----|-----|--------|
| 1    | Nike      | Product Manager                | contract  | Beaverton, OR | Hybrid | —                     | 85  | 70  | 100 | 84  | ⭐     |
| 2    | Fabletics | Sr. Product Manager II         | full-time | Remote (US)   | Remote | —                     | 80  | —   | 25  | 64  |        |
| 3    | Instagram | Product Manager                | full-time | Remote (US)   | Remote | $146K/yr - $204K/yr   | 70  | —   | 25  | 57  |        |
| 4    | Stitch Fix| Product Manager - Fix Exp.     | full-time | Remote (US)   | Remote | —                     | 65  | —   | 25  | 54  |        |
| 5    | Roadie    | Product Manager                | full-time | Remote (US)   | Remote | —                     | 55  | —   | 25  | 46  |        |
```

The ⭐ rows are at or above your threshold and are eligible for tailored drafting. The `—` cells under `Net` mean network scoring was skipped or failed for that company on this run (the per-job file explains why; `/rescore` or a manual `/network-check <company>` fills them in).

### Step 3: Read the per-job reasoning

For any posting you're curious about, open `jobs/scored/<id>.md`. The dashboard summary score is the headline; the narrative paragraphs are the substance. From the smoke run's Nike entry:

> **Title/level:** 80 — Posted title is "Product Manager" without a seniority qualifier. Jay's `criteria.md` targets Senior IC (Senior PM, Lead PM, Principal PM). A bare "Product Manager" title at Nike could be mid-level IC, which is below target. Slight downgrade applied. Without the JD body it's not possible to read the actual level — the JD likely clarifies in the responsibilities section, but the off-LinkedIn promoted posting hides it.
>
> **Skills/tools:** 85 — Limited by the missing JD body. Based on Jay's track record (Kasada/Akamai vendor work, ML model launches, GDPR/consent, passwordless/passkeys), broad fit is strong. Specific tooling can't be verified without the JD.

That's the agent talking. It's honest about gaps and limits ("limited by the missing JD body") rather than papering over them with a confident-looking score.

### Step 4: (Optional) Deep network research for a job you're pursuing

```
/research-network 1
```

For a specific ⭐ job, this Tier 2 command visits up to 10 individual 2nd-degree profiles at the target company, reads each person's actual role and area, judges relevance to the specific posting (Direct match / Adjacent / Off-area / Unclear), and writes a polished outreach planning doc at `jobs/research/<id>-network.md`.

The output is what you take to LinkedIn. See [`docs/sample-network-research.md`](docs/sample-network-research.md) for a representative (anonymized) version — including the prioritized Top 3 outreach plan with draft opening lines for each ask.

The skill is expensive (~5–15K tokens per profile × 10 profiles) and *manual* by design. Don't run it across the whole inbox. Run it on the one or two jobs you're seriously considering.

### Step 5: Draft a tailored application

```
/draft-tailored-resume-coverletter 1
```

For a ⭐ rank (or job id), the skill reads the scored file, the inbox file, and your base resume, then writes three files to `applications/<company>-<role>-<date>/`:

- `resume.md` — your base resume, tailored: bullets reordered to lead with what the JD asks for, phrasing tightened, off-topic bullets dropped. **Nothing fabricated.** If the JD asks for something your resume doesn't show, it goes into `notes.md`, not into the resume.
- `cover-letter.md` — three substantive paragraphs, ~300–400 words, written in your voice from the base resume — opening hook → evidence → close.
- `notes.md` — the agent's honesty file: highlights, gaps the agent declined to paper over, suggestions for what *you* might add manually, and an explicit "what I did NOT do" section.

DOCX is opt-in. By default the skill writes the three markdown files, then asks via `AskUserQuestion` whether to also generate DOCX via pandoc, defaulting to "markdown only — iterate first, generate DOCX when you're ready to submit." When you're ready, re-run the skill and choose the DOCX option.

### Step 6: Submit, then record the outcome

You submit the application yourself, in whatever portal the company uses. Then:

```
/mark-applied 1     # or /mark-passed 1
```

This moves the scored file into `jobs/applied/` or `jobs/passed/` and updates `seen.json` so the posting never resurfaces in a future scan.

---

## How it works

A short tour of the runtime and the directory layout. For depth, read [`docs/architecture.md`](docs/architecture.md).

### Runtime stack

- **Claude Code** is the agent runtime. It reads `CLAUDE.md`, executes the skills in `.claude/skills/`, and edits files in your local repo.
- **Claude in Chrome** is the browser controller. When a skill needs to read LinkedIn (job search, company People page, individual profiles), it drives your already-logged-in Chrome session at human pace.
- **Pandoc** runs locally to convert markdown to DOCX when you opt into it.
- **Git** distributes the repo. Personal data stays out via `.gitignore`.

There is no server, no shared database, no SaaS layer. Everything you produce is plain markdown or CSV on your filesystem.

### Repository shape

```
Job-Seeker/
├── CLAUDE.md                            # orients any Claude session
├── README.md                            # this file
├── .claude/skills/                      # one file per slash command
│   ├── setup.md, scan-jobs.md, score-jobs.md, network-check.md,
│   ├── build-connections.md, rate-connections.md,
│   ├── draft-tailored-resume-coverletter.md, research-network.md,
│   └── mark-applied.md, mark-passed.md, rescore.md
├── config/                              # personal config — gitignored except .example.*
│   ├── criteria.example.md, target_companies.example.csv,
│   ├── connections.example.csv, settings.example.md  (committed)
│   └── criteria.md, target_companies.csv,
│       connections.csv, settings.md  (gitignored, real)
├── templates/                           # resume, cover-letter, job-posting skeletons
├── docs/                                # scoring methodology, humans-in-the-loop, architecture, sample research
├── jobs/                                # GITIGNORED — inbox / scored / applied / passed / research / caches
└── applications/                        # GITIGNORED — one folder per drafted application
```

### What's in each skill file

Each `.claude/skills/*.md` file is a self-contained spec for one slash command — the steps, the failure modes, and a "LinkedIn UI dependencies" block at the bottom (for the browser-driven ones) that says exactly what page-structure assumptions it carries. Updating any one skill shouldn't require touching the others; the skills depend on each other only through the file artifacts and the `Skill` tool.

---

## Where humans stay in the loop

Read [`docs/humans-in-the-loop.md`](docs/humans-in-the-loop.md) for the full version. Short summary:

- **No auto-apply.** The tool stops at "tailored draft ready." Application portals are heterogeneous and fragile; cover letters benefit from a final-pass human edit; the submit moment is the right time to re-read the materials. Auto-submitting would skip the loop where the quality lives.
- **No scheduling.** Every command is manually triggered. No cron, no background runs. The dominant 2026 paradigm for AI in knowledge work is **partner, not autonomous worker**, and a job search is a few hours per week of judgment-intensive work — not an inbox that needs draining overnight.
- **No SaaS, no central database.** Your job-search data is sensitive. It stays on your Mac.

The CAPTCHA-clearing flow in `/scan-jobs`, the `notes.md` file in `/draft-tailored-resume-coverletter`, and the "Direct match / Adjacent / Off-area / Unclear" verdicts in `/research-network` are all places where the agent deliberately stops short and surfaces a decision to the human.

---

## Scoring methodology

For the full version, read [`docs/scoring-methodology.md`](docs/scoring-methodology.md).

Job-Seeker uses a **filter-then-score funnel**:

1. **Hard filters** (during `/scan-jobs`) drop obvious misfires — wrong title category, wrong seniority, wrong location, comp below floor, wrong job type. Each drop is logged with a reason.
2. **Soft scores** (during `/score-jobs`) rank the survivors across three dimensions:
   - Experience match (default weight 0.5)
   - Network proximity (default weight 0.3)
   - Target company (default weight 0.2)
3. **Threshold** (default 70) flags jobs eligible for tailored drafting.

The weights, threshold, and network cache TTL all live in `config/settings.md` — change them and the next run picks up the new values.

---

## FAQ and troubleshooting

### Pro or Max?

A full end-to-end weekly run (scan + score + a couple of research calls + a draft) lands around **200K–500K input tokens**. Pro works if you keep `max_pages_per_run` modest and don't run `/research-network` against every ⭐ job. Max gives material headroom — recommended if you're running the full pipeline weekly. The README is honest about this; nobody likes surprise rate limits mid-week.

### Indeed pushed a CAPTCHA mid-run. What now?

`/scan-jobs` pauses on the affected source and asks via `AskUserQuestion` whether to clear it now, skip the source for this run, or stop the whole run. The "clear it now" path is a 30-second flip to your Chrome window — once you've cleared it (and ideally logged in, which lowers future CAPTCHA risk), the skill retries once and continues. LinkedIn and Dice CAPTCHAs are handled the same way.

### Why are the network sub-score cells showing `—`?

The network sub-score requires a fresh `/network-check` call or a cache hit in `jobs/network-cache.json`. If neither is available (Chrome wasn't connected, the company hasn't been scanned yet, or the call was skipped for cost reasons), the per-job file says so and the dashboard shows `—`. The summary score for that posting uses a two-factor fallback (experience + target re-weighted). Re-run `/score-jobs` or `/rescore` once the underlying issue is fixed.

### My `connections.csv` is full of `Unrated` rows. Does that matter?

The Tier 1 network score works without it (base score only). The *bonus* score is the part that adds discriminating signal — High mutuals are worth +25, Medium are worth +10. If you don't want to walk through `/rate-connections`, open the CSV in Excel or Numbers, sort by `headline` (which groups people by company), and bulk-rate. Both paths produce the same result.

### A LinkedIn posting in my inbox has an empty Job Description and a note saying "Promoted by hirer · Responses managed off LinkedIn." What gives?

This is a real LinkedIn pattern — the hirer paid for promotion but the actual JD lives on their external ATS rather than on the LinkedIn page itself. `/scan-jobs` detects this and writes the inbox file with an empty `jd_body`, a frontmatter note, and a `## Notes` line pointing you at the apply link. Experience-match scoring is thinner without the JD body; the per-job narrative says so honestly. Follow the URL and click Apply to read the real JD.

### Can I run this on Windows or Linux?

The pieces are mostly portable (Claude Code, Claude in Chrome, pandoc, git all run on the major OSes), but the skills currently assume macOS-style absolute paths (`/Users/...`) and shell. Porting is straightforward; nobody's done it yet.

### I changed `criteria.md` (or weights, or `connections.csv` ratings). Do I need to re-scan?

No — just run `/rescore`. It re-scores everything currently in `jobs/inbox/` against the new config without touching the network cache (or re-fetching anything via Chrome). For new postings, run `/scan-jobs` again.

### How do I update Job-Seeker when there's a new version?

`git pull` in your local clone. Your `config/`, `jobs/`, and `applications/` directories are gitignored, so nothing personal gets touched. Skills, templates, and docs update in place.

### Does this tool send anything to a server somewhere?

No. The only network traffic is:

- Claude Code's normal usage of Anthropic's API (your prompts and Claude's responses).
- Claude in Chrome navigating LinkedIn, Indeed, Dice, and company career pages — as you, with your cookies, in your Chrome.
- Pandoc, when invoked, running purely locally.

There is no Job-Seeker server. There is no telemetry. Your config, your scored postings, your tailored drafts, and your connections list never leave your machine.

### A skill broke because LinkedIn changed its UI. What do I do?

Each browser-driven skill ends with a "LinkedIn UI dependencies" block listing exactly what page-structure assumptions it carries. The fix is usually a small selector update or a regex tweak. If you're comfortable editing markdown and a bit of JavaScript, the block tells you where to look. If you're not, open an issue (or ask Claude to walk through the affected skill file with you).

---

## License

MIT. See [`LICENSE`](LICENSE).

---

## Acknowledgements

- **Anthropic** — for Claude Code and Claude in Chrome, the two pieces of infrastructure this tool is built on.
- **The Friends-of-Phil Slack** — for being the audience whose existence made me build a real, polished tool instead of a one-off script.
- **The 11 years of teammates at Nike** — whose names and faces are mostly the reason the network-proximity dimension is worth scoring at all.

— Jay
