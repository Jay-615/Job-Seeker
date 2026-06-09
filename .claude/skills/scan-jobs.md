---
name: scan-jobs
description: Find new job postings across LinkedIn, Indeed, Dice, and selected company career pages. Applies hard filters (title category, seniority, location, comp floor, job type), dedupes against jobs/seen.json, writes survivors as markdown files to jobs/inbox/, and updates jobs/career-page-cache.json. Honors cost guardrails in settings.md for career-page scans.
---

# /scan-jobs — find new postings, filter, dedupe

You scan four sources for new job postings, apply the hard filters in `docs/scoring-methodology.md` (and Section 6 of the project spec), dedupe against `jobs/seen.json`, and write each survivor as a markdown file to `jobs/inbox/`. Survivors flow into `/score-jobs` next.

This is the most complex skill in the project. It is also the most expensive — career-page scans alone can run 100K–300K tokens. Honor the cost guardrails in `settings.md` strictly. When something looks wrong (CAPTCHA, rate limit, parser miss), fail loudly rather than guessing.

The skill takes no parameters. It runs the full pipeline.

---

## Step 0 — Preflight

Before touching any source:

1. Read `config/criteria.md`, `config/settings.md`, and `config/target_companies.csv`. If any of the three is missing, bail with:

   > Missing config — run `/setup` first. (Missing: <list of files>.)

2. From `settings.md` parse the **Sources** block (`linkedin`, `indeed`, `dice`, `company_career_pages` — `true`/`false` each) and the **Cost Guardrails** block (`priority_filter`, `max_pages_per_run`, `cache_results_days`). Defaults if unparseable: linkedin/indeed/dice/career-pages all `true`, priority_filter `High`, max_pages_per_run `10`, cache_results_days `3`.

3. From `criteria.md` parse: the **Roles** list (job titles to search for), the **Locations** list (used for matching, not just scoring), the **Seniority** rules, the **Compensation floor**, the **Job types**.

4. Load (or create empty `{}`) for both `jobs/seen.json` and `jobs/career-page-cache.json`.

5. If any LinkedIn/Indeed/Dice source is enabled, run the Chrome preflight (see **Browser preflight** below). If Chrome is unavailable, do not bail on the whole run — disable the affected sources, note it in the run summary, and continue with the others. Career-page scans via Lever/Greenhouse public JSON APIs do not need Chrome at all.

6. Set `today_iso = date(today)` as `YYYY-MM-DD`. Use it for `fetched_date` and the stable id prefix throughout the run.

---

## Browser preflight

Used by LinkedIn, Indeed, Dice, and any career-page scan that falls through to Chrome (no ATS API match, no clean public list).

1. Call `mcp__Claude_in_Chrome__list_connected_browsers`. If empty, mark Chrome as unavailable and bail out of Chrome-dependent sources with:

   > Chrome MCP isn't connected. Skipping LinkedIn / Indeed / Dice / Chrome-fallback career pages. Connect Claude in Chrome (re-run `/setup` if needed) and re-run `/scan-jobs` to pick those sources back up.

2. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true` to get a working tab. Reuse this tab across the run.

3. **LinkedIn — strict login check.** Navigate to `https://www.linkedin.com/feed/`. If it redirects to a login page, mark LinkedIn as unavailable for this run with:

   > Chrome is connected but you're not logged into LinkedIn. Log in in Chrome and re-run.

   LinkedIn is strict because their job-search results behind login are richer and the anti-bot is the strictest of the three.

