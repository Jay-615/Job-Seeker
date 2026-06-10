---
name: research-network
description: Tier 2 deep network research for a single scored job the user is actively considering. Visits up to N (default 10) 2nd-degree connection profiles at the target company, reads their title and role, judges area relevance to the specific posting, and writes jobs/research/<id>-network.md with per-profile findings and a prioritized Top 3 outreach plan with draft opening lines. Expensive — invoke manually, not for the whole inbox.
---

# /research-network — Tier 2 deep network research for one job

You take a rank or a job id, do a focused round of LinkedIn profile reading at the target company, and produce a polished outreach-planning document the user can act on directly.

This is the **on-demand, expensive** counterpart to `/network-check` (Tier 1). `/network-check` runs once per company per `/score-jobs` pass and stays cheap; `/research-network` runs *manually* against one job at a time, visits up to N individual 2nd-degree profiles, and burns real tokens to do it. Don't invoke it for the whole inbox. Invoke it for jobs the user is seriously pursuing.

Browser logic mirrors `/network-check`'s patterns — same LinkedIn People search URL, same `main.innerText` parsing approach, same loud-failure posture. When LinkedIn's UI changes, update both skills.

---

## Parameters

- **rank** (integer like `1`) — looked up in the most recent `jobs/scored/dashboard-*.md`, OR
- **job id** (string like `linkedin-2026-05-26-nike-director-lp-ops`) — used to find `jobs/scored/<id>.md` directly.

If no argument is given, list the ⭐ jobs from the most recent dashboard and ask via `AskUserQuestion` which one to research. If the dashboard is missing, bail with:

> No scored dashboard found. Run `/score-jobs` first, then come back with a rank or id.

---

## Step 0 — Preflight

1. **Files in place.** Confirm `config/settings.md`, `config/target_companies.csv`, and at least one `jobs/scored/*.md` exist. If any is missing, bail with a clear message pointing to `/setup` or `/score-jobs`.
2. **Read settings.** From `config/settings.md`:
   - `research_profile_cap` under **Network Proximity** (default `10` if missing or unparseable).
   - User's name (under **User Profile**) — used for the draft opening lines.
3. **Today.** Compute `today_iso = YYYY-MM-DD` for the output frontmatter.

Do NOT start Chrome yet — defer the browser preflight to Step 4, after all file reads and the cost-warning prompt.

---

## Step 1 — Resolve the argument to a scored job

### 1a. Argument is a string with a hyphen (a job id)

1. Look for `jobs/scored/<id>.md` directly. If it doesn't exist, bail:

   > No scored file found for id `<id>`. Available scored files: <list 5>. Re-run with one of those ids or a dashboard rank.

### 1b. Argument is an integer (a rank)

1. Find the most recent dashboard: list `jobs/scored/dashboard-*.md`, sort by filename descending (the date sorts lexically), pick the first.
2. Parse the dashboard's table. Rows start with `| <rank> | <company> | <title> | ...`. Match the argument against the `rank` column.
3. From the matching row, take `company` and `title`. Scan `jobs/scored/*.md` (excluding `dashboard-*.md`), read each one's frontmatter, and match on `company` + `title` to recover the `id`.
4. If the rank is out of range, bail:

   > Rank `<n>` not found in `jobs/scored/dashboard-<date>.md` (it has <m> rows). Re-check and try again.

### 1c. No argument

List ⭐ jobs from the latest dashboard with their ranks and ask the user via `AskUserQuestion` which to research. If no ⭐ rows, list all rows.

---

## Step 2 — Read the scored + inbox files

Read both:

- `jobs/scored/<id>.md` — for the summary score, network-proximity narrative, and the agent's prior reasoning about target/area
- `jobs/inbox/<id>.md` — for the company name, job title, and the JD body

From these, extract and hold in memory:

