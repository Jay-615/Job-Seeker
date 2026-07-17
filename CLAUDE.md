# Job-Seeker

You are running inside the Job-Seeker workflow — a personal job-search tool
that helps the user find, score, and apply to relevant job postings using
Claude in Chrome for LinkedIn access and local files for storage.

## What this project does

Job-Seeker is a personal job-search workflow built as a Claude Code workspace.
A user clones the repo, runs `claude` in the folder, runs a guided setup once,
and then twice a week runs a small set of commands to find new postings, score
them, review the ranked results, generate tailored resumes and cover letters,
and track outcomes so the same jobs don't resurface.

Postings are pulled from LinkedIn, Indeed, Dice, and selected company career
pages. Hard filters drop obvious misfits before scoring; soft scores rank what
survives across three dimensions — experience match, network proximity, and
target-company alignment — combined into a weighted summary score.

The tool is deliberately not an auto-applier and not a scheduled job. All runs
are manually triggered, and the workflow stops at "tailored draft ready" so
the user submits the application themselves. The user stays in the loop for
the judgment calls; Claude handles the heavy lifting around scanning, scoring,
and drafting.

## Repository layout

```
Job-Seeker/
├── .gitignore
├── LICENSE                              # MIT
├── README.md                            # public-facing; for portfolio audience
├── CLAUDE.md                            # this file — orients any Claude session
├── .claude/
│   └── skills/                          # one file per slash command
│       ├── setup.md                     # first-run guided configuration
│       ├── scan-jobs.md                 # find new postings, filter, dedupe
│       ├── score-jobs.md                # rank the inbox on three dimensions
│       ├── network-check.md             # Tier 1 network-proximity helper
│       ├── build-connections.md         # populate connections.csv from LinkedIn
│       ├── rate-connections.md          # interactive rating walk-through
│       ├── draft-tailored-resume-coverletter.md  # tailored resume + cover letter (markdown; DOCX opt-in)
│       ├── research-network.md          # Tier 2 deep network research per job
│       ├── map-company-connections.md   # find + document 2nd-degree connections per company → company_connections.csv
│       ├── mark-applied.md              # record an outcome: applied
│       ├── mark-passed.md               # record an outcome: passed
│       └── rescore.md                   # re-score the inbox after criteria changes
├── config/
│   ├── README.md                        # explains the .example convention
│   ├── criteria.example.md              # committed; fictional persona
│   ├── criteria.md                      # GITIGNORED (user's actual)
│   ├── target_companies.example.csv     # committed; fictional persona
│   ├── target_companies.csv             # GITIGNORED (user's actual)
│   ├── connections.example.csv          # committed; fictional persona
│   ├── connections.csv                  # GITIGNORED (user's actual; has cached member_id column)
│   ├── company_connections.csv          # GITIGNORED — 2nd-degree connections mapped per company (from /map-company-connections)
│   ├── settings.example.md              # committed; fictional persona
│   └── settings.md                      # GITIGNORED (user's actual)
├── templates/
│   ├── resume.md                        # markdown resume template
│   ├── cover-letter.md                  # markdown cover letter template
│   └── job-posting.md                   # frontmatter template for inbox jobs
├── docs/
│   ├── scoring-methodology.md           # explains the filter/score model
│   ├── humans-in-the-loop.md            # explains the design choices
│   └── architecture.md                  # summary of architectural decisions
├── jobs/                                # GITIGNORED (entire directory)
│   ├── inbox/                           # new postings, raw
│   ├── scored/                          # postings after scoring + dashboards
│   ├── applied/                         # postings the user applied to
│   ├── passed/                          # postings the user decided not to pursue
│   ├── research/                        # deep network research outputs
│   ├── seen.json                        # dedup index of URLs ever seen
│   ├── career-page-cache.json           # cache of company-page scan dates
│   └── network-cache.json               # cache of Tier 1 network analysis
└── applications/                        # GITIGNORED (entire directory)
    └── <company>-<role>-<date>/         # one folder per drafted application
```

## Key skills

