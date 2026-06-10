# Scoring methodology

Job-Seeker uses a **filter-then-score funnel**. Hard filters drop obviously irrelevant postings before any reasoning happens. Soft scores then rank what survives across three dimensions and combine them into a single weighted summary.

This file explains what the filters do, what the scores measure, and why the model is shaped this way.

---

## The funnel at a glance

```
raw postings  ──filters──▶  inbox  ──scoring──▶  ranked dashboard  ──threshold──▶  ⭐ jobs
   (everything)             (survivors)         (Exp / Net / Tgt / Sum)         (drafts eligible)
```

- **Filters are binary.** A posting either survives or gets dropped, with a one-line reason logged.
- **Scores are 0–100.** They're the agent's reasoned judgment, narrated in plain English in each per-job file — the number is a summary of the narrative, not a substitute for it.
- **The threshold** is a single integer (default `70`). Jobs at or above it earn a ⭐ on the dashboard and are eligible for `/draft-tailored-resume-coverletter`.

The whole funnel runs locally on your Mac. No data leaves the machine.

---

## Hard filters (applied during `/scan-jobs`)

A posting is dropped from the inbox entirely if it fails any of these. The skill logs each drop with a reason so the final summary can show the breakdown.

| Filter | Drop if... |
|---|---|
| **Title category** | Title doesn't reasonably read as Product / Program / Project Manager. Judgment call, not regex — "Product Marketing Manager" or "Engineering Manager" drops; "Sr. Program Manager, Trust & Safety" stays. |
| **Seniority** | Title indicates VP+, Head of, Chief, Associate, Junior, or Intern. **Director-level titles stay in** but get penalized at scoring time. |
| **Location** | Not Portland metro AND not US-based Remote AND not Hybrid-with-Portland-HQ. "Remote — Canada only" or "Remote — EU" drops. |
| **Compensation floor** | Comp is disclosed AND falls below `$80/hr` or `$110K annual`. **Undisclosed comp does NOT drop** — kept as `comp_floor_met: null`. |
| **Job type** | Doesn't match full-time, contract, or contract-to-hire. Part-time, internships, and temp-staffing-only drop. |

### Why these specific filters?

Job-board search filters are noisy. LinkedIn returns Director-of-Engineering postings under "Product Manager" keyword searches; Indeed returns Portland-tagged jobs that turn out to be Indianapolis. Spending tokens scoring obvious misfires is wasted work and clutters the dashboard.

The filter list above is the smallest set that drops 80%+ of the noise on a typical weekly scan without dropping any legitimate candidates. Each filter has a clear failure mode — when it drops something it shouldn't, the reason in the run summary makes it obvious which filter to relax.

---

## Soft scores (applied during `/score-jobs`)

Three sub-scores, each 0–100, combined into a weighted summary.

| Dimension | Default weight |
|---|---|
| Experience match | **0.5** |
| Network proximity | **0.3** |
| Target company | **0.2** |

```
summary_score = 0.5 × experience_match + 0.3 × network_proximity + 0.2 × target_company
```

Weights live in `config/settings.md` and you can change them. The skill checks they sum to 1.0 and warns if not, but won't silently normalize — you wrote a number for a reason.

### Experience match (0–100, default weight 0.5)

The substantive scoring work. The agent reads the JD and your base resume carefully and produces a sub-score for each component, narrated per posting in `jobs/scored/<id>.md`.

| Sub-component | What it measures |
|---|---|
| **Title / level alignment** | Does the title and implied seniority match your `criteria.md` target? Director-level for an IC-seeking user: −15 to −25 penalty. |
| **Years of experience** | Resume vs. JD's stated years requirement. Comfortably over → 100. |
| **Industry / domain** | Company's industry vs. your `criteria.md` Industries list. Exact match → 100. Adjacent → ~70. Off-target → 30–50. |
| **Skills / tools / methodologies** | Cross-reference JD requirements against the resume. Conspicuous gaps get called out in the narrative. |
| **Education / certifications** | Only when the JD names explicit requirements. If none stated, this doesn't lower the score. |

The agent **does not fabricate experience you don't have**. If the JD asks for something missing from your resume, it scores it as a gap and flags it. That gap also surfaces in `notes.md` when you draft the application later.

### Network proximity (0–100, default weight 0.3)

How well-positioned you are to get a warm intro into the role. Two tiers:

- **Tier 1** (cheap, automatic) — `/network-check` runs on every `/score-jobs` pass for every survivor, cached per company in `jobs/network-cache.json` for 30 days by default.
- **Tier 2** (expensive, manual) — `/research-network <rank>` for jobs you're actively pursuing. Visits up to 10 individual 2nd-degree profiles, reads their actual role and area, writes an outreach planning doc.

**Tier 1 base score** (from raw LinkedIn People-search counts):