- `company` (verbatim, used as the cache key)
- `job_title` (the role title, used as the area anchor)
- `job_area_keywords` — derive a short list (3-8 terms) from the job title and the first ~500 words of the JD. Examples: a "Director, Global Digital Loss Prevention Operations" role at Nike yields keywords like `loss prevention`, `fraud`, `digital ops`, `e-commerce abuse`, `bot detection`. A "Senior PM, E-commerce Platform" role yields `e-commerce`, `platform`, `product management`, `checkout`, `growth`. You're free to be liberal — the keywords are a fuzzy area filter, not a regex.
- `job_summary_one_line` — one short line, e.g., "Nike — Director, Global Digital Loss Prevention Operations (Beaverton, OR, Hybrid)"

---

## Step 3 — Read the Tier 1 cache and the connections file

1. Open `jobs/network-cache.json`. Find the entry keyed by `company` (case-insensitive on the keys). If no entry exists, bail with:

   > No Tier 1 network analysis found for `<company>` in `jobs/network-cache.json`. Run `/score-jobs` (or `/network-check <company>` directly) first so the candidate-pool data is available.

   We deliberately do not refresh Tier 1 here — that's `/network-check`'s job. M9 builds on M4's data.

2. From the cache entry, hold in memory:
   - `linkedin_company_id`
   - `first_degree_count`, `second_degree_count`
   - `first_degree_mutuals_top` — the ranked list of 1st-degrees who appeared as mutuals during the Tier 1 scan, each with `appears_as_mutual_count` and `rating`
   - `warmest_paths` — the names already prioritized for intros

3. Open `config/connections.csv` if it exists. Hold a `name -> rating` map for fast lookup. If the file is missing, note in `data_source` that all bonuses will read as "unrated" and the warmest-paths section of the output will be base-only.

---

## Step 4 — Cost transparency + go/no-go prompt

Before any browser work, print a clear cost preview to the user and ask once whether to proceed. Use `AskUserQuestion`.

```
/research-network — cost preview

Target: <job_summary_one_line>
Plan: visit up to <cap> 2nd-degree profiles on LinkedIn at <company>.
Estimated token cost: ~5-15K tokens per profile × <cap> profiles
                    = ~<5*cap>K-<15*cap>K tokens total for this run.
Tier 1 cache shows: <first_degree_count> 1st-degrees and <second_degree_count> 2nd-degrees at <company>.

This is a manual, on-demand command. Use it for jobs you're actively pursuing,
not the whole inbox.

Proceed?
```

`AskUserQuestion` options:

- **Yes, proceed** — continue to Step 5.
- **Yes, but lower the cap to 5** — cap = `min(cap, 5)`, continue.
- **No, cancel** — exit cleanly, write nothing.

If the user cancels, print one line confirming nothing was written and stop.

---

## Step 5 — Browser preflight (same as `/network-check`)

1. Call `mcp__Claude_in_Chrome__list_connected_browsers`. If empty, bail:

   > I can't reach Chrome. Make sure the Claude in Chrome extension is installed and connected, then re-run. If you've never set this up, run `/setup` first.

2. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`.
3. Navigate to `https://www.linkedin.com/feed/`. If you land on a login screen, bail:

   > Chrome is connected but you don't appear to be logged into LinkedIn. Log in in Chrome, then re-run.

   Don't try to log in for the user.

Pace all subsequent navigation at human speed — a few seconds between actions. If LinkedIn throws a CAPTCHA, security check, or rate-limit page at any point, bail loudly with the URL where it happened. Never try to solve a CAPTCHA.

---

## Step 6 — Build the candidate pool

The goal: assemble ~2-3× the cap in candidates so you can rank and pick the best N. We want candidates who are both **in or near the job's area** (from headline) and **reachable via a warm 1st-degree mutual** (rating ≥ Medium when available).

1. Construct the 2nd-degree People search URL (same as Tier 1 Step 4):

   ```
   https://www.linkedin.com/search/results/people/?currentCompany=%5B%22<linkedin_company_id>%22%5D&network=%5B%22S%22%5D
   ```