- `/setup` — first-run guided configuration (see .claude/skills/setup.md)
- `/scan-jobs` — find new postings (see .claude/skills/scan-jobs.md)
- `/score-jobs` — rank the inbox (see .claude/skills/score-jobs.md)
- `/map-company-connections <company>` — map Jay's 2nd-degree connections at a company + the 1st-degree bridges to them, drill-in-complete, into `config/company_connections.csv` (the source of truth that `/network-check` scores from and `/research-network` plans outreach from)
- `/draft-tailored-resume-coverletter <rank>` — generate tailored resume + cover letter for a scored job (the user still submits the actual application themselves)
- `/mark-applied <rank>` and `/mark-passed <rank>` — track outcomes
- `/rescore` — re-score the inbox after criteria changes

## Conventions

### Never write a real person's name into a committed file

**This repo is public.** Every file that is not gitignored is published to
strangers the moment it's pushed.

Real people's names — anyone in `config/connections.csv` or
`config/company_connections.csv` — must never appear in any tracked file. Not in
`docs/`, not in a skill file, not in a commit message, not in a code comment.
Use the fictional persona from `config/*.example.*` (Sam Chen and colleagues)
instead. Same for real employer names tied to the user's actual search, real
profile slugs, and real home-directory paths.

**This rule exists because it was broken.** Commit `bb3408f` — titled "portfolio
polish," whose whole purpose was making the repo presentable — published six real
colleagues' names and two profile slugs in `docs/`. It stayed public for weeks.

Understand *how* that happens, because the pull is real and it will pull on you
too. The leaking lines were a smoke-test bug report, shaped like this:

> First extraction had [real name]'s profile_url pointing to `/in/[real slug]`
> ([another real name], who's a mutual)…

That's *good documentation*. The specific mismatched pairs are the evidence for
the bug; naming them is what makes the finding credible instead of vague. The
trap is that **accurate and publishable diverge the moment the subject is a real
person**, and the instinct toward precision is exactly what walks you over the
line. "Two of the user's mutuals" is marginally worse documentation and enormously
better privacy. Take that trade every time. If a real name seems load-bearing to
an explanation, that is the signal to stop, not to proceed.

Note that this very section quotes the offending line with the names redacted —
not because the shape of the sentence is secret, but because writing them out
again to explain why you shouldn't write them out is exactly the trap described
above. It is easy to fall into. It was fallen into while drafting this paragraph.

`.gitignore` will not save you here. It protects *files*, and it has done its job
perfectly — no gitignored file has ever leaked. It cannot protect against a name
typed into a sentence. That's a judgment call, and it's yours.

A `pre-commit` hook (`.githooks/pre-commit`) checks staged files against the names
in `connections.csv` and blocks the commit. Treat it as a backstop for mistakes,
not as the thing doing the thinking. It only knows names you've already scraped.

### Other conventions

- All user-specific data lives in `config/` (gitignored) and is populated by `/setup`.
- Job postings and applications are written to local files under `jobs/` and
  `applications/` (also gitignored).
- Markdown is the source of truth for resumes and cover letters; DOCX is generated.
- Hard filters drop misfits before scoring; soft scores rank survivors.
  See `docs/scoring-methodology.md`.
- The tool is deliberately not an auto-applier and not a scheduled job — see
  `docs/humans-in-the-loop.md` for the rationale behind where humans stay in
  the loop. `docs/architecture.md` summarizes the runtime and design decisions.

## Writing preferences (Jay's voice)

When generating user-facing copy (resume bullets, cover-letter language, run
summaries, `notes.md` files):

- **Spell Kasada with a K**, not a C. If the user's base resume contains the
  misspelling, quietly correct it.
- **Don't use "spend" as a noun.** Use "cost" or "expenditure" instead.
- Tone is clear, confident, occasionally personal. Not corporate marketing.

## When a user opens this project for the first time

Check if `config/settings.md`, `config/criteria.md`, and `config/target_companies.csv`
exist. If any is missing, recommend they run `/setup`.
