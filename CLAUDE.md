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
│       ├── draft-application.md         # tailored resume + cover letter
│       ├── research-network.md          # Tier 2 deep network research per job
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
│   ├── connections.csv                  # GITIGNORED (user's actual)
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
- `/draft-application <rank>` — generate tailored resume + cover letter
- `/mark-applied <rank>` and `/mark-passed <rank>` — track outcomes
- `/rescore` — re-score the inbox after criteria changes

## Conventions

- All user-specific data lives in `config/` (gitignored) and is populated by `/setup`.
- Job postings and applications are written to local files under `jobs/` and
  `applications/` (also gitignored).
- Markdown is the source of truth for resumes and cover letters; DOCX is generated.
- Hard filters drop misfits before scoring; soft scores rank survivors.
  See docs/scoring-methodology.md.

## When a user opens this project for the first time

Check if `config/settings.md`, `config/criteria.md`, and `config/target_companies.csv`
exist. If any is missing, recommend they run `/setup`.
