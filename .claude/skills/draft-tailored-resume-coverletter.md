---
name: draft-tailored-resume-coverletter
description: Generate a tailored resume and cover letter for a specific scored job. Takes a rank or job id, reads the scored + inbox files and the user's base resume, writes markdown drafts under applications/<company>-<role>-<date>/, asks the user whether to also produce DOCX via pandoc (default — markdown only on first pass), and writes a notes.md flagging stretches, gaps, and strategic suggestions. Will not fabricate experience the user doesn't have. Does NOT touch the actual job application — the user submits themselves.
---

# /draft-tailored-resume-coverletter — tailored draft for one job

You generate a tailored resume and cover letter for a single scored job. Markdown is always the source of truth and is written on the first pass. **DOCX generation is opt-in** — you ask the user before invoking pandoc, defaulting to markdown only so they can iterate before committing to a "final" docx.

You do NOT touch the actual application portal. The user submits the application themselves.

The skill takes one argument: a **rank** (integer like `1`) or a **job id** (string like `linkedin-2026-05-26-nike-director-lp-ops`). If no argument is given, list the ⭐ jobs from the most recent dashboard and ask which one to draft.

---

## Step 0 — Preflight

1. Read `config/settings.md`. Parse:
   - **User Profile** — Name, Email, Phone, Location, Base resume path.
   - **Output — Pandoc path** — absolute path to pandoc. Default `/usr/local/bin/pandoc` if unparseable.

2. Read the user's base resume from `Base resume path`. Cache the full text once. If the file doesn't exist or isn't readable, bail with:

   > Couldn't read base resume at `<path>`. Update `Base resume path:` in `config/settings.md` and re-run.

3. Read `templates/resume.md` and `templates/cover-letter.md` once. These are structural references — sections, ordering, tone — not content. The tailored output should follow the template's structure but be populated from the base resume and JD.

4. Set `today_iso = YYYY-MM-DD`.

---

## Step 1 — Resolve the argument to a scored job

Three cases:

### 1a. Argument looks like a job id

A job id contains hyphens and starts with a source prefix (`linkedin-`, `indeed-`, `dice-`, `career-page-`). If the argument matches, look for `jobs/scored/<id>.md`. If it exists, use it. If not, list the available ids in `jobs/scored/` and ask the user to pick.

### 1b. Argument is an integer (a rank)

1. Find the most recent dashboard: list `jobs/scored/dashboard-*.md`, sort by filename descending (the date in the filename sorts lexically), pick the first.
2. If no dashboard exists, bail with:

   > No scored dashboard found. Run `/score-jobs` first, then come back with a rank.

3. Parse the dashboard's table. Each row starts with `| <rank> | <company> | <title> | ...`. Find the row where rank matches the argument. From that row, take `company` and `title` and use them to look up the matching `jobs/scored/<id>.md` — the id is derivable, but easier: scan `jobs/scored/*.md` (excluding `dashboard-*.md`), read each one's frontmatter, and match on `company` + `title` against the dashboard row.
4. If the rank is out of range, bail with:

   > Rank `<n>` not found in `jobs/scored/dashboard-<date>.md` (it has <m> rows). Re-check and try again.

### 1c. No argument

List the ⭐ rows from the latest dashboard with their ranks, then ask the user which rank to draft. If there are no ⭐ rows, list all rows. If the dashboard is missing, fall back to the same bail as 1b.

Once resolved, you have `scored_path` (`jobs/scored/<id>.md`) and the corresponding `inbox_path` (`jobs/inbox/<id>.md`, same id).

---

## Step 2 — Read the scored + inbox files

1. Read `scored_path`. Parse frontmatter (`id`, `experience_match`, `network_proximity`, `target_company`, `summary_score`, `above_threshold`) and the narrative body. The narrative tells you where the agent already flagged gaps — reuse those flags rather than re-reasoning from scratch.

2. Read `inbox_path`. Parse frontmatter (`company`, `title`, `location`, `mode`, `type`, `url`, `comp_text`) and the body (`## Job Description`, `## Key Requirements`, `## Nice-to-haves`, `## Notes`).

3. If either file is missing, bail with a clear message naming the missing path.

---

## Step 3 — Compute the application folder path and detect re-runs

1. Compute slugs (lowercase, hyphens, alphanumeric only):
   - `company_slug` from the inbox `company` field
   - `role_slug` from the inbox `title` field, truncated to ~50 chars if long

2. `app_dir = applications/<company_slug>-<role_slug>-<today_iso>/`

