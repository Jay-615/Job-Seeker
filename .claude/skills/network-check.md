---
name: network-check
description: Tier 1 network-proximity analysis for a single company. Uses Claude in Chrome on the user's LinkedIn People search to count 1st- and 2nd-degree connections, identifies which 1st-degrees appear as mutuals, cross-references against config/connections.csv ratings to compute bonus points, returns base/bonus/final scores plus warmest intro paths. Caches results in jobs/network-cache.json per the settings.md TTL.
---

# /network-check — Tier 1 network proximity for one company

You take a company name as a parameter and return a structured network-proximity analysis the user (or `/score-jobs`) can act on. The output structure is fixed — `/score-jobs` consumes it programmatically, so do not change the field names or shape.

This is the most-likely-to-break skill in the project because it depends on LinkedIn's UI. Fail loudly when something looks wrong rather than guessing. The **LinkedIn UI dependencies** section at the bottom of this file lists exactly what you depend on so a future maintainer can update it in one place.

---

## Data source — prefer `config/company_connections.csv`

`/map-company-connections` is now the canonical way to **find and document** Jay's connections at a
company. This skill's job is **scoring**, not re-discovery — so lean on that table when it exists.

**Before any 2nd-degree scraping (Step 4), check `config/company_connections.csv` for rows where
`company` == this company.** If a block exists, derive the network signal from it instead of the live
page-1 scrape. It is strictly more complete: `/map-company-connections` paginates all pages and
**drills the "…and N other mutual connections" truncation** that this skill's page-1 scrape silently
misses — a High-rated bridge hidden in a "+21 others" tail would otherwise be lost, undercounting the
bonus. From the block, compute:

- `second_degree_count` = number of rows for the company (still a floor if the map run hit LinkedIn's
  ~100 cap — say so in `reasoning`).
- `first_degree_mutuals_top` = aggregate the `bridge_connections` cells across the rows into a
  `{name -> count}` frequency map; take each name's `rating` from `config/connections.csv`.
- `bonus_score` and `warmest_paths` per Steps 5–6 using those rated bridges.
- `data_source: "company_connections.csv (mapped <scanned_date>)"`.

Still get `first_degree_count` from the company People page (Step 3a) — `company_connections.csv` only
holds 2nd-degrees (`network=["S"]`). Keep the Step 7 output shape identical either way; `/score-jobs`
reads it programmatically.

If the company has **no block** in `company_connections.csv`, fall back to the live scrape in Steps
3–4 as written, and note in `reasoning` that running `/map-company-connections <company>` would give a
fuller, drill-in-complete picture.

---

## Parameters

- **company name** (required) — e.g., `/network-check Nike`. Accept multi-word names (`/network-check "Columbia Sportswear"`).

If no company name was given, ask for it via `AskUserQuestion` and stop if the user doesn't provide one.

---

## Step 1 — Read settings and check the cache

1. Read `config/settings.md`. Pull `network_cache_days` from the **Network Proximity** section (default to 30 if missing or unparseable).
2. Read `jobs/network-cache.json` if it exists. If not, treat it as an empty `{}`.
3. Look for an entry matching the company name (case-insensitive). If you find one and its `scanned_date` is within `network_cache_days` of today, **return the cached structured result** and stop. Set `data_source: "cache"` and include the cached `scanned_date` so the caller knows how fresh it is.

A cache hit means you do not touch Chrome at all — that's the whole point of caching, and it's what keeps token cost down on weekly runs.

---

## Step 2 — Resolve `linkedin_company_id`

LinkedIn's People search uses the numeric company ID, not the company name. You need it before any browser work.

