---
name: rate-connections
description: Interactive walk-through to apply High/Medium/Low ratings to Unrated rows in config/connections.csv. Presents one connection at a time with name, headline, and notes; writes each rating back to the CSV immediately so a mid-session crash doesn't lose progress. Reminds the user that bulk-editing the CSV in Excel or Numbers is an equally valid path.
---

# /rate-connections — interactive rating walk-through

You are helping the user assign a qualitative rating to each `Unrated` row in `config/connections.csv`. The rating drives the bonus component of the network proximity score in `/network-check` and `/score-jobs`, so even partial progress is valuable — every rated connection makes the score more useful.

This skill is **one of two paths** the user has for rating. The other is bulk-editing the CSV directly in Excel or Numbers. Both are first-class. Make that clear in the preamble — don't pretend this is the only way.

---

## Opening message to the user

Print this preamble before doing any work:

> I'll walk you through your Unrated LinkedIn connections one at a time. For each, you'll choose **High**, **Medium**, **Low**, or **Skip**. Each rating is written back to `config/connections.csv` immediately, so if anything interrupts us, you keep what you've rated so far.
>
> Rating rubric (same language used everywhere in Job-Seeker):
>
> - **High** — You know these people well. You can ask them for help right off the bat.
> - **Medium** — You know these people, but not very well, or you've been out of touch. You may want to reconnect with them a bit before asking for help.
> - **Low** — You don't really know these people. You may want to introduce yourself and establish more of a real connection before asking for help.
>
> Heads up: this is one path, not the only one. If you'd rather batch-rate in a spreadsheet, open `config/connections.csv` in Excel, Numbers, or any spreadsheet tool — sort by headline (groups people by company) and mass-update the `rating` column. Both paths produce the same result, and you can stop here at any time and switch.

---

## Step 1 — Read the CSV and find Unrated rows

1. Read `config/connections.csv`. If it doesn't exist:

   > I don't see `config/connections.csv` yet. Run `/build-connections` first — that pulls your LinkedIn 1st-degree connections into the file. Then come back here to rate them.

   Bail.

2. Parse the CSV using Python's `csv` module (run via the `Bash` tool — same approach as `/build-connections`). Keep the original column order: `linkedin_url,name,headline,rating,last_interaction,notes`.

3. Filter to rows where `rating` is empty, `"Unrated"`, or any whitespace variant. Count them.

4. If zero Unrated rows remain:

   > All your connections are already rated. (High: N / Medium: N / Low: N) — nothing to do here.

   Stop.

5. Tell the user the count and ask via `AskUserQuestion`:
   - **Start rating** (recommended)
   - **Show me the breakdown first** — print the current rating distribution + the first 5 Unrated headlines so the user can sanity-check before committing time
   - **Cancel**

---

## Step 2 — Optional sort order

Ask the user once, before the loop, how they want to walk the list:

- **By headline (default)** — groups by company / job title, so you'll often rate a cluster of co-workers in a row
- **By name** — alphabetical
- **As-is** — whatever order the CSV is in

This is a small quality-of-life choice; default to "by headline" if the user just hits Enter on the recommended option.

---

## Step 3 — The rating loop

For each Unrated row, in the chosen sort order:

1. Show the user:

   ```
   [12 / 487]    <Name>
                 <Headline>
                 Notes: <existing notes, or "(none)">
                 Profile: <linkedin_url>
   ```

   The progress counter (`[12 / 487]`) is important — it lets the user see how much is left and decide whether to push through or stop.

2. Use `AskUserQuestion` to ask: "How well do you know <Name>?"

   Options:
   - **High** — You know them well; could ask for help directly
   - **Medium** — You know them, but not well or have been out of touch
   - **Low** — You don't really know them; would need to introduce yourself
   - **Skip** — Leave Unrated for now, move to next
   - **Stop** — Save and exit the loop

   These four-plus-one options match the rubric in the preamble. The user shouldn't have to remember what each rating means — the description on each option in `AskUserQuestion` carries the rubric.

3. **Immediately after the user answers, write the updated row back to `config/connections.csv`.** Do not batch writes — a crash or a context-window cutoff between rows must not lose progress.

   The safest pattern: read the CSV, mutate the in-memory representation, write it all back. Do this on every rating. CSV files of a few thousand rows write in milliseconds — the I/O cost is negligible compared to the cost of losing user input.

4. Optional: after every ~25 rated rows, briefly remind the user they can stop any time and that bulk-editing is still an option:

   > You've rated 25 so far. Doing great. (If you'd rather finish in a spreadsheet, just choose **Stop** here and open `config/connections.csv` directly — your progress is saved.)

5. If the user chooses **Stop**, break the loop cleanly and go to Step 4.

---

## Step 4 — Summary

When the loop ends (either because all Unrated rows were addressed, or the user chose Stop), print:

```
Rating session — done.
  Rated this session: 47 (High: 8 / Medium: 14 / Low: 25)
  Skipped this session: 3 (still Unrated)
  Total in config/connections.csv: 487
    High: 39 / Medium: 36 / Low: 39 / Unrated: 373
```

Then a short closing line:

> Re-run `/rate-connections` any time to pick up where you left off — Unrated rows are the only ones I revisit. Or open `config/connections.csv` directly in Excel or Numbers if a batch session feels easier from here.

---

## Edge cases

- **CSV with manual edits in flight.** If the user has the CSV open in Excel/Numbers while running this skill, write conflicts are possible. The skill doesn't try to detect this — just assume the user knows what they're doing. If they're using both paths, they should close the spreadsheet before running this skill.

- **Rating with bad values.** If a row's `rating` cell contains something outside `{High, Medium, Low, Unrated, ""}` (e.g., a typo from manual editing), treat it as `Unrated` for the purposes of this loop, but **do not overwrite the raw value until the user gives a new rating** — preserve the user's typo in case it was intentional. Surface it in the prompt:

   > Note: this row's existing `rating` value is `"high "` (with a trailing space). I'll treat it as Unrated for now; choose a new rating to clean it up.

- **Row count changed mid-session.** If `/build-connections` runs concurrently and adds new rows, you'll still be iterating over the snapshot you read at Step 1. That's fine — the next `/rate-connections` invocation will pick up the new Unrated rows.

- **Stop during the very first prompt.** Save nothing extra, just exit cleanly with the standard summary (which will show "Rated this session: 0").

---

## Why immediate-write matters

Each rating is one quick row update. The user is doing the slow part (the judgment). The skill's job is to make sure none of that judgment ever gets lost to a refresh, a context cutoff, or an accidental window close. Always write back to disk before asking the next question.
