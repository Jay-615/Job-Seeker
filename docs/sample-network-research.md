# Sample network research output

This is a representative `/research-network` output, anonymized from a real run. It shows what the Tier 2 deep network research command produces — the outreach planning artifact the user takes to LinkedIn to actually start asking for intros.

Real outputs live at `jobs/research/<id>-network.md` (gitignored). This sample is committed so anyone browsing the repo can see what the polished artifact looks like without needing to run the workflow themselves.

Names, profile URLs, mutual counts, and the role description are fictional. The structure, voice, and reasoning style are exactly what the skill produces.

---

```markdown
---
id: linkedin-2026-06-09-acme-senior-product-manager
researched_date: 2026-06-09
company: Acme Commerce
job_title: Senior Product Manager, Trust & Safety
profiles_visited: 5
cap_used: 5
data_source: tier-1-cache + fresh profiles
---

# Network research — Acme Commerce: Senior Product Manager, Trust & Safety

## Job summary

Acme Commerce — Senior Product Manager, Trust & Safety (Portland, OR, Hybrid). Full-time.

**Area focus:** trust and safety, fraud, abuse mitigation, account integrity, identity, marketplace integrity. The JD calls out experience with bot detection vendors, content-moderation tooling, and cross-functional partnership with Legal and Engineering.

## Tier 1 recap (from `jobs/network-cache.json`, scanned 2026-06-08)

- 1st-degree at Acme Commerce: 28
- 2nd-degree at Acme Commerce: 100+ (paginated floor — true count is several hundred)
- Tier 1 final score: 95 / 100
- Warmest paths from Tier 1: Riley Tanaka (High, ×9 mutuals), Sam Park (High, ×7), Casey Morgan (Medium, ×5)

Three High/Medium-rated 1st-degrees surfaced across the Tier 1 scan as frequent mutuals on Acme cards. Tier 2 below visits the most-promising 2nd-degree candidates each of them could intro to.

## Profiles visited (5)

### 1. Jordan Reyes — Director of Product, Trust & Safety at Acme Commerce

- **Role/team:** Director, Trust & Safety (Apr 2024 – Present). Prior: Sr. PM Fraud (3 yrs), PM Account Integrity (2 yrs). About blurb: "Building the team that keeps Acme's marketplace clean — fraud, abuse, account safety, and the cross-functional muscle around them."
- **Area verdict:** Direct match — Jordan runs the team the role is on. The Sr. PM, Trust & Safety opening almost certainly reports to Jordan or to a Group PM under Jordan.
- **Mutuals:** Riley Tanaka (High), Sam Park (High), and 4 other mutual connections.
- **Best intro path:** Riley Tanaka. Riley and Jordan worked together at the candidate's prior company before both moving to Acme.
- **Suggested outreach context:** "Riley overlapped with Jordan at <prior co> for several years and they've stayed in close professional touch — natural intro."
- Profile: https://www.linkedin.com/in/jordan-reyes-acme/

### 2. Priya Natarajan — Senior Product Manager, Account Integrity at Acme Commerce

- **Role/team:** Sr. PM, Account Integrity (Aug 2023 – Present, 2 yrs 10 mos). Prior: PM Identity & Authentication (3 yrs at a fintech). About blurb mentions specifically: passwordless authentication rollout, ML-based account takeover detection, partnership with Risk Engineering.
- **Area verdict:** Direct match — Priya owns adjacent product surface to the open role. Likely either a peer hire to the open req or a known internal candidate; the intro is informational either way.
- **Mutuals:** Sam Park (High), Casey Morgan (Medium), and 3 other mutual connections.
- **Best intro path:** Sam Park. Sam and Priya are co-leads on a working group that spans both their orgs at Acme.
- **Suggested outreach context:** "Sam works closely with Priya on Acme's identity workstream — would have current read on the team's priorities and whether the Sr. PM open req is a peer or a successor to existing scope."
- Profile: https://www.linkedin.com/in/priya-natarajan-acme/

### 3. Marcus Lin — Principal Product Manager, Marketplace Integrity at Acme Commerce

- **Role/team:** Principal PM (Jan 2022 – Present). Owns marketplace-side fraud and seller-trust workstreams. Prior: Sr. PM at a marketplace-tech vendor; before that, 4 years at a payments processor on chargeback ops.
- **Area verdict:** Adjacent — Marcus's surface is marketplace-side (seller fraud, listing abuse) while the open role appears to be account-side (buyer fraud, account takeover). Same broader org, almost certainly the same Director. Would be a high-signal informational conversation.
- **Mutuals:** Casey Morgan (Medium), 2 other mutual connections.
- **Best intro path:** Casey Morgan.
- **Suggested outreach context:** "Casey is a former colleague of Marcus's from the payments-processor days — relationship is warm but it's been a few years; would frame the ask as 'reconnect + light intro,' not a cold favor."
- Profile: https://www.linkedin.com/in/marcus-lin-acme/

### 4. Avery Kim — Engineering Manager, Trust & Safety Platform at Acme Commerce

- **Role/team:** Eng Manager (Jun 2022 – Present). Leads the platform team Trust & Safety PMs partner with. Prior: Staff Engineer in the same area at a larger commerce platform.
- **Area verdict:** Adjacent — Avery is the engineering counterpart to the open PM role, not a PM peer. Still valuable: the EM partner's read on what kind of PM the team is looking for is usually clearer than what the JD spells out.
- **Mutuals:** Riley Tanaka (High), 1 other mutual connection.
- **Best intro path:** Riley Tanaka (same person who'd intro to Jordan — flag and choose one rather than double-ask).
- **Suggested outreach context:** "Riley could intro to either Jordan or Avery — recommend choosing one. If you want the hiring read, ask for Jordan. If you want the day-to-day-partnership read, ask for Avery."
- Profile: https://www.linkedin.com/in/avery-kim-acme/

### 5. Taylor Brooks — Senior Product Manager, Risk Operations at Acme Commerce

- **Role/team:** Sr. PM, Risk Ops (Mar 2021 – Present). About blurb focuses on chargeback recovery and merchant-side risk tooling. Career arc has been entirely on the financial-risk side of trust.
- **Area verdict:** Off-area — Taylor's surface is financial risk and chargeback recovery; the open role is account integrity and abuse. Sister team at most; different decision-makers. Listing for completeness because Taylor showed up frequently in Tier 1 mutual lists, but I'd deprioritize this one.
- **Mutuals:** Sam Park (High), 6 other mutual connections.
- **Best intro path:** Sam Park (also the warmest path to Priya — recommend prioritizing Priya, who's a direct-area peer).
- **Suggested outreach context:** "Worth knowing Taylor exists at Acme, but I wouldn't ask for an intro here unless the Priya path stalls."
- Profile: https://www.linkedin.com/in/taylor-brooks-acme/

## Top 3 outreach plan

### 1. Ask Riley for an intro to Jordan Reyes

- **Why this one first:** Jordan is the Director the role almost certainly reports up to. Riley is the warmest-rated mutual and overlapped with Jordan at a prior company — natural, low-friction intro.
- **Candidate:** Jordan Reyes, Director of Product, Trust & Safety at Acme Commerce
- **Mutual:** Riley Tanaka (High)
- **Draft opening (to Riley):**

  > Hi Riley, hope you're doing well. I'm exploring a Senior PM, Trust & Safety role at Acme Commerce and noticed you and Jordan Reyes are connected from your <prior co> days. Would you feel comfortable making a quick intro? Happy to send a short blurb you can forward.

### 2. Ask Sam for an intro to Priya Natarajan

- **Why this one second:** Priya owns the directly-adjacent account-integrity surface — a peer-perspective read on the team and the role from someone whose day-to-day touches it.
- **Candidate:** Priya Natarajan, Senior PM, Account Integrity at Acme Commerce
- **Mutual:** Sam Park (High)
- **Draft opening (to Sam):**

  > Hi Sam, hope you're well. I'm looking at a Senior PM, Trust & Safety role at Acme and noticed you work closely with Priya Natarajan on the identity workstream. Would you be up for a quick intro? Would love her read on the team's priorities and where the open req fits.

### 3. Ask Casey for an intro to Marcus Lin

- **Why this one third:** Marcus is the adjacent marketplace-side perspective, and the path goes through a different mutual (Casey, not Riley or Sam) — spreads the asks across three distinct people rather than over-relying on the two High-rated mutuals.
- **Candidate:** Marcus Lin, Principal PM, Marketplace Integrity at Acme Commerce
- **Mutual:** Casey Morgan (Medium — note that it's been a few years since last meaningful contact; consider a brief reconnection note before the ask)
- **Draft opening (to Casey):**

  > Hi Casey — it's been a while, hope life and work are good. Quick ask: I'm exploring a Senior PM role on Acme's Trust & Safety side and noticed you're connected to Marcus Lin (Principal PM, Marketplace Integrity). Any chance you'd be comfortable making a light intro? Happy to send a blurb if so. Either way, would love to catch up properly sometime soon.

## Notes and caveats

- **Area-relevance verdicts are best-effort.** LinkedIn doesn't expose org charts. When I wrote "Direct match" or "Adjacent" I was reading the visible Experience and About sections — not internal team structures.
- **Same mutual appears for multiple candidates.** Riley Tanaka is the warmest path for both Jordan (#1) and Avery (#4). The Top 3 deliberately uses Riley once and routes around her for the rest, so you're asking three distinct people rather than three asks of one person. If you only have appetite for one intro, ask Riley about Jordan.
- **The Casey intro is the lowest-confidence of the three.** Medium-rated, a few years since last meaningful contact. Either reconnect briefly before asking, or skip and lean on the Riley + Sam asks alone.
- **Snapshot.** Profiles drift over weeks. If you're still pursuing this role in a month, re-run `/research-network <rank>` for a fresh read.
```

---

## What this artifact is for

The point of the document above is **not** to read like a research report. It's to be the file you have open in one window with LinkedIn in another window, sending messages to three specific people in the next half hour.

Three design choices worth noting:

1. **The "Why this one first" line** on each Top 3 entry is the reason the user does this step at all. The agent's reasoning about ordering matters more than the ordering itself — because the user is going to disagree with at least one of the picks based on context the agent doesn't have.
2. **The "Best intro path" line** explicitly handles the case where the same mutual is the warmest path for multiple candidates. Asking one person for three intros is over-asking; the skill spreads the asks across distinct mutuals when it can.
3. **The "Off-area" verdict on Taylor Brooks** is the kind of honest call the workflow exists to make. A naive ranker would surface Taylor as relevant (frequent mutual, Acme + "Risk" in the title); the area-verdict step looks at the actual role and downgrades the recommendation. The user benefits from being told "I wouldn't ask here" as much as from being told "I would."

The skill that produces these files is `/research-network` (see `.claude/skills/research-network.md`). Tier 1 (the cheap, automatic network score) lives in `/network-check`.
