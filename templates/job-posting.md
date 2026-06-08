---
id: {{source}}-{{yyyy-mm-dd}}-{{company-slug}}-{{role-slug}}
source: {{linkedin | indeed | dice | career-page}}
url: {{full URL to the posting}}
posted_date: {{yyyy-mm-dd, or "unknown" if not visible}}
fetched_date: {{yyyy-mm-dd}}
company: {{Company name}}
title: {{Job title exactly as posted}}
location: {{City, State — or "Remote (US)"}}
mode: {{Remote | Hybrid | Onsite}}
type: {{full-time | contract | contract-to-hire | part-time}}
comp_disclosed: {{true | false}}
comp_text: "{{verbatim comp range from posting, e.g. "$150,000 - $190,000" or "$90/hr"; empty string if not disclosed}}"
comp_floor_met: {{true | false | null if comp_disclosed is false}}
---

## Job Description

{{Full JD body text, preserved as posted. Keep paragraph breaks. Strip
boilerplate footers (EEO, benefits legalese) only if they crowd the file.}}

## Key Requirements (extracted)

- {{Bullet capturing a "must have" from the JD — years of experience, domain, etc.}}
- {{Bullet capturing a required skill or methodology.}}
- {{Bullet capturing tools or platforms named in the JD.}}
- {{Bullet capturing education / certification requirements, if any.}}

## Nice-to-haves (extracted)

- {{Bullet for stated "nice to have" or "bonus" qualifications.}}
- {{Bullet for cultural or working-style signals worth noting.}}

## Notes

{{Free-text notes from the scan — e.g. "JD mentions PCI compliance",
"Hiring manager named in posting: Jane Doe", "Posted 14 days ago — may be
stale". Optional.}}