2. Navigate, wait ~2-3 seconds for React render.

3. Use `mcp__Claude_in_Chrome__javascript_tool` to extract the cards, including their profile URLs this time (Tier 1 didn't bother with profile URLs since it only counts and reads excerpts). **Critical:** profile URLs must be paired to cards by **name-match anchor scoping**, not by document order. Each card's DOM contains anchors for the candidate *and* for the named mutual connections — document-order pairing produces silently wrong URLs (verified in the field: 1/9 correct vs. 9/9 correct with name-match). Reference snippet:

   ```javascript
   (function() {
     window.scrollTo(0, document.body.scrollHeight);
     const main = document.querySelector('main');
     if (!main) return { cards: [] };
     const text = main.innerText.split('\n').map(l => l.trim()).filter(Boolean);

     // First pass: parse cards from text, without URLs.
     // Card boundaries: a line matching /^•\s*(1st|2nd|3rd)/i appears right after each name.
     const text_cards = [];
     for (let i = 0; i < text.length; i++) {
       if (/^•\s*(1st|2nd|3rd)/i.test(text[i])) {
         const name = text[i - 1];
         const headline = text[i + 1] || '';
         let mutual = null;
         for (let j = i + 1; j < Math.min(i + 8, text.length); j++) {
           if (/mutual connection/i.test(text[j])) { mutual = text[j]; break; }
         }
         text_cards.push({ name, headline, mutual });
       }
     }

     // Second pass: pair each text-card to its profile URL by matching the candidate's name
     // against /in/* anchor innerText. Mutual-connection anchors share the card DOM, so
     // document-order pairing was wrong (it leaked mutual URLs into candidate slots).
     const all_anchors = Array.from(main.querySelectorAll('a[href*="/in/"]'));
     const enriched = text_cards.map(c => {
       let url = null;
       for (const a of all_anchors) {
         const atext = (a.innerText || '').trim();
         // startsWith tolerates trailing degree/verification badges that LinkedIn nests inside the anchor.
         if (atext === c.name || atext.startsWith(c.name + '\n') || atext.startsWith(c.name + ' ')) {
           url = a.href.split('?')[0].replace(/\/$/, '');
           break;
         }
       }
       return { ...c, profile_url: url };
     });

     // Defensive: bail noisily if any card couldn't be matched to a URL.
     const missing_url_count = enriched.filter(c => !c.profile_url).length;
     return { cards: enriched, missing_url_count };
   })()
   ```

   If `missing_url_count > 0`, log which cards lacked URLs (by name) and treat those as unrouteable — drop them from the candidate pool rather than visiting the wrong profile. A confident-looking wrong recommendation costs the user social capital; an honest skip does not.

4. **Pagination.** Page 1 typically yields ~10 cards. Score them with the heuristic in Step 6a; if you have at least `cap × 2` candidates after page 1, stop. Otherwise paginate to page 2 (append `&page=2` to the URL), wait, re-run the snippet. Stop at page 3 max — you do not need a perfect candidate pool; you need a *good enough* one to fill the cap.

5. **Parse mutuals.** Use the same four-shape parser as `/network-check` Step 4 to pull explicitly named 1st-degree mutuals from each card. Build per-card: `mutual_names` (list of strings).

### Step 6a — Rank candidates

For each card, compute a composite score:

- **headline_relevance** (0-3): count how many of the `job_area_keywords` appear (case-insensitive substring) in the headline. Cap at 3.
- **mutual_warmth** (0-3): max rating across the card's `mutual_names`. `High` = 3, `Medium` = 2, `Low` or `Unrated` = 1, no rated mutual = 0.
- `composite = headline_relevance * 2 + mutual_warmth`

Sort cards by `composite` descending, tie-broken by presence of any mutual at all (cards with named mutuals come first). Take the top `cap` cards as your candidates. If fewer than `cap` cards have `headline_relevance >= 1`, fill the remainder with the next-warmest cards even if their headline doesn't match — the per-profile read may still reveal area-relevant work.

---

## Step 7 — Visit each candidate profile

For each candidate, in ranked order:

1. Navigate to `candidate.profile_url`. Wait ~3-4 seconds for the React-rendered Experience section to populate.

2. Use `javascript_tool` to extract structured data. **Two parser details that bit the field-tested smoke run:** the headline skip-list must drop degree markers (`· 2nd`, `· 3rd`) — otherwise the parser returns a literal `"· 2nd"` as the headline — and the About-section search must be **scoped above the Experience section** to avoid pulling the LinkedIn footer's "About" link as the About block when the profile has no About.

   ```javascript
   (function() {
     const main = document.querySelector('main');
     if (!main) return null;
     const text = main.innerText.split('\n').map(l => l.trim()).filter(Boolean);

     // Top-card: name is usually the first non-empty text line; headline is the line below it.
     // LinkedIn inserts noise lines between the name and the real headline:
     //   - pronoun chips ("He/Him", "She/Her", "They/Them")
     //   - verification badges ("Verified", "·")
     //   - the candidate's degree marker ("· 2nd", "· 3rd", "• 1st") — appears right below the name
     const name = text[0] || null;
     let headline = null;
     for (let i = 1; i < Math.min(8, text.length); i++) {
       const s = text[i];
       if (/^(he\/him|she\/her|they\/them|·|verified)$/i.test(s)) continue;
       if (/^•?\s*(1st|2nd|3rd)$/i.test(s)) continue;
       if (/^·\s*(1st|2nd|3rd)$/i.test(s)) continue;
       if (/^·$/.test(s)) continue;
       headline = s;
       break;
     }

     // Find the Experience section; collect ~6-10 lines under it. LinkedIn renders each role as
     // a stack of: <title>, <company> · <employment type>, <date range>, <location>, <description...>.
     const expIdx = text.findIndex(l => /^Experience$/i.test(l));
     let experience_block = [];
     if (expIdx >= 0) {
       experience_block = text.slice(expIdx + 1, expIdx + 30);
     }

     // About section — scope the search to BEFORE the Experience section so we don't accidentally
     // match the LinkedIn footer's "About" link when the profile has no About of its own.
     // (Profile body's "About" always appears above Experience; footer's "About" is far below.)
     const aboutSearchSlice = expIdx >= 0 ? text.slice(0, expIdx) : text.slice(0, 30);
     const aboutLocal = aboutSearchSlice.findIndex(l => /^About$/i.test(l));
     const about_block = aboutLocal >= 0 ? aboutSearchSlice.slice(aboutLocal + 1, aboutLocal + 8) : [];

     return { name, headline, experience_block, about_block };
   })()
   ```

3. From the result, identify:

   - **current_title** — the topmost entry in `experience_block` that names a role at `<company>` (or a match on the company name within ~3 lines below the title). If the candidate has multiple current roles, take the one at the target company.
   - **role_or_team** — the next 2-4 lines under the current title, looking for team hints ("Loss Prevention", "Brand Marketing", "Platform Engineering"), the role's bullet-point first sentence, or the About blurb if Experience is sparse.

4. **Judge area relevance.** Compare `current_title` + `role_or_team` against `job_area_keywords` and the job's title. Three possible verdicts:

   - **Direct match** — the person works in the same area as the posting (e.g., the JD is for LP Ops Director and the person is a Sr. Manager in Loss Prevention).
   - **Adjacent** — different team but same broader function (e.g., the JD is in Loss Prevention and the person is in Fraud Ops, or in Digital Ops).
   - **Off-area** — clearly unrelated (e.g., JD is in LP and the person is in Brand Marketing). **State this honestly**, don't bury it.

   Be explicit and conservative. If you can't tell from the visible Experience/About text, write "unclear from profile" rather than guessing. This is the value of the manual research — honest area judgment, not hopeful pattern-matching.

5. **Build the per-profile finding object**:

   ```
   {
     name, profile_url, current_title, role_or_team,
     area_verdict: "Direct match" | "Adjacent" | "Off-area" | "Unclear",
     area_reasoning: "<one-sentence why>",
     mutual_names: [<names from card>],
     mutual_ratings: [{name, rating}],   // looked up against connections.csv map
     best_intro: <name>,                  // the highest-rated mutual; tie-break by appears_as_mutual_count from Tier 1 cache
     suggested_outreach_context: "<one sentence the user could borrow when asking for the intro>"
   }
   ```

Pace requests: 3-4 seconds between profile loads. If you hit a CAPTCHA or rate-limit page mid-walk, **stop**, write whatever you have so far to the output file with a clear note, and report honestly. Better a partial-but-honest research doc than no doc at all.

---

## Step 8 — Build the Top 3 outreach plan

From the per-profile findings:

1. Rank by `(area_verdict_weight, mutual_warmth, headline_relevance)`:
   - area_verdict_weight: Direct match = 3, Adjacent = 2, Unclear = 1, Off-area = 0
   - mutual_warmth: from the best_intro's rating (High = 3, Medium = 2, Low/Unrated = 1, none = 0)
   - headline_relevance: as computed in Step 6a
2. Take the top 3 by this composite. If multiple findings share the same best_intro, prefer keeping the top one and skip down for the next slot — you want **3 distinct intro paths**, not 3 asks of the same person.
3. For each of the Top 3, draft a brief opening line the user can adapt. Voice: warm but professional, ~25-40 words. Use the **user's first name from settings**, and refer to the mutual contact and the target person by first name. Don't fabricate a relationship the user doesn't have. Don't over-claim the candidate's role — only state what you actually saw on their profile.

   Template:

   > Hi <mutual_first>, hope you're well. I'm exploring a <job_title> role at <company> and noticed you're connected to <candidate_first>, who's in <role_or_team>. Would you feel comfortable making a quick intro? Happy to send a short blurb you can forward.

   Adapt the template per case. If `mutual_rating == "High"`, the language can be more casual ("hope all's good"). If `Medium`, more formal and include a reconnection line ("it's been a while — would love to reconnect"). If only `Low`/`Unrated`, flag that the user may want to warm up the mutual relationship first rather than ask for an intro cold.

---

## Step 9 — Write `jobs/research/<id>-network.md`

Use this structure exactly. The file is visible both to the user (their outreach planning doc) and potentially to portfolio viewers (if it gets anonymized into `docs/sample-network-research.md` later, per the spec), so the polish matters.

```markdown
---
id: {{id}}
researched_date: {{today_iso}}
company: {{company}}
job_title: {{job_title}}
profiles_visited: {{actual_count}}
cap_used: {{cap}}
data_source: {{ "tier-1-cache + fresh profiles" | "tier-1-cache + fresh profiles (connections.csv missing — intro warmth unrated)" }}
---

# Network research — {{company}}: {{job_title}}

## Job summary

{{job_summary_one_line}}

Area focus: {{comma-separated job_area_keywords}}

## Tier 1 recap (from `jobs/network-cache.json`, scanned {{cache_scanned_date}})

- 1st-degree at {{company}}: {{first_degree_count}}
- 2nd-degree at {{company}}: {{second_degree_count}}
- Tier 1 final score: {{final_score}}/100
- Warmest paths from Tier 1: {{comma list, or "(none — connections.csv missing or no High/Medium mutuals)"}}

## Profiles visited ({{actual_count}})

### 1. {{candidate.name}} — {{candidate.current_title}}

- **Role/team:** {{candidate.role_or_team}}
- **Area verdict:** {{candidate.area_verdict}} — {{candidate.area_reasoning}}
- **Mutuals:** {{names with ratings, e.g. "Sarah Smith (High), Pat Lee (Medium)"}}
- **Best intro path:** {{candidate.best_intro}}
- **Suggested outreach context:** {{candidate.suggested_outreach_context}}
- Profile: {{candidate.profile_url}}

### 2. {{candidate.name}} — ...

(repeat for each profile, in ranked order)

## Top 3 outreach plan

### 1. Ask {{mutual_first}} for an intro to {{candidate_first}}

- Why this one first: {{one short sentence — area fit + warmest intro}}
- Candidate: {{candidate.name}}, {{candidate.current_title}}
- Mutual: {{mutual.name}} ({{mutual.rating}})
- Draft opening (to {{mutual_first}}):
  > {{draft opening line, 25-40 words}}

### 2. Ask {{mutual_first}} for an intro to {{candidate_first}}

(same structure)

### 3. Ask {{mutual_first}} for an intro to {{candidate_first}}

(same structure)

## Notes and caveats

- Area-relevance verdicts are best-effort judgments from each profile's visible Experience and About sections. LinkedIn doesn't expose org charts — when I wrote "Unclear" I genuinely couldn't tell.
- If `connections.csv` was missing or sparse at the time of this run, the intro-warmth ratings will read "Unrated" and the Top 3 plan should be treated as a starting point, not a final ranking. Run `/build-connections` and `/rate-connections` to enrich future runs.
- This is a snapshot — LinkedIn profiles change. Re-run `/research-network {{rank_or_id}}` after a few weeks if you're still pursuing this role.
```

Write the file. Create `jobs/research/` if it doesn't already exist (the M0 scaffolding should have created it, but defensive code is fine here).

---

## Step 10 — Report to the user

Print a short, human-readable summary to the console:

```
/research-network — done.

Job: {{job_summary_one_line}}
Profiles visited: {{actual_count}} of {{cap}}
Direct-match candidates: {{count}}    Adjacent: {{count}}    Off-area: {{count}}    Unclear: {{count}}

Top 3 outreach plan:
  1. Ask {{mutual_first}} to intro you to {{candidate.name}} ({{candidate.current_title}})
  2. Ask {{mutual_first}} to intro you to {{candidate.name}} ({{candidate.current_title}})
  3. Ask {{mutual_first}} to intro you to {{candidate.name}} ({{candidate.current_title}})

Full report: jobs/research/{{id}}-network.md
Approx token cost this run: ~{{estimated total}}K (manual command — invoke only for jobs you're seriously pursuing).
```

Include the cost figure honestly. If you bailed mid-walk (CAPTCHA, rate-limit), say so in this summary and link the file even though it's partial.

---

## Failure modes — what to do when LinkedIn pushes back

- **Chrome not connected / not logged in** → bail at preflight. Same wording as `/network-check`.
- **No Tier 1 cache entry for this company** → bail at Step 3. Point to `/score-jobs` or `/network-check <company>`.
- **Job id or rank not found** → bail at Step 1 with the available range / available ids.
- **CAPTCHA or security-check page mid-walk** → stop visits, write a partial report with a clear `data_source` note ("partial — stopped at profile N of cap M due to LinkedIn security check"), report honestly, and recommend the user complete the check in Chrome before re-running.
- **Rate-limit page** ("You're searching too fast") → stop, write whatever's complete, and recommend a 1-hour cool-down. Do not retry inside this session.
- **Profile page can't be parsed** (top card or Experience section missing) → record the candidate with `area_verdict: "Unclear"` and `area_reasoning: "could not parse profile — LinkedIn UI may have shifted; see /network-check for the latest field-tested selectors"`, then continue to the next candidate.

Loud, honest failure beats a confident-looking wrong recommendation. Outreach is a relationship — if the agent recommends asking the wrong person, that costs the user social capital.

---

## What this skill must NOT do

- **Do not re-run Tier 1.** Tier 1 lives in `/network-check`. If the cache is stale or missing, point the user at `/network-check <company>` and bail. M9 builds on M4's data, doesn't duplicate it.
- **Do not message anyone on LinkedIn.** Drafting the opening line is the entire output. The user sends the message themselves, in their own voice, from their own account.
- **Do not visit the same profile twice in one run.** De-dupe by `profile_url`.
- **Do not exceed `research_profile_cap`.** Even if there's budget to visit more, stop at the cap. The cap is the user's cost guardrail.
- **Do not fabricate roles or teams.** If the profile doesn't say the person works in Loss Prevention, don't write "works in Loss Prevention." Write what the profile actually says, or "Unclear".

---

## LinkedIn UI dependencies (for future maintenance)

This skill assumes the following about LinkedIn's UI as of June 2026. When LinkedIn changes things, update this list and the equivalent block in `network-check.md`:

1. **2nd-degree People search URL** — `https://www.linkedin.com/search/results/people/?currentCompany=%5B%22<id>%22%5D&network=%5B%22S%22%5D`. Pagination via `&page=<n>`. Same shape as `/network-check`.

2. **Result-card profile URLs** — each card contains anchors not only for the candidate but also for the named **mutual connections** rendered in the "X, Y and N other mutual connections" text. Document-order pairing breaks because mutual anchors are interleaved with candidate anchors (field-verified: produces 1/9 correct URLs on a real Nike 2nd-degree page). The correct pairing is **name-match anchor scoping**: for each text-parsed card name, walk `main.querySelectorAll('a[href*="/in/"]')` once and pick the first anchor whose `innerText` starts with the candidate name (tolerating trailing degree/verification badges). Carry a `missing_url_count`; if any card lacks a matched URL, drop it from the candidate pool rather than visiting the wrong profile. See Step 6 for the implementation.

3. **Mutual-connection excerpt shapes** — four forms, same as in `/network-check`:
   - `"<Name> is a mutual connection"`
   - `"<Name1> and <Name2> are mutual connections"`
   - `"<Name1>, <Name2>, and <Name3> are mutual connections"`
   - `"<Name1>, <Name2> and N other mutual connections"`

4. **Individual profile page** — `https://www.linkedin.com/in/<slug>/`. We rely on these sections being labeled with exactly these headings as line-text within `<main>.innerText`:
   - `Experience` — followed by a stack of roles. Each role's first line is the title; the next 1-3 lines name the company, employment type, dates, and location.
   - `About` — short blurb. Optional; not all profiles have one.

   If LinkedIn renames either heading or stops rendering them as plain text in the main element, the parser breaks; the right response is to bail per-profile (record "Unclear"), not to guess.

   **About-section search must be scoped above the Experience section.** A global `findIndex(/^About$/i)` falls through to the LinkedIn footer's "About" link when the profile has no About of its own, and pulls in `["Accessibility", "Talent Solutions", "Community Guidelines", ...]` as the "About" block. The profile body's About always appears above Experience; the footer's About appears far below. Slice to `text[0 .. expIdx]` before searching.

5. **Top-card name and headline** — the name is the first non-empty line in `<main>.innerText`. The headline is the next non-noise line below it. LinkedIn inserts these noise lines between the name and the real headline, and the parser must skip all of them:
   - Pronoun chips: `He/Him`, `She/Her`, `They/Them`
   - Verification: `Verified`, bare `·`
   - **Degree markers: `· 2nd`, `· 3rd`, `• 1st`** — the candidate's own degree relative to the viewer, rendered right below the name. Field-verified miss in the smoke run: without this skip, the parser returned `"· 2nd"` as the headline.

6. **No org-chart data.** LinkedIn does not expose which team a person sits on within a company. Our area-relevance judgment is best-effort from job title + Experience description + About blurb only. This is the structural limit on Tier 2's accuracy and is worth being honest about in the output.

If a maintainer needs to update any of this, the search URL is in Step 6, the per-profile parser is in Step 7, and all browser interaction is confined to Steps 5, 6, and 7. Nothing browser-related lives elsewhere in this file.
