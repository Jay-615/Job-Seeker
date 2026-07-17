---
name: build-connections
description: Populate config/connections.csv from the user's 1st-degree LinkedIn connections. Uses Claude in Chrome to paginate through linkedin.com/mynetwork/invite-connect/connections/ at human pace, writes each connection to the CSV with rating=Unrated. Re-runnable — preserves existing ratings, only adds new connections. Optionally seeds rating=High from a meetings log, if config/settings.md sets meetings_seed_path.
---

# /build-connections — populate connections.csv from LinkedIn

You are doing a bulk pull of the user's 1st-degree LinkedIn connections into `config/connections.csv`. This is a one-shot operation that's expected to be expensive in tokens (a network of 500 connections may cost 50-100K tokens). The user invokes it once after onboarding and then again periodically (e.g., monthly) to pick up new connections. Existing ratings must be preserved across re-runs.

The CSV schema is fixed (Section 5 of the spec):

```
linkedin_url,name,headline,rating,last_interaction,notes
```

`linkedin_url` is the unique key — that's how you match an existing row to a freshly scraped one.

---

## Opening message to the user

Before doing anything, set expectations. Print:

> I'm about to pull all of your 1st-degree LinkedIn connections into `config/connections.csv`. This uses Claude in Chrome to scroll through your connections list at human pace — for a network with ~500 connections, expect 3-5 minutes and 50-100K tokens. Existing ratings in the CSV (if any) will be preserved.
>
> If you'd rather skip the interactive walk-through later, you can edit `config/connections.csv` directly in Excel, Numbers, or any spreadsheet tool — sort by headline or company and mass-update the `rating` column. Both paths work.

Then ask via `AskUserQuestion`: **Proceed** / **Cancel**. If Cancel, stop cleanly.

---

## Step 1 — Browser preflight

1. Call `mcp__Claude_in_Chrome__list_connected_browsers`. If empty:

   > I can't reach Chrome. Make sure the Claude in Chrome extension is installed and connected, then re-run. If you've never set this up, run `/setup` first.

   Bail.

2. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true` to get a working tab.
3. Navigate to `https://www.linkedin.com/feed/`. If the page redirects to a login screen, bail with:

   > Chrome is connected but you don't appear to be logged into LinkedIn. Log in in Chrome (open https://www.linkedin.com and sign in), then re-run.

   Don't try to log in for the user.

---

## Step 2 — Load existing CSV and the optional meetings seed

1. Read `config/connections.csv` if it exists. Parse it into a dict keyed by `linkedin_url`. If the file doesn't exist yet, start with an empty dict — you'll create the file from scratch in Step 6.

2. **Optional meetings seed.** If the user keeps a log of people they've actually spoken to, those are `High` connections by definition and shouldn't need re-rating by hand.

   Read the optional `meetings_seed_path` setting from `config/settings.md`. If it's absent, or the file it points at doesn't exist, skip this silently — most users won't have one. If it resolves, read the `contact_name` column and build a set of names to seed as `High`.

   The path lives in `settings.md` rather than here because it's personal and machine-specific — `settings.md` is gitignored, this skill file is not. Never hardcode a user's home directory into a committed file.

   The matching rule is **full name, case-insensitive, trim whitespace**. A full name in the seed file matches the same full name (any case) on LinkedIn, but not an abbreviated or expanded variant — `"Sam Chen"` won't match `"Sam C."` or `"Samuel Chen"`. That's deliberate: it means partial or placeholder entries in the log (`"Andreas"`, `"COO (name TBD)"`) can't accidentally match a real person and mis-rate them. Under-matching is the safe failure here; over-matching silently marks a stranger as someone you know well.

3. Hold the existing-rows dict and the seed-names set in memory; you'll apply them when merging in Step 5.

---

## Step 3 — Navigate to the connections page and discover total count

1. Navigate to `https://www.linkedin.com/mynetwork/invite-connect/connections/`. Wait ~3 seconds for the React render.

2. Use `mcp__Claude_in_Chrome__javascript_tool` to read the total connection count from the page header. The line reads `"<N> connections"` (lowercase 'c', often comma-separated for thousands). Reference snippet:

   ```javascript
   (function() {
     const text = (document.body.innerText || '').split('\n').map(l => l.trim());
     for (const line of text) {
       const m = line.match(/^([\d,]+)\s+[Cc]onnections?$/);
       if (m) return { total: parseInt(m[1].replace(/,/g, ''), 10) };
     }
     return { total: null };
   })()
   ```