4. **Indeed and Dice — soft login check (don't block).** These two sites work anonymously. Being logged in is recommended (lower CAPTCHA risk on Indeed especially, and persisted saved searches), but not required. The check happens *opportunistically* the first time the skill navigates to each site in Steps 2 and 3 — it doesn't add navigation here in preflight.

5. Pace yourself. Wait ~2–3 seconds after each navigation. Never fire more than one browser action per 1–2 seconds when paging through search results.

---

## Step 1 — LinkedIn (if `sources.linkedin: true`)

LinkedIn job search is per-role × per-location. Use the user's logged-in session via Chrome MCP. Don't try to fetch via curl — LinkedIn requires login cookies and aggressive bot detection.

### 1a. Build search queries

For each role in `criteria.md` Roles, and for each location-mode pair you care about, build a LinkedIn jobs URL:

- Geographic search (Portland metro): `https://www.linkedin.com/jobs/search/?keywords=<URL-encoded role>&location=Portland%2C%20Oregon%2C%20United%20States&distance=25&f_TPR=r604800` — `f_TPR=r604800` is "past week" (604800 seconds), which is the sweet spot for weekly scans.
- US Remote: `https://www.linkedin.com/jobs/search/?keywords=<URL-encoded role>&location=United%20States&f_WT=2&f_TPR=r604800` — `f_WT=2` is the Remote workplace-type filter.
- Hybrid Portland HQ falls naturally out of the Portland search above; LinkedIn returns Hybrid postings there.

You end up with `len(roles) × 2` queries — for Jay's current criteria, that's 3 × 2 = 6 search loads. Each load is one navigation. Don't paginate beyond the first page unless the first page returned the maximum result count (usually 25); the post-filter survival rate is low enough that page-1 already covers the realistic weekly delta.

### 1b. For each query

1. Navigate to the URL. Wait ~2–3 seconds for React to render.
2. Use `mcp__Claude_in_Chrome__javascript_tool` to extract candidate cards:

   ```javascript
   (function() {
     window.scrollTo(0, document.body.scrollHeight);
     const out = [];
     // Each result card carries the canonical /jobs/view/<id> link.
     const links = Array.from(document.querySelectorAll('a[href*="/jobs/view/"]'));
     const seen = new Set();
     for (const a of links) {
       const m = a.href.match(/\/jobs\/view\/(\d+)/);
       if (!m) continue;
       const id = m[1];
       if (seen.has(id)) continue;
       seen.add(id);
       const card = a.closest('li') || a.closest('div');
       const text = (card ? card.innerText : '').split('\n').map(s => s.trim()).filter(Boolean);
       // First two non-empty lines are usually title then company.
       out.push({
         id, url: `https://www.linkedin.com/jobs/view/${id}`,
         title: text[0] || '', company: text[1] || '',
         location: text[2] || '', text_excerpt: text.slice(0, 8).join(' | ')
       });
     }
     return out;
   })()
   ```

3. The card excerpt holds enough to apply title-category, seniority, and basic location filters. **Don't open every job detail page** — that's the expensive trap. Open the JD only for cards that pass the cheap filters AND aren't already in `seen.json`.

### 1c. Fetch survivor JDs

For each card that passes the cheap filters and is new:

1. Navigate to its `/jobs/view/<id>` URL. Wait ~2 seconds.
2. Extract the JD body and metadata with one `javascript_tool` call:

   ```javascript
   (function() {
     const main = document.querySelector('main') || document.body;
     const lines = (main.innerText || '').split('\n').map(s => s.trim()).filter(Boolean);
     const titleEl = document.querySelector('h1');
     const title = titleEl ? titleEl.innerText.trim() : (lines[0] || '');
     // Posted-time excerpt is usually a phrase like "Posted 3 days ago" near the top.
     const postedLine = lines.find(l => /posted|reposted/i.test(l)) || '';
     // Salary / comp text appears in a "Pay range provided by employer" block when disclosed.
     const compLine = lines.find(l => /\$[\d,]+(\.\d+)?\s*(\/\s*(hr|hour|yr|year))?(\s*-\s*\$[\d,]+)?/i.test(l)) || '';
     // The JD body is the big block of text after the meta header.
     const aboutIdx = lines.findIndex(l => /^about the job$/i.test(l));
     const body = aboutIdx >= 0 ? lines.slice(aboutIdx + 1).join('\n\n') : lines.slice(0, 200).join('\n\n');
     return { title, postedLine, compLine, body, all_chars: (main.innerText || '').length };
   })()
   ```

3. Pace: wait 2–3 seconds between consecutive job-detail loads. Never more than ~15 detail loads per minute. If you spot a CAPTCHA or rate-limit interstitial, stop LinkedIn scanning for this run and note it in the run summary.

### 1d. Normalize each candidate into the common record

Build a `RawPosting` (Python-ish dict shape used internally) per survivor:

```
{
  source: "linkedin",
  url: "https://www.linkedin.com/jobs/view/<id>",
  posted_date: <YYYY-MM-DD or "unknown">,
  fetched_date: today_iso,
  company: <string>,
  title: <string as posted>,
  location: <string as posted>,
  mode: <"Remote" | "Hybrid" | "Onsite" | "unknown">,
  type: <"full-time" | "contract" | "contract-to-hire" | "part-time" | "unknown">,
  comp_text: <verbatim comp text from the page, or "">,
  comp_disclosed: <bool>,
  comp_floor_met: <true | false | null>,
  jd_body: <full JD text>,
  jd_key_requirements: [<bullet strings extracted>],
  jd_nice_to_haves: [<bullet strings extracted>],
}
```

For `mode`: read it off the JD's location/workplace line. "Remote" → Remote; "Hybrid" → Hybrid; anything else → Onsite (or "unknown" if absent).

For `type`: infer from the title and JD. Most LinkedIn cards expose a Job Type chip near the top.

For `comp_*`: parse the comp line. Salary "$185,000 - $235,000" → floor $185,000 annual. Hourly "$90/hr" → floor $90/hr. Empty comp line → `comp_disclosed: false`, `comp_floor_met: null`.

---

## Step 2 — Indeed (if `sources.indeed: true`)

Indeed has aggressively shut down public RSS feeds and publisher API access. The least-fragile permitted path is Chrome MCP against their public search page. Be ready for them to push a CAPTCHA — if they do, skip the source for this run and report it.

### 2a. Build search queries

For each role in `criteria.md` Roles:

- Portland metro: `https://www.indeed.com/jobs?q=<URL-encoded role>&l=Portland%2C+OR&radius=25&fromage=7`
- US Remote: `https://www.indeed.com/jobs?q=<URL-encoded role>&l=Remote&fromage=7`

