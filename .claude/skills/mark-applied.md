---
name: mark-applied
description: Record that the user submitted an application for a scored job. Takes a rank or job id, moves the posting file from jobs/scored/ to jobs/applied/, and updates jobs/seen.json so the same posting doesn't resurface in future scans.
---

# /mark-applied — record an outcome: applied

> **Status: stub.** The full skill is a small follow-on after the first few weeks of real weekly use. The interface and intent below are stable; the file movement and `seen.json` update are straightforward to implement on first invocation.

You take a rank (from the most recent `jobs/scored/dashboard-*.md`) or a job id and record that the user submitted an application for that posting.

## What this skill should do

1. **Resolve the argument** — same logic as `/draft-tailored-resume-coverletter` Step 1 (rank → dashboard lookup → id; or id-directly). If no argument is given, list ⭐ jobs from the latest dashboard and ask via `AskUserQuestion` which one was applied to.

2. **Move the scored file** — `mv jobs/scored/<id>.md jobs/applied/<id>.md`. Keep the file's contents intact; the move is the record. Also move any matching `jobs/research/<id>-network.md` if present.

3. **Update `jobs/seen.json`** — update the entry for the posting's URL with `"status": "applied"` and add `"applied_date": "<today_iso>"`.

4. **Optionally update `config/target_companies.csv`** — if the company's `status` column is `Researching`, ask the user via `AskUserQuestion` whether to bump it to `Active` or `Applied`. Don't change it silently.

5. **Report** — print a one-line confirmation:

   > Marked applied: <company> · <title>. Moved jobs/scored/<id>.md → jobs/applied/<id>.md.

## Until this skill is implemented

If a user invokes `/mark-applied` and the implementation isn't filled in, the skill should:

1. Tell the user honestly that this is a stub.
2. Show them the three manual steps above (move the file, update `seen.json`, optionally edit `target_companies.csv`).
3. Offer to do the work inline this one time as a fallback.

Honest stubs that explain themselves beat silently failing skills.