3. Report the total to the user. If the total is unusually large (≥1000), ask whether to proceed with full pagination or cap at, say, the first 500. Default suggestion: proceed in full but confirm every 250 connections (Step 4).

   If the total can't be parsed, that's fine — just note it. You'll still paginate; you just won't have a progress denominator.

---

## Step 4 — Paginate through all connections

LinkedIn's connections page uses **infinite scroll triggered by real wheel events**, with no "Show more" button. Two field-tested facts that matter:

1. **The scroll container is `<main>`, not `<body>`.** `document.body.scrollHeight` is the viewport height; `document.querySelector('main').scrollHeight` is the connections-list height.
2. **Programmatic `scrollTop` assignment does *not* trigger the load.** Neither does `Element.scrollIntoView()`. You must emit a real wheel event. The Chrome MCP's `computer` tool with `action: "scroll"` does this; `javascript_tool` setting `scrollTop` does not.

Loop:

1. Use `mcp__Claude_in_Chrome__computer` with `action: "scroll"`, `coordinate: [500, 400]` (anywhere over the main column), `scroll_direction: "down"`, `scroll_amount: 10`. This emits real wheel events that LinkedIn's intersection observer picks up.

2. Wait ~2 seconds for the next chunk to render. Each scroll batch typically reveals 8–10 new cards.

3. Check the card count using the `aria-label^="More actions for"` selector (more reliable than counting `/in/` anchors — each card has two anchors, but exactly one "More actions for <Name>" button):

   ```javascript
   document.querySelectorAll('button[aria-label^="More actions for"]').length
   ```

4. Stop the loop when:
   - The count reaches (or nearly reaches — off-by-one or two is normal for ghosts/recent removals) the discovered total from Step 3, **or**
   - The count stops increasing across two consecutive scroll batches, **or**
   - You hit a user-defined cap.

5. **Throughput planning.** Batches of 5 scrolls + waits load ~40-50 cards. For a 270-connection network: ~5-6 batches, ~3 minutes total. Use `browser_batch` to chain scroll/wait/scroll/wait into one round-trip — much faster than separate calls and still paced at human speed (the waits inside the batch are real waits).

6. **Confirmation prompt every 250 cards** for users with large networks: after 250, 500, 750, ... cards are loaded, pause and ask via `AskUserQuestion`: **Continue** / **Stop here and save what I've got**. This lets the user bail without losing progress if the run is taking longer than expected.

7. **Failure modes during pagination:**
   - If you see a CAPTCHA, security check, or "You've reached a temporary limit" interstitial → stop immediately, save what you've collected so far, and tell the user honestly what happened. Suggest they wait an hour and re-run.
   - If the page text shape changes (no card markers found at all) → stop, save partial results, and tell the user the LinkedIn UI may have shifted. Reference this skill file's "LinkedIn UI dependencies" section at the bottom so they can update the selectors.

---

## Step 5 — Extract per-connection fields

Once the loop in Step 4 finishes (or you bail), extract every connection currently rendered in the DOM. Field-tested snippet:

```javascript
(function() {
  // Each card contains an <a href="/in/<slug>/"> link to the profile. Class names
  // are obfuscated server-side, so we anchor on the URL pattern.
  // The card root is the first ancestor whose innerText length looks like a single
  // card (50–500 chars) — bigger and we've walked up into the grid container.
  const anchors = Array.from(document.querySelectorAll('a[href*="/in/"]'));
  const seen = new Map();

  for (const a of anchors) {
    let href = a.href.split('?')[0].split('#')[0];
    if (!/\/in\/[^/]+\/?$/.test(href)) continue;
    if (!href.endsWith('/')) href += '/';
    if (seen.has(href)) continue;

    let card = a.parentElement;
    for (let i = 0; i < 6 && card; i++) {
      const tlen = (card.innerText || '').length;
      if (tlen > 50 && tlen < 500) break;
      card = card.parentElement;
    }
    if (!card) continue;

    const lines = (card.innerText || '').split('\n').map(l => l.trim()).filter(Boolean);
    // Filter out chrome lines.
    const filtered = lines.filter(l =>
      !/^(message|connected on|view profile|remove connection|more options)/i.test(l)
    );
    const name = filtered[0] || null;
    // Headlines can wrap across multiple lines — join everything after the name and
    // strip a trailing "Message" if it leaked in.
    let headline = filtered.slice(1).join(' ').trim().replace(/\s*Message\s*$/, '').trim();
    if (headline.length > 400) headline = headline.slice(0, 400);

    if (name) seen.set(href, { linkedin_url: href, name, headline });
  }
  return Array.from(seen.values());
})()
```

