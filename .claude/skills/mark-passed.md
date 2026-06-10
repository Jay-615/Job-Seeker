---
name: mark-passed
description: Record that the user decided not to pursue a scored job. Takes a rank or job id, moves the posting file from jobs/scored/ to jobs/passed/, and updates jobs/seen.json so the same posting doesn't resurface in future scans.
---

# /mark-passed — record an outcome: passed

> **Status: stub.** Mirror image of `/mark-applied`. Same shape, different destination folder and `seen.json` status.

You take a rank (from the most recent `jobs/scored/dashboard-*.md`) or a job id and record that the user reviewed the posting and chose not to pursue it.

## What this skill should do

1. **Resolve the argument** — same logic as `/draft-tailored-resume-coverletter` Step 1 (rank → dashboard lookup → id; or id-directly). If no argument is given, list all jobs from the latest dashboard and ask via `AskUserQuestion` which one was passed.

2. **Move the scored file** — `mv jobs/scored/<id>.md jobs/passed/<id>.md`. Keep the file's contents intact. Also move any matching `jobs/research/<id>-network.md` if present.

3. **Update `jobs/seen.json`** — update the entry for the posting's URL with `"status": "passed"` and add `"passed_date": "<today_iso>"`.

4. **Optionally ask why** — short `AskUserQuestion` with options like *Wrong fit* / *Wrong seniority* / *Comp* / *Location* / *Other / Skip*. Append the answer to the moved file as a `## Passed reason` section. The reason gives useful signal if the user later wants to look at their own pass-rate by category — strictly optional.

5. **Report** — one-line confirmation:

   > Marked passed: <company> · <title>. Moved jobs/scored/<id>.md → jobs/passed/<id>.md.

## Until this skill is implemented

Same posture as `/mark-applied`: be honest, show the manual steps, offer to do them inline once as a fallback.