1. Read `config/target_companies.csv`. Find the row whose `company` column matches the parameter (case-insensitive, trim whitespace). If the row exists and `linkedin_company_id` is non-empty, use it. Done.
2. If the company is on the CSV but `linkedin_company_id` is empty, look it up live (Step 2b) and **write the value back** to the same row in `config/target_companies.csv` so future runs are fast and unambiguous.
3. If the company is not on `target_companies.csv` at all, look it up live and proceed without writing back (the CSV is the user's target list — don't auto-add companies they haven't chosen). Note this in the `reasoning` field of the output.

### Step 2b — Live lookup of `linkedin_company_id`

The robust path (verified in the field) is to navigate to the company's People tab and read the most-frequent `fsd_company:<id>` URN out of the page HTML. That URN is repeated many times on the company's own page, so it's the highest-frequency numeric ID — much more reliable than clicking through navigation that may change.

1. Ensure Chrome is connected — see the **Browser preflight** block below. Bail with a clear message if not.
2. Open or reuse a tab in the MCP tab group (`tabs_context_mcp` with `createIfEmpty: true`).
3. Navigate to `https://www.linkedin.com/company/<slug>/people/`, where `<slug>` is the lowercase hyphenated company name (e.g., "Columbia Sportswear" → `columbia-sportswear`). Wait ~2 seconds for the page to render.
4. Use `mcp__Claude_in_Chrome__javascript_tool` to extract candidate IDs:

   ```javascript
   (function() {
     const html = document.documentElement.outerHTML;
     const re = /fsd_company:(\d{4,12})/g;
     const counts = {};
     let m;
     while ((m = re.exec(html)) !== null) counts[m[1]] = (counts[m[1]] || 0) + 1;
     return Object.entries(counts).sort((a, b) => b[1] - a[1]).slice(0, 5);
   })()
   ```

   The top-ranked ID is the company's numeric ID. (Field-tested results: Nike = 2029, Adidas = 3748, Stripe = 2135371.)
5. If the page redirects to a 404 or you find zero `fsd_company` URNs, the slug guess was wrong. Fall back to `https://www.linkedin.com/search/results/companies/?keywords=<URL-encoded company name>`, read the page text, and report the first 1-2 candidate matches (name + URL) to the user via `AskUserQuestion` rather than guessing.
6. If you can't find an unambiguous match (multiple companies with the same name, no results, etc.), stop and report honestly to the user. Don't guess.

---

## Browser preflight

Before any LinkedIn navigation:

1. Call `mcp__Claude_in_Chrome__list_connected_browsers`. If empty, bail with:

   > I can't reach Chrome. Make sure the Claude in Chrome extension is installed and connected, then re-run. If you've never set this up, run `/setup` first.

2. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true` to get a working tab.
3. Navigate to `https://www.linkedin.com/feed/`. If the page redirects to a login screen, bail with:

   > Chrome is connected but you don't appear to be logged into LinkedIn. Log in in Chrome (open https://www.linkedin.com and sign in), then re-run.

   Don't try to log in for the user.

---

## Step 3 — Fetch the 1st-degree count

**Important reality check from in-the-field testing (2026-06):** LinkedIn's filtered People-search page (the `network=F`/`network=S` URL) does *not* render a total-count header. There is no "About N results" line on these pages. So we get the 1st-degree count from a different source — the company's People tab — and use the search page only for surfacing mutual-connection excerpts.

### 3a. Get the 1st-degree count from the company People page

1. Navigate to `https://www.linkedin.com/company/<slug>/people/`. The `<slug>` is the URL-friendly form of the company name (lowercase, hyphens, no spaces — e.g., "Nike" → `nike`, "Columbia Sportswear" → `columbia-sportswear`).

2. Use `mcp__Claude_in_Chrome__javascript_tool` to read the page text and pull the connections hint line. It appears in the form:

   ```
   <Name1> & <N> other connections work here
   ```

   So total 1st-degree at this company = `N + 1`. Reference snippet:

   ```javascript
   (function() {
     const lines = (document.body.innerText || '').split('\n').map(l => l.trim());
     // e.g. "Chris & 43 other connections work here" -> 44
     const m = lines.map(l => l.match(/^(.+?)\s+&\s+(\d+)\s+other connections work here$/)).find(Boolean);
     if (m) return { first_degree_count: parseInt(m[2], 10) + 1, named_anchor: m[1] };
     // If the line is missing entirely, the company People page may show no in-network line at all,
     // which usually means 0 1st-degree connections at that company.
     return { first_degree_count: 0, named_anchor: null };
   })()
   ```

3. If the line is absent, the user has 0 1st-degree connections at this company. Don't guess otherwise.

### 3b. Surface the 1st-degree result cards (optional, for the named-anchor sanity check)

1. Construct:

   ```
   https://www.linkedin.com/search/results/people/?currentCompany=%5B%22<linkedin_company_id>%22%5D&network=%5B%22F%22%5D
   ```

2. Navigate, wait ~2-3 seconds for React to render.

3. You can use this page to confirm the count seems right, but **the per-card data on the F page isn't needed for the Tier-1 score** — the score and warmest paths come from the 2nd-degree page (Step 4). Skip parsing this page unless you want a sanity check.

---

## Step 4 — Fetch 2nd-degree and their mutuals

1. Construct:

   ```
   https://www.linkedin.com/search/results/people/?currentCompany=%5B%22<linkedin_company_id>%22%5D&network=%5B%22S%22%5D
   ```

2. Navigate, wait ~2-3 seconds for React render.

3. Parse with `javascript_tool`. Reference snippet:

   ```javascript
   (function() {
     // Scroll so lazy-rendered pagination appears at the bottom.
     window.scrollTo(0, document.body.scrollHeight);
     const main = document.querySelector('main');
     const text = (main ? main.innerText : '').split('\n').map(l => l.trim()).filter(Boolean);

     // Cards: each card has a line like "• 1st" / "• 2nd" / "• 3rd" right after the name.
     const cards = [];
     for (let i = 0; i < text.length; i++) {
       if (/^•\s*(1st|2nd|3rd)/i.test(text[i])) {
         const name = text[i - 1];
         const headline = text[i + 1] || '';
         let mutual = null;
         for (let j = i + 1; j < Math.min(i + 8, text.length); j++) {
           if (/mutual connection/i.test(text[j])) { mutual = text[j]; break; }
         }
         cards.push({ name, headline, mutual });
       }
     }

     // Pagination: capped at 10 visible page numbers + "Next". Use as a lower-bound estimate.
     const pageNums = text.filter(l => /^\d{1,3}$/.test(l)).map(n => parseInt(n, 10));
     const max_page_visible = pageNums.length ? Math.max(...pageNums) : 1;
     return { cards, max_page_visible };
   })()
   ```

4. **2nd-degree count estimation.** LinkedIn doesn't expose the total. Use `max_page_visible × 10` as a *lower-bound* estimate. If `max_page_visible === 10` and a `Next` button is present, the true count is likely much higher — set `second_degree_count` to `100` and note in `reasoning` that it's a floor.

5. **Parse mutual-connection excerpts.** Four shapes confirmed in the field:

   - `"<Name> is a mutual connection"` → 1 mutual
   - `"<Name1> and <Name2> are mutual connections"` → 2 mutuals
   - `"<Name1>, <Name2>, and <Name3> are mutual connections"` → 3 mutuals
   - `"<Name1>, <Name2> and N other mutual connections"` → 2 named, N unnamed

   Parse out the **explicitly named mutuals** (the first 1-3 names). Do not try to expand the `"and N other mutual connections"` tail — that requires clicking into each result, which is Tier 2 work (M9).

6. Aggregate across all visible 2nd-degree cards on the first page only (do not paginate — pagination would multiply token cost across every job being scored): build a frequency map `{ name -> appears_as_mutual_count }` for each 1st-degree the user has who LinkedIn surfaced as a mutual on at least one card.

This map is the raw signal for Step 5's bonus and Step 6's warmest-paths ranking.

---

## Step 5 — Compute the score

### Base score (from raw counts, per spec Section 6)

| Signal | Base score |
|---|---|
| `first_degree_count >= 1` | 70 |
| `second_degree_count >= 5` | 50 |
| `1 <= second_degree_count <= 4` | 30 |
| 0 known connections, but company on `target_companies.csv` | 20 |
| 0 known connections, not on `target_companies.csv` | 10 |

Use the highest applicable row. So a company with 1+ 1st-degrees scores 70 regardless of 2nd-degree count.

### Bonus score (requires `config/connections.csv`)

1. Check whether `config/connections.csv` exists.
2. If **not present**: skip bonuses, set `bonus_score: 0`, and add to `reasoning`: `"connections.csv not found; ran /build-connections has not been completed yet, so this score is base-only. Run /build-connections to enable High/Medium rating bonuses."` Also set `data_source: "fresh (connections.csv missing — base score only)"`.
3. If **present**: load it. Cross-reference each name in your Step 4 frequency map against the CSV's `name` column (case-insensitive, trim). For each match, read the `rating`:
   - **High** → +25 per distinct person
   - **Medium** → +10 per distinct person
   - **Low** or **Unrated** → +0

   Sum the bonuses across all distinct matched High/Medium people. `bonus_score` is the sum. (Note: it's per distinct person, not per appearance. A High-rated mutual appearing on 12 result cards still adds +25, not +300.)

### Final score

`final_score = min(100, base_score + bonus_score)`

---

## Step 6 — Identify warmest intro paths

From your Step 4 frequency map, restrict to people whose `rating` in `connections.csv` is `High` or `Medium`. Rank by `appears_as_mutual_count` descending. Take the top 3-5. These are the people the user can ask for intros to people inside the company.

If `connections.csv` is missing, set `warmest_paths: []` and note in `reasoning` that warmest-path ranking requires connection ratings.

---

## Step 7 — Build the structured output

This is the exact shape `/score-jobs` (M3) reads. Field names and types matter.

```json
{
  "company_name": "<as provided, trimmed>",
  "linkedin_company_id": "<numeric string>",
  "first_degree_count": <int>,
  "second_degree_count": <int>,
  "first_degree_mutuals_top": [
    {"name": "<full name>", "appears_as_mutual_count": <int>, "rating": "<High|Medium|Low|Unrated|Unknown>"}
  ],
  "warmest_paths": ["<full name>", "<full name>", ...],
  "base_score": <int 0-100>,
  "bonus_score": <int>,
  "final_score": <int 0-100>,
  "reasoning": "<one-paragraph narrative explaining the result>",
  "data_source": "cache | fresh | fresh (connections.csv missing — base score only)",
  "scanned_date": "<YYYY-MM-DD>"
}
```

Notes:

- `first_degree_mutuals_top` should hold up to 10 entries, sorted by `appears_as_mutual_count` descending. `rating` is `"Unknown"` if `connections.csv` exists but the name didn't match any row.
- `warmest_paths` is the *list of names* only — a quick at-a-glance projection of who to talk to first.
- `reasoning` is for humans. Mention: counts, top mutuals (with ratings), what's driving the score, any caveats (counts elided, connections.csv missing, etc.).

---

## Step 8 — Cache the result

Write the result into `jobs/network-cache.json` keyed by `company_name`:

```json
{
  "Nike": {
    "scanned_date": "2026-06-08",
    "linkedin_company_id": "7789",
    "first_degree_count": 39,
    "second_degree_count": 412,
    "first_degree_mutuals_top": [...],
    "base_score": 70,
    "bonus_score": 60,
    "final_score": 100,
    "warmest_paths": ["Sarah Smith", "John Doe", "Pat Lee"]
  }
}
```

If `jobs/network-cache.json` doesn't exist, create it. If it exists, read it, merge in or replace the entry for this company, and write it back. Preserve other companies' entries.

Don't cache a `data_source: "cache"` result back onto itself — that's a no-op.

---

## Step 9 — Report to the user

Print a short, human-readable summary:

```
Nike — network proximity
  1st-degree at company: 39
  2nd-degree at company: 412
  Top mutuals: Sarah Smith (High, ×12), John Doe (High, ×8), Pat Lee (Medium, ×7)
  Base 70 + Bonus 60 → Final 100/100
  Warmest paths: Sarah Smith, John Doe, Pat Lee
  Cached for 30 days.
```

When called by `/score-jobs`, return the structured JSON above and skip the human-readable summary.

---

## Failure modes — what to do when LinkedIn pushes back

- **Chrome not connected** → bail at preflight. Don't try to recover. Point to `/setup`.
- **Not logged in** → bail at preflight. Don't try to log in.
- **Company not found / ambiguous** → stop and tell the user. Don't guess an ID.
- **In-network hint line missing on `/company/<slug>/people/`** → treat `first_degree_count` as 0. (LinkedIn shows the "X & N other connections work here" line only when the count is positive.)
- **Pagination strip missing on 2nd-degree page** → set `second_degree_count` to the visible card count and flag it as a floor in `reasoning`.
- **CAPTCHA or interstitial** ("Let's do a quick security check") → bail. Tell the user what you saw and suggest they complete it manually in Chrome before re-running. Never try to solve CAPTCHAs.
- **Rate-limit page** ("You're searching too fast") → bail and recommend waiting an hour. Reduce future activity.
- **Total page-text changed shape** (LinkedIn UI rev) → bail with an honest "I couldn't parse the results page" and list the page sections you tried. Better a loud failure than a quiet wrong score.

Loud-and-clear failure is always preferable to a confident-looking wrong score. `/score-jobs` will note the missing network sub-score and use experience + target only — the user can re-run later.

---

## LinkedIn UI dependencies (for future maintenance)

This skill assumes the following about LinkedIn's UI as of June 2026. When LinkedIn changes things, update this list:

1. **Company-People page URL** — `https://www.linkedin.com/company/<slug>/people/`. The slug is the URL-friendly form of the name. We use this page for two things: pulling the 1st-degree count from the in-network hint line, and discovering the numeric company ID from the page's `fsd_company:<id>` URN frequency.

2. **In-network hint line** — on `/company/<slug>/people/`, when the user has 1+ 1st-degree connections at the company, the page text contains exactly one line of the form:

   ```
   <Name> & <N> other connections work here
   ```

   We compute `first_degree_count = N + 1`. If the line is absent, the user has 0 1st-degree connections at the company.

3. **Numeric company ID** — every LinkedIn company has a numeric ID embedded in `urn:li:fsd_company:<id>` strings across the company page's HTML. The most-frequent such ID on `/company/<slug>/people/` is the canonical company ID. Field-verified examples: Nike = 2029, Adidas = 3748, Stripe = 2135371.

4. **People search URL parameters** — `currentCompany=%5B%22<id>%22%5D&network=%5B%22F%22%5D`. The IDs are JSON-encoded inside the URL-encoded brackets. `F` = 1st-degree, `S` = 2nd-degree. The filter chips visible at the top of the page in the rendered UI read "People · 1st · <Company>" (or "2nd").

5. **No total-count header on filtered People search.** As of June 2026, LinkedIn does *not* render an "About N results" header on the network-degree-filtered People search page. The page jumps straight from the filter pills into the result cards. This is why we get the 1st-degree count from the company-People page (item 2 above) and estimate 2nd-degree count from pagination.

6. **Pagination** — the filtered People search page paginates ~10 results per page, with a numbered strip at the bottom that maxes out at 10 visible page numbers + a "Next" button. The presence of `Next` after page 10 indicates the true count is well above 100. Use `max_page_visible × 10` as a lower-bound estimate.

7. **Result cards have no stable class names.** LinkedIn obfuscates CSS class names server-side; selectors like `entity-result__item` no longer work. The robust extraction path is to read `document.querySelector('main').innerText`, split on `\n`, trim, and identify cards by the presence of a line matching `/^•\s*(1st|2nd|3rd)/i` — the bullet-prefixed degree marker that appears immediately after each card's name line.

8. **Mutual-connection excerpt shapes** — four forms confirmed in the field:
   - `"<Name> is a mutual connection"`
   - `"<Name1> and <Name2> are mutual connections"`
   - `"<Name1>, <Name2>, and <Name3> are mutual connections"`
   - `"<Name1>, <Name2> and N other mutual connections"`

   We extract only the explicitly named mutuals (the first 1-3 names). The `"and N other mutual connections"` tail is intentionally not expanded at this tier.

If a maintainer needs to update any of this, the search URL construction is in Step 3 and Step 4, the company-ID lookup is in Step 2b, the parsing logic is described in the embedded JS snippets in Steps 3a and 4, and all browser interaction is confined to Steps 2b, 3, 4, and the Browser preflight block. Nothing browser-related lives elsewhere in this file.