**Two pitfalls that bit the first implementation:**

1. **`javascript_tool` truncates large return values around 1.5-2 KB.** A 250+ connection extraction returns ~40 KB of JSON, which won't fit. The fix: store the result on `window.__connections` and read it back another way. Easiest: trigger a Blob download to `~/Downloads/` (Chrome doesn't prompt for known downloads paths), then read the file via Bash:

   ```javascript
   const blob = new Blob([JSON.stringify(window.__connections)], { type: 'application/json' });
   const url = URL.createObjectURL(blob);
   const a = document.createElement('a');
   a.href = url; a.download = 'job-seeker-connections-dump.json';
   document.body.appendChild(a); a.click(); document.body.removeChild(a);
   ```

   Then in Bash: `cat ~/Downloads/job-seeker-connections-dump.json | python3 merge.py`. Clean up the dump file after Step 6.

2. **Two `/in/` anchors per card.** Each card renders the profile link twice (profile photo + name text), but only one "More actions for <Name>" button per card. Use `button[aria-label^="More actions for"]` for accurate card counts during pagination; use the `/in/` anchors only for URL extraction, dedup'd via a `Map` keyed on the normalized URL.

---

## Step 6 — Merge with existing data and write the CSV

For each scraped `{linkedin_url, name, headline}`:

1. **If `linkedin_url` already exists in the loaded existing-rows dict:** preserve the existing `rating`, `last_interaction`, and `notes`. Update `name` and `headline` to the scraped values (those can legitimately change — name change, job change). This is how re-running the skill preserves user ratings.

2. **If it's a new row:** add it with `rating=Unrated`, `last_interaction=""`, `notes=""`. Then check the seed names from Step 2 — if the scraped `name` matches one (case-insensitive, trimmed), upgrade `rating=High`.

   - Important: **only seed `High` on brand-new rows**, never overwrite an existing rating. If a name is in the seed file but the user previously rated them `Low` via `/rate-connections`, respect the user's rating.

3. Write the merged result using Python's `csv` module (safer than hand-building the file — names contain commas, headlines contain quotes).

   **Read the schema from the file; never hardcode it.** If `connections.csv` exists, its own header is the truth — take `fieldnames` from `csv.DictReader.fieldnames` and write those columns back. Only fall back to the default order below when creating the file from scratch:

   ```
   linkedin_url,name,headline,rating,last_interaction,notes
   ```

   This is not hypothetical. This file has already drifted from that default — it now carries a cached `member_id` column, and `rating`/`headline` sit in the opposite order. Hardcoding the documented list against the real file doesn't fail cleanly: `DictWriter` raises **after** `open(path,'w')` has already truncated the file to a bare header. That has happened once, and only a backup saved the ratings.

   **Write to a temp file, verify, then swap.** Never write in place. `config/connections.csv` is gitignored hand-entered judgment — there is no git history to recover it from, and no way to regenerate the ratings.

   ```bash
   python3 - <<'PY'
   import csv, json, sys, os

   SRC = 'config/connections.csv'
   DEFAULT = ['linkedin_url','name','headline','rating','last_interaction','notes']

   fields = DEFAULT
   if os.path.exists(SRC):
       with open(SRC) as f:
           fields = csv.DictReader(f).fieldnames or DEFAULT   # the file's own schema wins

   rows = json.loads(sys.stdin.read())          # merged rows, dicts
   rows.sort(key=lambda r: (r.get('name') or '').lower())

   # Preserve any column this skill doesn't know about (don't drop what you didn't write).
   for r in rows:
       for k in fields:
           r.setdefault(k, '')

   tmp = SRC + '.tmp'
   with open(tmp, 'w', newline='') as f:
       w = csv.DictWriter(f, fieldnames=fields, quoting=csv.QUOTE_MINIMAL, extrasaction='ignore')
       w.writeheader(); w.writerows(rows)

   check = list(csv.DictReader(open(tmp)))      # verify before it goes near the real file
   assert len(check) == len(rows), f'row count changed: {len(check)} != {len(rows)}'
   assert all(len(r) == len(fields) for r in check), 'ragged row'
   os.replace(tmp, SRC)                         # atomic
   print(f'wrote {len(check)} rows, {len(fields)} columns')
   PY
   ```

   Pipe the JSON payload of merged rows into stdin. Rows sort by `name` ascending — it makes the CSV easier to scan by hand in Excel.

