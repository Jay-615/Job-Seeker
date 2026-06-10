---
name: rescore
description: Re-score every posting currently in jobs/inbox/ — useful after the user edits criteria.md, settings.md weights or threshold, target_companies.csv priorities, or connections.csv ratings. Re-uses /network-check cache where still valid; overwrites the prior per-job scored files and dashboard.
---

# /rescore — re-score the inbox after a config change

> **Status: stub.** Trivial layered call on top of `/score-jobs`. Implementable as a thin wrapper that re-invokes the M3 logic against the existing inbox without re-running `/scan-jobs`.

You re-run the scoring pass over every posting currently in `jobs/inbox/`. The trigger is almost always a user-side config change:

- Edited `config/criteria.md` (seniority rules, industries, comp floor)
- Adjusted `config/settings.md` (scoring weights or threshold)
- Updated `config/target_companies.csv` (priorities or status)
- Rated more rows in `config/connections.csv` (raises network sub-scores via the rating-bonus mechanism)

## What this skill should do

1. **Confirm the inbox is non-empty.** If it is empty, point the user at `/scan-jobs` and stop.

2. **Tell the user what you're about to do.** A one-line preview:

   > Re-scoring N postings in jobs/inbox/ against the current config. Network cache (jobs/network-cache.json) will be reused for any company whose cache is still within network_cache_days. No LinkedIn calls will be made for cached companies.

3. **Run the `/score-jobs` logic.** Same Step 1 through Step 4 as M3. Overwrites the per-job scored files at `jobs/scored/<id>.md` and writes a fresh `jobs/scored/dashboard-<today_iso>.md`. Today's dashboard is overwritten; prior dates are preserved as history.

4. **Report.** Same summary shape as `/score-jobs`, with a leading note that this was a rescore rather than a fresh scan.

## Why this is its own skill

`/score-jobs` is the right command after `/scan-jobs`. `/rescore` is the right command after a config change — the difference is purely a matter of user intent and run summary wording. Implementation can be a thin wrapper that calls into M3's logic with a "this is a rescore" flag.

## Until this skill is implemented

`/score-jobs` does the right thing today — it re-scores everything in `jobs/inbox/` on every invocation. `/rescore` is a convenience alias with friendlier summary wording. If a user invokes `/rescore` against this stub, the honest move is: tell them `/score-jobs` already does what they want, run it for them, and note the wording difference.
