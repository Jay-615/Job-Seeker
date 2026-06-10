# Job-Seeker Settings

*Example persona: Sam Chen. Copy this file to `settings.md` and edit values
for your own profile. All defaults below are shown explicitly so you can see
what `/setup` would write.*

## User Profile
- Name: Sam Chen
- Email: sam.chen@example.com
- Phone: (206) 555-0142
- Location: Seattle, WA
- Base resume path: /Users/samchen/Documents/Resumes/sam-chen-resume.md

## Scoring Weights
- Experience match: 0.5
- Network proximity: 0.3
- Target company: 0.2

## Threshold
- Draft-ready threshold: 70

## Sources (turn off any to skip)
- linkedin: true
- indeed: true
- dice: true
- company_career_pages: true

## Cost Guardrails (company career pages only)
- priority_filter: High         # only scan companies at this priority or above
- max_pages_per_run: 10         # cap total companies scanned per /scan-jobs run
- cache_results_days: 3         # don't re-scan a company within N days

## Network Proximity
- network_cache_days: 30        # reuse cached network analysis per company within N days
- include_deep_research_default: false  # /score-jobs does NOT auto-trigger Tier 2 deep research
- research_profile_cap: 10      # /research-network visits at most N 2nd-degree profiles per run

## Output
- Resume formats: markdown, docx
- Cover letter formats: markdown, docx
- Pandoc path: /usr/local/bin/pandoc   # adjust if installed elsewhere