`fromage=7` = past 7 days. Same role × location matrix as LinkedIn.

### 2b. For each query

1. Navigate. Wait ~3 seconds.
2. **Opportunistic login check (first query only).** On the first Indeed page of the run, peek at the nav before extracting cards:

   ```javascript
   (function() {
     const top = (document.body.innerText || '').split('\n').slice(0, 40).map(s => s.trim()).join(' | ').toLowerCase();
     // Logged-out Indeed shows "Sign in" prominently in the top nav.
     // Logged-in Indeed surfaces "My jobs" / "Messages" / a profile menu instead.
     const looks_logged_out = /\bsign in\b/.test(top) && !/my jobs|profile|messages/.test(top);
     return { looks_logged_out };
   })()
   ```

   If it looks logged out, add a one-line note to the run summary (do NOT halt):

   > Indeed: scanning anonymously. CAPTCHA risk is higher than logged-in. To reduce that, log into indeed.com in your Chrome window before the next `/scan-jobs` run.

   Then continue normally.
3. Check for a CAPTCHA / "verify you are human" interstitial. Heuristic: look for `/verify you are (human|a human)|cloudflare|press and hold|are you a robot/i` in the page text, OR a missing card list when one was expected. If present, follow the shared **CAPTCHA handling** flow below (one section after Failure modes) — prompt the user, retry once, then either continue or skip the source based on their choice.
4. Extract cards with `javascript_tool`:

   ```javascript
   (function() {
     window.scrollTo(0, document.body.scrollHeight);
     const cards = Array.from(document.querySelectorAll('a[id^="job_"], a[data-jk]'));
     const out = [];
     const seen = new Set();
     for (const a of cards) {
       const jk = a.getAttribute('data-jk') || (a.id || '').replace(/^job_/, '');
       if (!jk || seen.has(jk)) continue;
       seen.add(jk);
       const root = a.closest('li') || a.closest('div');
       const text = (root ? root.innerText : '').split('\n').map(s => s.trim()).filter(Boolean);
       out.push({
         jk,
         url: `https://www.indeed.com/viewjob?jk=${jk}`,
         title: text[0] || '',
         company: text[1] || '',
         location: text[2] || '',
         text_excerpt: text.slice(0, 8).join(' | ')
       });
     }
     return out;
   })()
   ```

5. As with LinkedIn, apply cheap filters to the card excerpts first, then open detail pages only for new survivors. Same `javascript_tool` pattern as Step 1c, but reading from the Indeed detail page structure.

6. Indeed is the source most likely to CAPTCHA-block. The **CAPTCHA handling** section covers the prompt-and-retry-once flow — use it for every Indeed interstitial, not just the first.

---

## Step 3 — Dice (if `sources.dice: true`)

Dice's old RSS endpoint now returns the full search-page HTML (~360KB per query). Their internal API requires auth. The least-fragile path is Chrome MCP against `dice.com/jobs/q-<query>-l-<location>-jobs.html`.

### 3a. Build search queries

Same role × location matrix. URL: `https://www.dice.com/jobs?q=<URL-encoded role>&location=Portland%2C+OR&radius=25&radiusUnit=mi&pageSize=20&filters.postedDate=SEVEN`

For Remote, swap `&workplaceTypes=Remote` in and drop the location filter.

### 3b. For each query

1. Navigate. Wait ~2–3 seconds.
2. **Opportunistic login check (first query only).** On the first Dice page of the run, peek at the nav before extracting cards:

   ```javascript
   (function() {
     const top = (document.body.innerText || '').split('\n').slice(0, 40).map(s => s.trim()).join(' | ').toLowerCase();
     // Logged-out Dice surfaces "Sign in" or "Log in" in the top right.
     // Logged in, Dice shows "Dashboard" / "My profile" / a profile menu.
     const looks_logged_out = /\b(sign in|log in)\b/.test(top) && !/dashboard|my profile|my dice/.test(top);
     return { looks_logged_out };
   })()
   ```

   If it looks logged out, add a one-line note to the run summary (do NOT halt):

   > Dice: scanning anonymously. Optional — log into dice.com in your Chrome window before the next run if you want personalized results and saved searches to apply.

   Then continue normally.
