# config/

This folder holds your personal configuration for Job-Seeker. Two kinds of files
live here:

## `.example.*` files (committed to git, shared with everyone)

Files like `criteria.example.md`, `target_companies.example.csv`,
`connections.example.csv`, and `settings.example.md` are committed to the repo.
They show a fictional persona (Sam Chen — a Senior PM in e-commerce) so anyone
browsing the GitHub repo can see exactly what the data model looks like, and so
new users have a working starting point to copy from.

**Don't put your real data in these files.** They are reference material.

## Real config files (gitignored, stay on your Mac)

The real files — `criteria.md`, `target_companies.csv`, `connections.csv`,
`settings.md` — are gitignored. They never leave your machine, never get
committed, and never get pushed to GitHub. Your job criteria, target companies,
LinkedIn connections, and contact info are yours alone.

## How to set them up

The easiest path is to run `/setup` once after cloning. The setup skill
interviews you for the values it needs and writes the real files for you. You
can edit them by hand any time afterwards — they're plain markdown and CSV.

If you'd rather populate them by hand: copy each `*.example.*` file to the same
name without `.example` (for example, `cp criteria.example.md criteria.md`) and
edit. The `.gitignore` will keep your real files out of git automatically.

## What lives where

| File | Purpose |
|---|---|
| `criteria.md` | Roles, locations, seniority, comp floor, job types, industries |
| `target_companies.csv` | Companies you're tracking, with priority and status |
| `connections.csv` | Your LinkedIn 1st-degrees with High/Medium/Low ratings |
| `settings.md` | Contact info, resume path, scoring weights, thresholds, guardrails |