3. **If `app_dir` already exists**, the user has run this skill against this job before. Use `AskUserQuestion` with three options:

   - **Regenerate from scratch** — overwrite `resume.md`, `cover-letter.md`, `notes.md`. Use this when you want a fresh tailored pass.
   - **Keep existing markdown, generate DOCX now** *(Recommended)* — skip Step 4–6, go straight to the DOCX prompt in Step 7. Use this when you've iterated on the markdown and are ready to submit.
   - **Cancel** — exit without changing anything.

   If they pick **Regenerate** or **Cancel**, behave accordingly. If they pick **Keep existing markdown, generate DOCX now**, jump to Step 7 with `force_docx_prompt = true`.

4. If `app_dir` doesn't exist yet, create it.

---

## Step 4 — Draft the tailored resume

The tailored resume **reorders, sharpens, and emphasizes** content from the user's base resume to align with the JD. It does NOT invent new experience.

Rules:

- **Preserve the structure of `templates/resume.md`** — name + contact header, Summary, Experience, Skills, Education, Certifications.
- **Header** — populate from `settings.md` User Profile (Name, Location, Email, Phone). LinkedIn URL is optional; include it only if it appears in the base resume.
- **Summary** — rewrite the base resume's summary to lead with whatever the JD calls for most. Two to three sentences. Use the user's voice from the base resume; don't generate corporate-speak.
- **Experience** — keep the same jobs and dates as the base resume. Reorder bullets within each role to lead with the most JD-relevant work. You may also:
  - Tighten phrasing for impact (active verbs, quantified outcomes).
  - Drop bullets that are clearly off-topic for this role (note any dropped bullets in `notes.md`).
  - **Do NOT add bullets describing work the user didn't do.** If the JD asks for something the resume doesn't show, flag it in `notes.md`, don't insert it.
- **Skills** — re-group or reorder to lead with the categories most relevant to the JD. Don't add skills not in the base resume.
- **Education, Certifications** — copy verbatim from the base resume.

Honor Jay's writing preferences (from his global memory):
- **Spell Kasada with a K, not a C** if it appears.
- **Avoid "spend" as a noun** — use "cost" or "expenditure".

Write the result to `<app_dir>/resume.md`.

---

## Step 5 — Draft the tailored cover letter

Three substantive paragraphs, ~300–400 words total. Shorter is better. Use the structure in `templates/cover-letter.md`.

Header block (top of the file):
- Name, location, email, phone — pulled from `settings.md`.
- Today's date.
- Addressee block — `Hiring Team` + `<company>` + the inbox `location`. (If the inbox `## Notes` names a hiring manager, use that name instead of "Hiring Team".)

Body:

1. **Opening hook (1–2 sentences)** — name the role you're applying for and connect a single specific strength of yours to a specific need in the JD. Do not open with "I am writing to apply for…".

2. **Evidence paragraph (2–3 sentences)** — point to concrete experience from the resume that maps to the role's top requirement. Quantify where you can. Explain why it matters for *this* job; don't just restate the resume.

3. **Optional second evidence paragraph** — only if there's a clear second requirement to address, or a likely concern to address head-on (career pivot, gap, geography). Keep it tight. Skip this paragraph rather than padding.

4. **Close (1–2 sentences)** — reaffirm interest, state availability for a conversation, thank them. End with the user's name.

Honor Jay's writing preferences as in Step 4.

**Don't fabricate.** If a strong-sounding sentence requires experience the user doesn't have, don't write it. Flag the gap in `notes.md` instead.

Word count check: count words in the body (excluding header and signature). If over 400, tighten before writing. Record the final word count for the summary.

Write the result to `<app_dir>/cover-letter.md`.

---

## Step 6 — Write notes.md

This is the agent's honesty file. The user reads it to decide what to revise before submitting. Structure:

```markdown
# Notes — <company> · <title>

**Job ID:** <id>
**Drafted:** <today_iso>
**Summary score:** <summary_score> (Exp <experience_match> / Net <network_proximity> / Tgt <target_company>)
**Posting URL:** <url>

## Highlights

- <One-line explanation of what was emphasized in the resume and why.>
- <One-line explanation of the cover letter's framing choice.>
- <Anything notable about how the JD aligns with the user's background.>

## Gaps and stretches

- <Each JD requirement the resume doesn't clearly cover. Be specific: "JD asks for PCI compliance experience; resume doesn't mention it. I omitted the claim — add a line if you have relevant experience." One bullet per gap.>
- <Any bullet you dropped from the base resume and why.>

## Suggestions

- <Optional: small revisions the user might consider, e.g., "Consider naming the Kasada bot-detection project explicitly in the summary — it's the clearest match to the JD's fraud-and-abuse line.">
- <Optional: outreach angle drawn from the scored file's warmest paths, if present.>

## What I did NOT do

- I did not invent experience. Items the JD asks for that aren't in your base resume are listed under Gaps above — add them yourself if you have the evidence.
- I did not submit the application. You submit it yourself when the draft is ready.
```