| Signal | Base score |
|---|---|
| 1+ 1st-degree connections at the company | 70 |
| 5+ 2nd-degree connections | 50 |
| 1–4 2nd-degree connections | 30 |
| 0 known connections, but company on target list | 20 |
| 0 known connections, not on target list | 10 |

**Tier 1 bonus** (only when `config/connections.csv` is populated and rated):

- `+25` for each distinct **High**-rated 1st-degree who appears as a mutual on the search result cards
- `+10` for each distinct **Medium**-rated 1st-degree mutual
- **Low** or **Unrated** mutuals add no bonus

`final_score = min(100, base_score + sum_of_bonuses)`

The bonuses are where the rated `connections.csv` earns its keep. In the smoke run, every Nike mutual surfaced as `Unrated`, so the Tier 1 Nike score sat at base-only `70`. Rating just three frequent mutuals as `High` would lift it to `100` and re-rank the dashboard. The `Top mutuals` line in each per-job file calls this out by name so you know which ratings to fill in first.

If `connections.csv` doesn't exist yet (`/build-connections` hasn't been run), Tier 1 falls back to base score only and notes this in the per-job file. The score remains useful — just less discriminating.

### Target company (0–100, default weight 0.2)

A preference signal driven by `config/target_companies.csv`:

| Match | Score |
|---|---|
| On list, priority = **High** | 100 |
| On list, priority = **Medium** | 75 |
| On list, priority = **Low** | 50 |
| **Not** on list | 25 |

This dimension is intentionally low-weight. It says "you'd prefer to work here," not "you'd be successful here" — that's the experience-match dimension's job.

---

## The threshold

Default `70`. Jobs at or above earn a ⭐ in the dashboard and are eligible for `/draft-tailored-resume-coverletter <rank>`.

Tune it for your funnel:

- Too many ⭐ jobs to draft each week? **Raise to 75 or 80.**
- Inbox is small and you'd rather draft borderline-but-interesting jobs? **Lower to 65.**

The threshold lives in `config/settings.md` under `Threshold — Draft-ready threshold:`. Change it and the next `/score-jobs` (or `/rescore`) uses the new value.

---

## Why this scoring structure?

A few specific design choices worth calling out.

### Why filter-then-score, instead of scoring everything?

Three reasons:

1. **Cost.** Scoring a single posting costs a few thousand tokens of careful reasoning. Filtering it costs roughly zero. Filters drop the misfires before any expensive work happens.
2. **Signal quality.** A dashboard with 30 highly-relevant scored jobs is more useful than a dashboard with 100 jobs of which 30 are relevant. The user shouldn't have to mentally re-filter the agent's output.
3. **Auditability.** Each drop in the run summary names the filter that fired. If a real candidate ever gets dropped, the reason is one click away from being fixed.

### Why weight experience 0.5?

Experience is what gates the recruiter screen. Network proximity gates whether your application gets prioritized once it's in front of a hiring manager — important, but downstream of experience. Target company is a preference, not an outcome predictor.

This split also matters when the user has gaps in their network (`connections.csv` missing) or hasn't built their target list yet: experience-only scoring still produces a useful dashboard. The other two dimensions enrich the ranking without being prerequisites.

### Why is the dashboard sometimes thinner than it looks?

Two real-world caveats from the field-tested smoke run:

- **Promoted-by-hirer postings.** A growing share of LinkedIn postings render only a meta header on LinkedIn itself — the full JD lives on the hirer's external ATS (`Responses managed off LinkedIn`). When this happens, `/scan-jobs` writes the inbox file with an empty `jd_body` and a note pointing you at the apply link. Experience-match scoring is correspondingly thinner — the agent reasons from title, company, and metadata only. The per-job narrative says so honestly.
- **Network sub-scores can be `—`.** If `/network-check` can't reach LinkedIn during a run (Chrome not connected, rate-limited, CAPTCHA, fresh company never scanned and the network call was skipped to control cost), the per-job network score is `null` and the dashboard cell shows `—`. The summary uses a two-factor fallback (`experience + target` re-weighted) for that posting, and the dashboard note flags which jobs are affected so you can `/rescore` after fixing the issue.

Honest fallbacks beat confidently wrong scores. The whole point of `/score-jobs` is to **help you decide what to apply to** — a number you can't trust isn't a number worth printing.

---

## What lives where

- Per-job reasoning narrative: `jobs/scored/<id>.md`
- Ranked summary: `jobs/scored/dashboard-<date>.md`
- Tier 1 network cache: `jobs/network-cache.json`
- Tier 2 deep research outputs: `jobs/research/<id>-network.md`
- Weights, threshold, network TTL: `config/settings.md`
- Hard-filter definitions: this file + `.claude/skills/scan-jobs.md` (the source of truth)
- Sub-score formulas: `.claude/skills/network-check.md` (Tier 1), `.claude/skills/score-jobs.md` (combining and weighting)

If you want to tweak the model, every value lives in `config/settings.md`. The skills themselves don't hardcode weights, thresholds, or TTLs.