4. **Back up before the first write of a session** if the file already exists: `cp config/connections.csv config/connections.csv.bak-$(date +%Y%m%d-%H%M%S)`. It's gitignored, so the backup stays local. Cheap insurance on the one file here that can't be regenerated.

---

## Step 7 — Report to the user

Print a short summary:

```
LinkedIn connections sync — done.
  Total scraped: 487
  Already in CSV: 412 (ratings preserved)
  Newly added: 75
  Seeded as High from meetings log: 12
  Now in config/connections.csv: 487 rows
    High: 31 / Medium: 22 / Low: 14 / Unrated: 420
```

Then point the user at next steps:

> To rate the 420 Unrated connections, you have two paths — pick whichever fits your style:
>
> 1. Run `/rate-connections` for an interactive walk-through, one connection at a time.
> 2. Open `config/connections.csv` in Excel, Numbers, or any spreadsheet tool, sort by headline or company, and bulk-edit the `rating` column. Save when you're done. (`/network-check` and `/score-jobs` will pick up the ratings on their next run regardless of which path you use.)

---

## Re-running this skill

Re-running `/build-connections` weeks or months later is supported and expected. The merge logic in Step 6 preserves all existing ratings, last_interaction values, and notes. The only changes on a re-run are:

- New connections get appended with `rating=Unrated`.
- Existing connections' `name` and `headline` get refreshed (people change jobs).

Nothing destructive happens on a re-run. If something looks wrong, the user can always check git history of the file — but since `config/connections.csv` is gitignored, that's a "save a backup yourself" situation. The skill itself does not back up the CSV before overwriting.

---

## LinkedIn UI dependencies (for future maintenance)

This skill assumes the following about LinkedIn's UI as of June 2026 (field-tested against a real 270-connection account). When LinkedIn changes things, update this list:

1. **Connections page URL** — `https://www.linkedin.com/mynetwork/invite-connect/connections/`. Canonical "my connections" page; stable for years.

2. **Total count header** — the page renders a line of the form `"<N,N> connections"` (lowercase 'c') near the top. We parse this for the total denominator.

3. **Scroll container is `<main>`, not `<body>`.** The connections list lives inside the `<main>` element which has its own `overflow-y: auto`. `document.body` is essentially viewport-sized; `document.querySelector('main')` is where the cards stack.

4. **Infinite scroll requires real wheel events.** LinkedIn uses an intersection observer that fires on user-driven scroll events. Programmatic `element.scrollTop = ...` and `element.scrollIntoView()` do *not* trigger the load. The Chrome MCP `computer` tool's `scroll` action emits real wheel events that LinkedIn's observer picks up. There is no "Show more" / "Load more" button anywhere on the page — pure infinite scroll.

5. **Per-card structure** — each card contains:
   - Two `<a href="/in/<slug>/">` anchors (profile photo + name link)
   - Exactly one `<button aria-label="More actions for <Name>">` — the most reliable selector for **counting cards**
   - A "Message" action button
   - Plain text lines for name, headline, and "Connected on <date>"

   Class names are obfuscated server-side and change. The text-line ordering inside a card is stable: name first, then headline (possibly multi-line), then "Connected on <date>". Filter out chrome lines (`Message`, `Connected on`, `View profile`, `Remove connection`, `More options`) and take the first remaining line as the name, joined remainder as the headline.

6. **No bulk export endpoint via MCP.** LinkedIn does offer a data export under Settings → Data Privacy, but it requires a captcha-protected manual download and is intentionally slow (24 hours for some accounts). For an MCP-driven flow, the connections page is the only realistic source.

7. **Card count vs declared total.** Off-by-one or off-by-two between the declared "N connections" header and the rendered card count is normal — ghosts of recently removed connections, account pauses, etc. Don't treat exact match as a precondition for stopping the scroll loop.

All browser interaction is confined to Steps 1, 3, 4, and 5. Nothing browser-related lives elsewhere in this file.
