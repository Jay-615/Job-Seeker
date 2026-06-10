# Where humans stay in the loop

Job-Seeker is deliberately not an auto-applier. It isn't a scheduled cron job. It stops at "tailored draft ready" and hands the application back to you.

This document explains why — and where the human/agent boundary sits at each step of the workflow. It's the one I most want a hiring manager to read, because it's the part where I had to make actual PM choices about where the human belongs.

---

## The three "no"s

### 1. No auto-apply

The tool drafts a tailored resume and cover letter in markdown (and, when you ask, in DOCX). It does **not** open the application portal, fill the form, upload the files, or click Submit.

The temptation to build that last mile is real. The honest reasons not to:

- **Application portals are heterogeneous and fragile.** Workday, Greenhouse, Lever, iCIMS, Workable, Ashby, Jobvite, bespoke ATSes — each with its own field map, custom questions, and assessment quirks. Auto-fill across all of them is an indefinite engineering investment for a workflow most users run a few times a week.
- **Cover letters and resumes get final-pass human edits anyway.** Almost every draft I generate I edit once before submitting — a sentence to sharpen, a metric to add, a phrase that's almost-but-not-quite right. The "iterate on the markdown, generate DOCX when ready" flow in `/draft-tailored-resume-coverletter` is built around that loop. Auto-submitting would skip the loop where most of the quality actually lives.
- **The submit-button moment is the right time to re-read the materials.** Every submitted application is a public artifact attached to your name. Even a well-tailored draft benefits from one calm read-through before it ships. The tool intentionally builds in that pause.
- **Failure modes are asymmetric.** A scoring miss costs you a few minutes of reading a borderline JD. A bad auto-submit costs you a real application slot — and possibly your credibility with that recruiter for future roles.

The tool ends with `notes.md` — the agent's honest list of what it emphasized, what it dropped, and what gaps it didn't try to paper over. That file exists because the *next* step belongs to the human.

### 2. No scheduling

No cron jobs. No background runs. No "scan every Monday at 7am." Every command is manually triggered.

This is partly practical (Claude in Chrome operates against your logged-in browser session — there's no "headless" mode that doesn't break the trust model) and partly philosophical. The dominant 2026 paradigm for AI in knowledge work is **partner, not autonomous worker**. A job search is a few hours per week of judgment-intensive work, not an inbox that needs draining overnight. The tool is built to be invoked when you sit down to job-search, and to give you 30 useful minutes — not to email you a dashboard at 6am that you'll skim and ignore.

There's also a small subsidy in the manual-trigger model: every run is an active choice. If you skip a week, the inbox doesn't grow stale and the network cache doesn't quietly drift. You come back, you `/scan-jobs`, and you see what's there.

### 3. No SaaS, no central database, no accounts

The whole tool runs on your Mac. Your criteria, target companies, connections, scored postings, and tailored drafts never leave the machine. There's no server-side code, no shared database, no authentication. The repo itself is public, but `config/`, `jobs/`, and `applications/` are gitignored — they only exist on your local filesystem.

Two reasons:

- **Job-search data is sensitive.** Your target companies are a window into where you'd rather work. Your connections list and ratings are deeply personal. Your tailored drafts reveal what you're applying to. None of that should sit on someone else's server.
- **Local-first is auditable.** Every artifact is plain markdown or CSV on your disk. You can read what the agent wrote, edit it by hand, version it yourself, or delete it. There's no "see how the algorithm scored you" because the algorithm is in `score-jobs.md` and you can read it.

---

## Where the human shows up at each step

A walk through the weekly workflow, with explicit human/agent boundaries.

### `/scan-jobs` — find new postings

- **Agent does:** search LinkedIn / Indeed / Dice / company career pages, parse cards, apply hard filters, dedupe against `seen.json`, write inbox files.
- **Human stays in the loop when:** Indeed pushes a CAPTCHA. The tool pauses, asks via `AskUserQuestion` whether to clear it manually (a 30-second flip to Chrome) or skip the source. This is the "AI as partner" pattern in microcosm: when the site asks for proof of humanity, the human shows up, clears it, and the agent picks up where it stopped.