3. Check for a CAPTCHA / anti-bot interstitial — same heuristic as Indeed (`/verify you are (human|a human)|cloudflare|press and hold|are you a robot/i` in page text). Dice is less aggressive than Indeed but it does happen. If present, follow the shared **CAPTCHA handling** flow below.
4. Extract cards with `javascript_tool`. Dice uses `data-cy="card-title-link"` (or recent variant) for the result link. If that selector misses, fall back to `a[href*="/job-detail/"]`.

   ```javascript
   (function() {
     window.scrollTo(0, document.body.scrollHeight);
     const links = Array.from(document.querySelectorAll('a[href*="/job-detail/"]'));
     const seen = new Set();
     const out = [];
     for (const a of links) {
       const href = a.href.split('?')[0];
       if (seen.has(href)) continue;
       seen.add(href);
       const root = a.closest('div[role="listitem"]') || a.closest('article') || a.closest('div');
       const text = (root ? root.innerText : '').split('\n').map(s => s.trim()).filter(Boolean);
       out.push({
         url: href, title: text[0] || '', company: text[1] || '', location: text[2] || '',
         text_excerpt: text.slice(0, 10).join(' | ')
       });
     }
     return out;
   })()
   ```

5. Cheap-filter, dedup, fetch JD per survivor (same pattern as Steps 1c–1d).

---

## Step 4 — Company career pages (if `sources.company_career_pages: true`)

This is the most-controllable source and the one the cost guardrails were written for. Skip this entire step if `sources.company_career_pages: false`.

### 4a. Pick which companies to scan

From `target_companies.csv`:

1. Filter to rows whose `priority` matches `priority_filter` (e.g., `High` means High-only; if the user sets `Medium`, accept High and Medium; if `Low`, accept all).
2. Drop rows whose entry in `jobs/career-page-cache.json` has `last_scanned` within `cache_results_days` of today.
3. Cap the remaining list at `max_pages_per_run`. If you have more eligible companies than the cap, prefer companies the user hasn't scanned in the longest, then break ties by priority then by row order.

### 4b. ATS detection per company (auto-pipeline)

For each chosen company, walk this auto-detection chain and stop at the first hit. The first two hits cost ~5K–50K tokens and don't need Chrome at all; the third is the expensive fallback.

**Step 4b-i — Try Lever public JSON API.** Slug = lowercase company name with hyphens, but Lever uses simple slugs without `inc`/`co`/`group` suffixes. Try a few variants.

```bash
curl -s -o /tmp/scan_lever.json -w "%{http_code}" \
  "https://api.lever.co/v0/postings/<slug>?mode=json"
```

- HTTP 200 with a non-empty JSON array → Lever board. Parse it (Step 4c).
- Anything else → next.

**Step 4b-ii — Try Greenhouse public JSON API.** Token = lowercase company name, sometimes shortened.

```bash
curl -s -o /tmp/scan_gh.json -w "%{http_code}" \
  "https://boards-api.greenhouse.io/v1/boards/<token>/jobs"
```

- HTTP 200 with `{ "jobs": [...] }` → Greenhouse board. Parse it (Step 4c).
- Anything else → next.

**Step 4b-iii — Chrome MCP fallback.** Navigate to the company's career page (Google `"<company> careers"` only if you don't have a known URL — but for Jay's target list, try `https://www.<company-domain>/careers/` and `https://careers.<company-domain>/` first). If you find an embedded ATS link (Lever, Greenhouse, Workday URL pattern, Workable, Ashby, SmartRecruiters, iCIMS, Jobvite, BambooHR), navigate to that and parse the list. If it's a custom Workday tenant (`<co>.wd<N>.myworkdayjobs.com/<site>`), try the search POST endpoint with body `{"appliedFacets":{},"limit":50,"offset":0,"searchText":""}` to `wday/cxs/<tenant>/<site>/jobs`. Required headers usually include a same-origin `Referer`. If even that fails, scrape the rendered page text with `get_page_text` and pull job links from `<a>` href patterns.

### 4c. Parse the ATS payload

For Lever (`mode=json`):
- Each entry has `text` (title), `categories.location`, `categories.team`, `categories.commitment` (type), `hostedUrl`, `descriptionPlain` (clean text body), `createdAt` (Unix ms).
- Use `descriptionPlain` over `description` (the latter is HTML).

