---
name: score-jobs
description: Score every posting in jobs/inbox/ on three dimensions — experience match (50%), network proximity (30%, via /network-check), and target-company alignment (20%). Writes per-job scored files to jobs/scored/ and a ranked dashboard table to jobs/scored/dashboard-<date>.md. Reads weights and threshold from config/settings.md.
---

# /score-jobs — rank the inbox

You read every posting in `jobs/inbox/`, compute three sub-scores plus a weighted summary, write one scored file per posting to `jobs/scored/<id>.md`, and produce a ranked dashboard table at `jobs/scored/dashboard-<today>.md`.

Hard filters were already applied in `/scan-jobs`. **Do not re-filter here** — every file in `jobs/inbox/` is a survivor that deserves a score.

The skill takes no parameters. It scores the whole inbox.

---

## Step 0 — Preflight

1. Read `config/settings.md`. Parse:
   - **Scoring Weights** — three floats that should sum to 1.0. Defaults if unparseable: `experience_match: 0.5`, `network_proximity: 0.3`, `target_company: 0.2`.
   - **Threshold — Draft-ready threshold** — integer. Default 70.
   - **User Profile — Base resume path** — absolute path to the user's resume file.

   If the weights don't sum to ~1.0 (within ±0.01), warn the user in the console summary but proceed with what's in the file.