### `/score-jobs` — rank the inbox

- **Agent does:** score each posting on three dimensions, narrate the reasoning per posting, write a ranked dashboard table.
- **Human stays in the loop when:** reading the per-job `jobs/scored/<id>.md` files. The dashboard summary score is the headline; the narrative paragraphs are the substance. The agent flags conspicuous gaps ("JD mentions PCI compliance — not on your resume") rather than papering over them, and the human decides whether the gap matters for *this* application.

### `/research-network <rank>` — Tier 2 deep network research

- **Agent does:** visit up to 10 2nd-degree profiles at the target company, read each person's role, judge area relevance honestly, write an outreach planning document with a prioritized Top 3 and draft opening lines.
- **Human stays in the loop when:** *actually sending the messages*. The draft opening lines are starting points in your voice and your relationship context. The agent has no idea whether you and the mutual went to college together or last spoke a decade ago. The output is "here are the three people I'd ask, and here's an opening line you can adapt" — never "I sent the message."

### `/draft-tailored-resume-coverletter <rank>` — generate drafts

- **Agent does:** read the scored file + inbox file + your base resume, tailor a resume that reorders/sharpens (without inventing) experience, write a 300–400 word cover letter, produce a `notes.md` listing highlights, gaps, and what the agent declined to write.
- **Human stays in the loop when:**
  - **Reading `notes.md` before editing the resume.** The agent's honesty file is the most valuable artifact of this step — it tells you which JD requirements it didn't try to fake.
  - **Iterating on the markdown.** DOCX generation is opt-in by default. The expected workflow is: generate markdown, read it, edit it, possibly regenerate, then ask for DOCX once you're happy.
  - **Submitting the application.** The folder on disk is named `applications/<company>-<role>-<date>/` because that's where the materials live — the tool never opens a browser window pointed at the apply form.

### `/mark-applied` / `/mark-passed` — record outcomes

- **Agent does:** move the posting file from `jobs/scored/` to `jobs/applied/` or `jobs/passed/`, update `seen.json` so the same job doesn't resurface.
- **Human stays in the loop when:** deciding to invoke either of these in the first place. Marking a posting "applied" is an honest statement that you submitted; marking it "passed" is a judgment that you've considered it and chosen not to pursue. Both are decisions the tool defers to the human entirely.

---

## What this looks like in practice

In the smoke run that informed the current cut of M7, the dashboard surfaced one ⭐ job: a Nike "Product Manager" contract posting. The full chain played out like this:

1. **Agent flagged** that the posting was "Promoted by hirer · Responses managed off LinkedIn," so the JD body wasn't readable inside LinkedIn. Experience scoring was thinner than usual; the per-job narrative said so honestly.
2. **Agent suggested** rating the three Nike mutuals who recur across every candidate's list as a 60-second move that would lift the Tier 1 Nike network score from 70 to 100. **Human chose** whether to do that or proceed with the current score.
3. **Agent ran** `/research-network 1` with a smoke-scoped cap of 3 profiles, identified three direct-area candidates at Nike, drafted opening lines for three distinct intro paths.
4. **Human reads** the opening lines, decides whether to send any of them, edits the language to match the actual relationship with each mutual, and sends from their own LinkedIn account.

The agent does the work that benefits from being fast, systematic, and tireless. The human does the work that benefits from judgment, relationship context, and accountability for the outcome. Neither side does the other's job, and the file boundaries make the split visible.

---

## Why this is the M7 artifact I most want hiring managers to read

A lot of 2026 AI tooling solves the "make it autonomous" problem. The harder, more interesting problem is **deciding where to stop**. Every step above could be automated further. The choice to *not* automate it is a PM call: it's where I think the marginal cost of a wrong agent action exceeds the marginal benefit of saving the human's time.

I'd rather build a tool that respects the user's judgment at the right moments than one that impresses on first demo and erodes trust on the third run. The auto-apply gap is the clearest example — that's where this tool stops because that's where the next action belongs to me, not to Claude.