For Greenhouse:
- The light list endpoint gives `title`, `location.name`, `absolute_url`, `id`, `first_published`, `updated_at`, and `metadata` (sometimes carries Workplace Type).
- Apply cheap filters to titles + locations FIRST. Only after a posting passes the cheap filters do you fetch its detail at `https://boards-api.greenhouse.io/v1/boards/<token>/jobs/<id>?questions=false` to get the `content` HTML.

For Chrome-fallback (custom sites):
- Extract title + location + URL per result. Same cheap-filter-first pattern. Open the detail page only for survivors.

### 4d. Normalize and write back the cache

Build a `RawPosting` record per survivor in the same shape as Step 1d, with `source: "career-page"`.

After scanning a company, write its entry to `jobs/career-page-cache.json`:

```json
{
  "<Company name>": {
    "last_scanned": "<today_iso>",
    "url": "<the URL or API endpoint actually used>",
    "ats": "<lever | greenhouse | workday | custom | chrome-fallback>",
    "raw_postings_seen": <int>
  }
}
```

This is what the guardrail in Step 4a-2 reads next run.

---

## Step 5 — Hard filters (applied to every `RawPosting`)

These are the filters from Section 6 of the project spec. A posting is **dropped from the inbox entirely** if it fails any of them. Log every drop with a one-line reason.

1. **Title category.** Title must read reasonably as Product Manager / Program Manager / Project Manager (or close variants — Senior PM, Lead PM, Principal PM, Group PM, Sr. Program Manager, TPM, etc.). Use judgment, not regex. "Product Marketing Manager", "Engineering Manager", "Account Manager", "Customer Success Manager" → drop unless the JD body clearly establishes a PM-track scope.

2. **Seniority.** Drop if the title indicates VP+, "Head of", Chief, Associate, Junior, or Intern. Director-level titles **stay in** (they get penalized at scoring time, not filtered out).

3. **Location.** Keep only postings that match one of:
   - Portland metro (within ~30 miles of Portland, OR)
   - US-based Remote (reject postings that say "Remote — Canada only" or "Remote — EU")
   - Hybrid with Portland HQ
   
   Anything else → drop. If the location field is genuinely unclear, read the JD body for a location clue before deciding.

4. **Compensation floor.** If `comp_disclosed` is true AND `comp_floor_met` is false → drop. Postings without disclosed comp ARE NOT dropped — keep them, with `comp_floor_met: null`. The floor: $80/hr OR $110K annual.

5. **Job type.** Drop if the type doesn't match one of: full-time, contract, contract-to-hire. (Drop part-time, internship, temp-staffing-only postings.) If the type is genuinely unknown, keep it and let scoring handle it.

Track every drop in a structured array: `{url, reason, source, title, company}`. The summary at the end prints aggregate counts.

---

## Step 6 — Dedup against `jobs/seen.json`

For every surviving `RawPosting`:

1. Look up `posting.url` in `jobs/seen.json`. If present, **skip** — note it in the dedup counter and move on.
2. Otherwise, the posting is new. Add it to `jobs/seen.json`:
   ```json
   "<url>": { "first_seen": "<today_iso>", "status": "inbox", "summary_score": null }
   ```

Write `jobs/seen.json` back to disk after each scan source completes (not after every single posting — that's too chatty — but at least once per source so a mid-run crash doesn't lose all dedup progress).

---

## Step 7 — Write each new survivor to `jobs/inbox/`

For each new survivor, build the stable id:

```
<source>-<YYYY-MM-DD>-<company-slug>-<role-slug>
```

Slugify rules: lowercase, ASCII only, hyphens for spaces, drop punctuation. Example: `linkedin-2026-06-09-nike-director-global-digital-lp-ops`. Truncate the role slug to ~60 chars if it's a long title. If a collision still happens (rare), append `-2`, `-3`, etc.

Write `jobs/inbox/<id>.md` using the template at `templates/job-posting.md`. The frontmatter fields all come from the `RawPosting` record. The body sections:

- **Job Description** — the full JD body text. Preserve paragraph breaks. Strip boilerplate footers (EEO statements, benefits legalese) only if they crowd the file beyond ~6K words.
- **Key Requirements (extracted)** — 3–6 bullets pulled from the JD's must-have list. If the JD doesn't explicitly enumerate must-haves, extract them by judgment from the responsibilities/qualifications sections.
- **Nice-to-haves (extracted)** — same, for the stated bonus / preferred qualifications. Optional — skip if the JD doesn't surface any.
- **Notes** — free-text observations from the scan. Examples: "Hiring manager named in posting: Jane Doe", "Posted 14 days ago — may be stale", "JD mentions PCI compliance". Optional.