2. Read `config/criteria.md`. Pull the **Roles** list, **Seniority** rules (so you know what "Director-level penalty" looks like for *this* user — Jay's criteria say Senior IC is the target, so Director is penalized), and the **Industries** list (used as a soft signal for the experience-match industry sub-component).

3. Read `config/target_companies.csv` into memory (header row + every data row). You'll use this for the target-company score and to pass company names to `/network-check`.

4. Read the user's base resume from the path in `settings.md`. Cache the full text once. **Do not re-read it for every job.** If the file doesn't exist or isn't readable, bail with:

   > Couldn't read base resume at `<path>`. Update `Base resume path:` in `config/settings.md` and re-run.

5. List `jobs/inbox/*.md`. If empty, tell the user:

   > Inbox is empty. Run `/scan-jobs` first to find postings, then come back.

   …and stop. If non-empty, count and tell the user how many you're about to score.

6. Set `today_iso = YYYY-MM-DD` for `scored_date` and the dashboard filename.

---

## Step 1 — For each inbox posting, compute three sub-scores

Iterate the inbox files in any order. For each file:

1. Read the file. Parse the frontmatter (`id`, `company`, `title`, `location`, `mode`, `type`, `comp_text`, `url`, etc.) and the body (`## Job Description`, `## Key Requirements`, etc.).

2. Compute the three sub-scores below.

3. Compute the weighted summary.

4. Write the per-job scored file (Step 2) and record one row for the dashboard (Step 3).

### 1a. Experience match (0–100)

This is the substantive reasoning work. Read the JD and the cached base resume carefully. Compute sub-component scores, then combine.

Sub-components (use judgment on internal weighting — the spec says implementer's call):

| Sub-component | What to look at |
|---|---|
| **Title / level alignment** | Does the title and implied seniority match the user's targeted seniority in `criteria.md`? **If the user targets IC-level and the posting is Director-level**, apply a penalty of **−15 to −25 points** on this sub-component. This is explicit in the spec. Mention the penalty in the narrative. |
| **Years of experience** | Does the user's resume show ≥ the JD's stated years requirement? Score 100 if comfortably over, scale down if under. |
| **Industry / domain** | Does the company's industry match an industry in the user's `criteria.md` Industries list? Exact match → 100. Adjacent → 70. Off-target → 30–50. |
| **Skills / tools / methodologies** | Cross-reference JD requirements against the resume. Score by coverage. Note any conspicuous gaps in the narrative (the user wants to see them). |
| **Education / certifications** | Only if the JD names explicit requirements. If none stated, this sub-component doesn't lower the score. |

Combine into a single 0–100 number. Round to an integer. Write a short reasoning paragraph **per sub-component** in the per-job file (see Step 2 for the structure).

**Don't fabricate experience the user doesn't have.** If the JD asks for something the resume doesn't show, score it as a gap and call it out in the narrative. This is the same principle as `/draft-tailored-resume-coverletter`.

### 1b. Network proximity (0–100)

Delegate to `/network-check`. Do not duplicate any browser logic here.

Invoke via the `Skill` tool: `Skill(skill="network-check", args="<company name>")`. Pass the company name exactly as it appears in the inbox file's frontmatter `company:` field (quote multi-word names: `"Columbia Sportswear"`).

`/network-check` returns the structured JSON shape documented in `.claude/skills/network-check.md` Step 7. From that result, use:

- `final_score` → the **network proximity sub-score** (already 0–100, already capped)
- `first_degree_count`, `second_degree_count` → for the per-job narrative
- `first_degree_mutuals_top` → name + rating + appearance count, for the narrative
- `warmest_paths` → list of names for the narrative
- `data_source` → so you can mention "cache hit" vs "fresh pull" in the narrative
- `reasoning` → fold into your own per-job narrative

**If `/network-check` fails** (Chrome not connected, rate limit, parse error, etc.), don't bail the whole scoring run. Record `network_proximity: null` for that job, note the failure in the narrative, and use a **two-factor weighted summary** for that job:

```
summary = (exp_weight * experience + tgt_weight * target) / (exp_weight + tgt_weight)
```

Flag the missing sub-score in the dashboard row's narrative column ("Net: —"). The user can re-score later with `/rescore`.

**Token-cost note:** `/network-check` is cached per company in `jobs/network-cache.json` with the TTL from `settings.md`. Multiple postings at the same company on the same run will hit the cache after the first call. You don't need to deduplicate company calls yourself — the cache handles it.

### 1c. Target-company alignment (0–100)

Look up the posting's `company` in the in-memory `target_companies.csv`. Case-insensitive, trim whitespace.

| Match | Score |
|---|---|
| On list, priority = `High` | 100 |
| On list, priority = `Medium` | 75 |
| On list, priority = `Low` | 50 |
| **Not** on list | 25 |

Mention the match (or lack thereof) and the company's `status` column in the narrative when relevant ("on target list, priority High, status Researching").

### 1d. Weighted summary

```
summary = round(
    weights.experience_match * experience_score +
    weights.network_proximity * network_score +
    weights.target_company * target_score
)
```

If `network_score is null`, use the two-factor fallback formula from §1b.

`above_threshold = summary >= threshold`.

---

## Step 2 — Write the per-job scored file

For each posting, write `jobs/scored/<id>.md` where `<id>` is exactly the `id:` value from the inbox file's frontmatter. Use this structure:

```markdown
---
id: {{id from inbox frontmatter}}
scored_date: {{today_iso}}
experience_match: {{0-100 int}}
network_proximity: {{0-100 int, or null if /network-check failed}}
target_company: {{0-100 int}}
summary_score: {{0-100 int}}
above_threshold: {{true | false}}
---

## Experience Match ({{experience_match}}/100)

**Title/level:** {{sub-score}} — {{one-to-three sentence narrative. If Director-level penalty applied, state the penalty amount.}}

**Years:** {{sub-score}} — {{narrative — JD asks X, resume shows Y.}}

**Industry/domain:** {{sub-score}} — {{narrative — match against criteria.md Industries.}}

**Skills/tools:** {{sub-score}} — {{narrative — coverage and conspicuous gaps.}}

**Education/certs:** {{sub-score, or "n/a" if JD doesn't state requirements}} — {{narrative if applicable.}}

## Network Proximity ({{network_proximity}}/100)

- 1st-degree connections at {{company}}: {{first_degree_count}}
- 2nd-degree connections at {{company}}: {{second_degree_count}}
- Top mutuals: {{name (rating, ×count), name (rating, ×count), ...}} (or "none surfaced")
- Warmest intro paths: {{comma-separated names}} (or "none — `connections.csv` not yet populated" if applicable)
- Data source: {{cache | fresh | fresh (connections.csv missing — base score only)}}

{{One-paragraph narrative folding in /network-check's reasoning.}}

## Target Company ({{target_company}}/100)

- {{company}} {{is | is NOT}} on `target_companies.csv`.
- {{If on list:}} Priority: {{High | Medium | Low}}. Status: {{Researching | Active | Applied | Closed | Not interested}}.

## Summary

**Weighted: {{summary_score}}/100 — {{Above threshold ({{threshold}}). | Below threshold ({{threshold}}).}}** {{If above: "Recommended for application drafting." If below: "Below threshold — review before pursuing."}}

Weights used: experience {{exp_weight}}, network {{net_weight}}, target {{tgt_weight}}.
```

Notes on the narrative:

- The per-job file is the substantive artifact — `summary_score` is just a number, but the **reasoning is what the user reads to decide whether to apply**. Write narratives that explain the *why*, not just the number. The Section 5 example in the project spec is the target quality bar.
- If the inbox file has a `## Notes` section worth flagging (e.g., "JD mentions PCI compliance — not in your resume"), surface it in the **Skills/tools** sub-component narrative.
- Don't include the full JD or full resume in the scored file — link by id, narrate concisely.

---

## Step 3 — Build the dashboard table

After all postings are scored, sort by `summary_score` descending (ties broken by `experience_match` descending). Assign `rank` starting at 1.

Write `jobs/scored/dashboard-<today_iso>.md` with this structure:

```markdown
# Scored Jobs — {{today_iso}}

{{N}} postings scored. Threshold: {{threshold}}. Weights: Experience {{exp_weight}} / Network {{net_weight}} / Target {{tgt_weight}}.

| Rank | Company | Title | Type | Location | Mode | Comp | Exp | Net | Tgt | Sum | Status |
|------|---------|-------|------|----------|------|------|-----|-----|-----|-----|--------|
| 1 | {{company}} | {{title}} | {{type}} | {{location}} | {{mode}} | {{comp_text or "—"}} | {{experience_match}} | {{network_proximity or "—"}} | {{target_company}} | {{summary_score}} | {{⭐ if above_threshold else blank}} |
| 2 | ... |

## Notes

- Jobs marked ⭐ are at or above the threshold of {{threshold}} and are eligible for `/draft-tailored-resume-coverletter <rank>`.
- Per-job reasoning lives in `jobs/scored/<id>.md` — open the one you want to dig into.
- {{If any network sub-score is null:}} {{N}} job(s) couldn't be network-scored this run (see per-job files for why). Use `/rescore` after fixing the issue.
```

Column rules:

- **Title** — truncate to ~50 chars if it's a long one; full title lives in the per-job file.
- **Comp** — verbatim from inbox `comp_text` if disclosed, else `—`.
- **Net** — show `—` if `network_proximity` is null; otherwise the integer.
- **Status** — ⭐ if `above_threshold`, blank otherwise. Don't conflate with the CSV's `status` column.
- **Mode** — `Remote` / `Hybrid` / `Onsite` exactly.

---

## Step 4 — Report to the user

Print the dashboard table inline (the same markdown the file contains, so it renders in the terminal), followed by a one-line summary:

```
Scored {{N}} postings.
  {{above_threshold_count}} above the threshold of {{threshold}} (⭐ in the table).
  {{below_threshold_count}} below.
  {{network_fail_count, if > 0}} job(s) skipped network scoring — see per-job files.

Dashboard written to jobs/scored/dashboard-{{today_iso}}.md.
Per-job reasoning written to jobs/scored/<id>.md.

Next: pick a ⭐ rank and run /draft-tailored-resume-coverletter <rank>.
```

If there are zero ⭐ jobs, end with:

```
No postings cleared the threshold this run. Either widen criteria, lower the threshold in settings.md, or wait for the next scan.
```

---

## Failure modes

- **No inbox files** → bail at preflight with a `/scan-jobs first` message.
- **Base resume path missing or unreadable** → bail at preflight; point the user to `settings.md`.
- **`/network-check` fails for one company** → per-job graceful degradation (Step 1b). Don't kill the whole run.
- **`/network-check` fails for every company in a row** (e.g., Chrome down for the whole run) → continue, but in the final summary lead with a clear "network scoring was unavailable this run — re-run after reconnecting Chrome" message so the user knows the dashboard is missing a dimension.
- **Weights don't sum to 1.0** → warn but proceed with what's in `settings.md`. Don't silently normalize — the user wrote a number for a reason.
- **An inbox file is malformed** (missing required frontmatter fields) → skip that file, log the filename and the missing field in the run summary, continue with the rest.

Loud, honest failure beats a confidently wrong score. The user can re-score after fixing the issue.

---

## Implementation notes

- Cache the base resume content **once** at the top of the run. Multiple reads of the same file across N postings is wasted work.
- `/network-check`'s own cache (`jobs/network-cache.json`) handles dedup across postings at the same company — you don't need a second cache layer here.
- The per-job scored file is overwritten if it already exists for the same `id`. That's intentional — re-running `/score-jobs` should refresh everything.
- The dashboard file is dated, so multiple runs on the same day overwrite that day's dashboard. Multiple runs on different days produce multiple dashboards, which is useful history.
- When iterating inbox files, sort by filename (which starts with the source + ISO date) so the run is deterministic and reproducible across re-runs.