Write the result to `<app_dir>/notes.md`.

---

## Step 7 — Ask about DOCX (opt-in)

After the three markdown files are written, ask the user via `AskUserQuestion`:

- **Question:** "Generate DOCX versions now?"
- **Header:** "DOCX output"
- **Options:**
  1. **Markdown only (recommended for first pass)** — *Default. Iterate on the markdown, generate DOCX later when you're ready to submit.*
  2. **Markdown + DOCX (ready to submit)** — *Run pandoc now and produce resume.docx and cover-letter.docx alongside the markdown.*

If you reached Step 7 from the "Keep existing markdown, generate DOCX now" branch in Step 3, default to option 2 (still use `AskUserQuestion` so the user can back out, but lead with the DOCX option).

### If the user picks Markdown only

Skip Step 8. Go to Step 9.

### If the user picks Markdown + DOCX

Continue to Step 8.

---

## Step 8 — Generate DOCX via pandoc (only if opted in)

1. **Verify pandoc is installed.** Run `<pandoc_path> --version` (use the path from `settings.md`). If it errors:
   - Try `which pandoc` as a fallback.
   - If still not found, do NOT crash. Tell the user:

     > Pandoc isn't installed (or not at `<pandoc_path>`). Install with `brew install pandoc` (or update the path in `settings.md`), then re-run `/draft-tailored-resume-coverletter <rank>` and choose the DOCX option. Your markdown files are already written.

   …and skip to Step 9.

2. Run pandoc on each markdown file. From the working directory:

   ```bash
   <pandoc_path> -f markdown -t docx -o <app_dir>/resume.docx <app_dir>/resume.md
   <pandoc_path> -f markdown -t docx -o <app_dir>/cover-letter.docx <app_dir>/cover-letter.md
   ```

3. After each pandoc invocation, verify the `.docx` file actually exists and is non-empty. If either is missing, tell the user clearly which one failed and what the pandoc output was — don't claim success.

---

## Step 9 — Report to the user

Print a concise summary:

```
Drafted application: applications/<company-slug>-<role-slug>-<today_iso>/
  - resume.md  (source){{  →  resume.docx  if DOCX was generated}}
  - cover-letter.md  (source){{  →  cover-letter.docx  if DOCX was generated}} ({{N}} words)
  - notes.md

Highlights from notes:
- {{2–4 of the most important bullets from notes.md, copied verbatim. Lead with strengths, then flags.}}
```

If DOCX was skipped on this pass, append:

```
DOCX not generated this run. When you're ready to submit, re-run
/draft-tailored-resume-coverletter <rank> and choose the DOCX option,
or run pandoc directly:
  <pandoc_path> -f markdown -t docx -o applications/<...>/resume.docx applications/<...>/resume.md
```

End the turn. Do NOT auto-iterate. The user will read the drafts and come back with revisions in conversation.

---

## Failure modes — handle cleanly

| Situation | What to do |
|---|---|
| No argument given AND no dashboard exists | "No scored dashboard found. Run `/scan-jobs` then `/score-jobs` first." |
| Argument given but rank/id not found | Name the dashboard path you searched and the available range; ask the user to retry. |
| `jobs/scored/<id>.md` exists but `jobs/inbox/<id>.md` is missing | Bail. The pair must exist together — that's a data-integrity issue worth surfacing. |
| Base resume path in `settings.md` is wrong | Bail with the path and remediation (edit `settings.md`). |
| `templates/resume.md` or `templates/cover-letter.md` missing | Bail. These come from M6; tell the user the template is missing and suggest re-cloning or checking the repo. |
| Pandoc not installed (and user opted into DOCX) | Don't crash. Markdown is already written; tell the user how to install pandoc and how to retry. |
| `<app_dir>` already exists | Use the `AskUserQuestion` branch in Step 3 — never silently overwrite. |
| User's base resume contains "Casada" or similar misspellings of Kasada | Quietly correct to "Kasada" in the tailored output (per Jay's writing preference). |

---

## What this skill must NOT do

- **Do not fabricate experience.** No invented bullets, no inflated titles, no fake metrics. Gaps go in `notes.md`, not into the resume.
- **Do not touch the application portal.** This skill writes local files only.
- **Do not generate DOCX without asking.** Markdown-only is the default first pass.
- **Do not overwrite an existing application folder without asking.**
- **Do not score, re-score, or re-rank jobs.** That's `/score-jobs` and `/rescore`.
- **Do not write the cover letter past ~400 words.** Tighten instead.