---

## Step 8 — Run summary

Print a clean per-source summary at the end. Format:

```
Scanned 4 sources.
- LinkedIn: 31 postings → 18 after filters
- Indeed: 12 → 7
- Dice: 4 → 1
- Career pages (8 priority-High companies): 6 → 4

30 total survivors. 12 are dedupes from prior runs — skipped.
18 new postings written to jobs/inbox/.

Filter drops: 8 too senior (VP+), 4 wrong location, 2 too junior, 11 dedupes.

Ready when you are: /score-jobs
```

If any source was skipped or blocked, name it and the reason in the summary. Possible variants:

```
- Indeed: scanning anonymously — log in next time to reduce CAPTCHA risk.
- Indeed: CAPTCHA cleared by user mid-run; resumed and finished normally.
- Indeed: skipped by user after CAPTCHA.
- Indeed: CAPTCHA reappeared after user retry — skipped.
- LinkedIn: skipped (Chrome not connected).
- LinkedIn: skipped (not logged in).
- Career pages: 4/10 scanned; cache hit on 6 (last scanned in past 3 days).
```

---

## Hard filter reference (the cheat sheet)

| Filter | Drop if... |
|---|---|
| Title category | Title isn't reasonably a Product / Program / Project Manager role. |
| Seniority | Title indicates VP+, Head of, Chief, Associate, Junior, or Intern. (Director stays.) |
| Location | Not Portland metro AND not US-Remote AND not hybrid-with-Portland-HQ. |
| Compensation | `comp_disclosed` true AND below $80/hr or $110K annual. (Undisclosed → keep.) |
| Job type | Doesn't match full-time, contract, or contract-to-hire. |

Apply in order. Stop at the first failed filter. Record the failed-filter name in the per-posting drop log.

---

## Stable id generation (the cheat sheet)

```
<source>-<YYYY-MM-DD>-<company-slug>-<role-slug>
```

- `source`: `linkedin` | `indeed` | `dice` | `career-page`
- `YYYY-MM-DD`: the `fetched_date` (today)
- `company-slug`: lowercase ASCII, hyphens for spaces, drop punctuation (`Columbia Sportswear` → `columbia-sportswear`)
- `role-slug`: same rules, truncate to ~60 chars
- Collisions append `-2`, `-3`, ...

Example: `career-page-2026-06-09-kasada-senior-product-manager`

---

## Failure modes — what to do when things go wrong

- **Chrome not connected** → mark LinkedIn / Indeed / Dice / Chrome-fallback career-pages as skipped. Continue with ATS-API career-page scans (which don't need Chrome). Tell the user clearly in the summary.
- **Not logged into LinkedIn** → skip LinkedIn for this run. Don't try to log in for the user.
- **CAPTCHA / "verify you are human" on Indeed or Dice** → use the **CAPTCHA handling** flow below: pause the source, prompt the user to clear it manually in Chrome (and log in while they're there to reduce future hits), retry once, fall back to "skip this source" if it reappears.
- **LinkedIn rate limit ("You're searching too fast")** → halt LinkedIn. Suggest the user wait an hour.
- **ATS API returns 404 or empty array for a company** → fall through to the next detection step (Greenhouse → Chrome) silently. The company genuinely may have no open postings.
- **Custom career page returns a CAPTCHA or anti-bot block** → log it, skip the company for this run, write the cache entry anyway so we don't re-attempt for `cache_results_days`.
- **JD body extraction returns suspiciously short text** (< 300 chars) → flag it in the per-posting `Notes`, keep the posting (better to have a thin file you can revisit than to silently drop it).
- **A page's overall text changed shape** → fail loudly with the URL and the page sections you tried. Better an honest miss than a silent wrong record.

Loud failure beats quiet wrongness. `/score-jobs` will run cleanly on a partial inbox.

---

## CAPTCHA handling (Indeed and Dice)

This skill aligns with the project's "AI as partner, not autonomous worker" stance: when one of these sites pushes a CAPTCHA, the user — who is sitting right at the Chrome window — clears it manually, and the scan continues.

**Trigger.** Any time Step 2b item 3 (Indeed) or Step 3b item 3 (Dice) detects an interstitial. Also trigger if the cards extractor returns zero results on a query that should plausibly have results and the page text contains anti-bot phrases.

**The flow.**

1. **Pause the source** (not the whole run). Don't navigate away from the CAPTCHA page — the user needs to see the same page in their Chrome window.

2. **Prompt the user via `AskUserQuestion`.** Use this exact shape (Indeed example shown — substitute "Dice" / "dice.com" for the Dice variant):

   - Question: `Indeed is showing a CAPTCHA on the search results page. How would you like to handle it?`
   - Header: `CAPTCHA on Indeed`
   - Options:
     - `Clear it now` — *(Recommended)* — "I'll switch to my Chrome window, complete the CAPTCHA (and log in while I'm there to reduce future hits), then come back. Retry once."
     - `Skip Indeed for this run` — "Continue with the other sources. The summary will note that Indeed was skipped."
     - `Stop the run` — "Save what's been scanned so far and stop. I'll re-run later."

3. **On `Clear it now`:**
   - Wait for the user's reply. When they confirm they're done, call `mcp__Claude_in_Chrome__navigate` to re-load the same URL once. Wait ~3 seconds.
   - Re-check for the interstitial. If gone → resume from Step 2b item 4 (extract cards) and continue the source normally.
   - If the interstitial is still present → tell the user `That CAPTCHA didn't clear. Skipping Indeed for this run.`, add the skip note to the run summary, and continue with the next source. Do not prompt them again.

4. **On `Skip Indeed for this run`:**
   - Mark Indeed (or Dice) as skipped. Add `Indeed: skipped by user after CAPTCHA.` to the run summary. Continue with other sources.

5. **On `Stop the run`:**
   - Write whatever's pending to `jobs/inbox/`, `jobs/seen.json`, and `jobs/career-page-cache.json`.
   - Print the partial run summary including `Run stopped by user mid-scan after Indeed CAPTCHA.`
   - Exit cleanly.

**One prompt per source per run.** If you've already prompted the user about Indeed once this run and they chose `Clear it now`, do not prompt again — if Indeed CAPTCHAs again, treat that as `Skip Indeed for this run` automatically and note it in the summary.

**The journey this enables.**

- Run 1: User declined the soft "log in to Indeed" suggestion. Scan worked anonymously.
- Run 2 or 3: Indeed CAPTCHAs. Skill pauses, prompts. User flips to Chrome, clears it (and logs in while they're there), returns, says ready. Skill retries once, succeeds, continues.
- Run 4+: Indeed cookies remember the login for weeks. CAPTCHA risk drops materially.

This is the difference between a tool that quietly drops a source and a tool that asks for 30 seconds of help and recovers.

---

## Calibration findings (run 2026-06-09)

Token-cost numbers measured during the M2 build, used to set the `max_pages_per_run` default of 10. Re-measure if LinkedIn / ATS surfaces change.

| Source | Method | Per-company cost | Notes |
|---|---|---|---|
| Kasada careers | Lever public JSON API (`api.lever.co/v0/postings/kasada?mode=json`) | ~24K input tokens (6 jobs, ~95KB payload, ~16KB/job inc. HTML+plain desc) | Lever returns the entire board in one call. Use `descriptionPlain`, not `description`, to halve the per-job size. |
| Airbnb careers | Greenhouse public JSON API — light list at `boards-api.greenhouse.io/v1/boards/airbnb/jobs` then per-job detail | ~43K tokens for the 223-job list (no content), then ~2K tokens per survivor's detail | The trap: fetching `?content=true` returns a single 2.2MB payload — ~550K tokens. **Don't do that.** Use the light list + targeted detail fetches. |
| Custom enterprise (Nike, Adidas, etc.) | Chrome MCP — career page navigation + paged scraping | ~30K–80K tokens per company, depending on JS rendering and depth | Highly variable. Workday tenants sometimes expose a `wday/cxs/.../jobs` POST endpoint that returns clean JSON; try it before scraping the rendered page. |

**Per-run budget at current defaults** (`max_pages_per_run: 10`):

- If ~7 of 10 target companies are Lever/Greenhouse (avg ~20K input tokens each → ~140K) and ~3 are custom enterprise (avg ~50K each → ~150K): **~290K input tokens** for the career-page slice alone.
- LinkedIn + Indeed + Dice on top: ~100K–200K input tokens depending on result volume and how many survivors need JD-detail fetches.
- **Total per `/scan-jobs` run: roughly 400K–600K input tokens.** That matches Section 8 of the project spec ("200K–500K tokens for one end-to-end run", which covered scan + score combined; scan alone runs warmer).

**Recommendation for `max_pages_per_run` defaults:**

- **10 is the right ceiling for Pro users** if their target list skews public-ATS (Lever/Greenhouse hits show up cheaply).
- **Drop to 6–8 if the target list is mostly custom enterprise** (Nike, Adidas, Apple, Google, etc., all use bespoke career sites). Each custom site burns 50K–80K tokens.
- **Max users can safely raise to 15–20** if the target list is curated and they want fuller coverage on weekly runs.

The skill enforces whatever number is in `settings.md`. Edit `settings.md` to change it.

---

## LinkedIn / Indeed / Dice / ATS UI dependencies (for future maintenance)

These are the page-structure assumptions this skill carries. When a site changes its UI, update this list.

### LinkedIn

1. **Jobs search URL** — `https://www.linkedin.com/jobs/search/?keywords=<q>&location=<loc>&distance=<mi>&f_TPR=r<sec>&f_WT=<mode>`. `f_WT` codes: `1` = Onsite, `2` = Remote, `3` = Hybrid (combinable, e.g., `f_WT=2%2C3`). `f_TPR=r604800` = past week.
2. **Result-card link** — `<a href="/jobs/view/<id>...">` anchored inside an `<li>`. The first 3 non-empty `innerText` lines of the card are typically `[title, company, location]`.
3. **Detail page** — `<h1>` is the job title. The block under "About the job" is the JD. The salary/comp string, when disclosed, lives in a "Pay range provided by employer" block somewhere in the upper third of the page.
4. **Posted-date phrasing** — "Posted N days/weeks ago" or "Reposted N days ago". Use this for `posted_date`.

### Indeed

1. **Search URL** — `https://www.indeed.com/jobs?q=<q>&l=<loc>&radius=<mi>&fromage=<days>`.
2. **Result-card link** — `<a id="job_<jk>" ...>` OR `<a data-jk="<jk>" ...>`. The `jk` is the canonical posting id; detail URL is `/viewjob?jk=<jk>`.
3. **CAPTCHA interstitial** — Indeed renders a Cloudflare-style "Verify you are human" page. If present, halt the source for this run.

### Dice

1. **Search URL** — `https://www.dice.com/jobs?q=<q>&location=<loc>&radius=<mi>&radiusUnit=mi&pageSize=20&filters.postedDate=SEVEN`.
2. **Result-card link** — `<a href="/job-detail/...">`. Title and company are in adjacent text nodes inside the same card container.
3. **Old RSS endpoint** (`dice.com/jobs/rss/...`) now returns the full search-page HTML — do not use it.

### Lever public API

1. `https://api.lever.co/v0/postings/<company-slug>?mode=json` returns the full active board as a JSON array. Public, CORS-open, no auth.
2. Per-posting fields: `text` (title), `categories.location`, `categories.team`, `categories.commitment`, `hostedUrl`, `description` (HTML), `descriptionPlain` (text), `createdAt` (Unix ms).
3. The slug is the user-facing URL slug on `jobs.lever.co/<slug>/` — for companies on the target list, the slug usually matches a lowercased version of the company name.

### Greenhouse public API

1. `https://boards-api.greenhouse.io/v1/boards/<token>/jobs` returns a light list — `title`, `location.name`, `absolute_url`, `id`, timestamps, `metadata`.
2. `https://boards-api.greenhouse.io/v1/boards/<token>/jobs/<id>?questions=false` returns the per-job `content` HTML. ~5–10K chars per job.
3. **Do not** pass `?content=true` on the list endpoint — for boards with 200+ jobs it returns a 1–3MB payload.
4. The token is the user-facing path on the company's Greenhouse-hosted board (`boards.greenhouse.io/<token>` or `job-boards.greenhouse.io/<token>`).

### Workday (`<co>.wd<N>.myworkdayjobs.com/<site>`)

1. There is a JSON search endpoint at `wday/cxs/<tenant>/<site>/jobs` that accepts a POST with body `{"appliedFacets":{},"limit":50,"offset":0,"searchText":""}`. Required headers usually include `Origin` and `Referer` matching the tenant. Responses return a structured `jobPostings` array with `title`, `locationsText`, `externalPath`, `postedOn`.
2. Each posting's detail JSON is at `wday/cxs/<tenant>/<site>/job/<externalPath>` — same headers.
3. Workday tenants sometimes 422 if the request body or headers are off. If the JSON endpoint fails, fall back to Chrome MCP scraping the rendered search page.

---

## Maintenance pointers

Where things live in this skill:

- LinkedIn search URL construction and parsing: **Step 1**.
- Indeed search URL construction and parsing: **Step 2**.
- Dice search URL construction and parsing: **Step 3**.
- Career-page ATS auto-detection chain: **Step 4b** (Lever → Greenhouse → Chrome fallback).
- Hard-filter logic: **Step 5**.
- Dedup index format: **Step 6**.
- Inbox file format and template: **Step 7** + `templates/job-posting.md`.
- Cache file format: **Step 4d**.
- Run summary format: **Step 8**.
- CAPTCHA prompt-and-retry flow: **CAPTCHA handling** (referenced from Steps 2b and 3b).
- Calibration numbers behind the defaults: **Calibration findings**.

Updating one source should not require touching the others. Keep them independent.
