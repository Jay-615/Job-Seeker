---
name: map-company-connections
description: For one target company (from config/target_companies.csv or a job listing), map Jay's 2nd-degree LinkedIn connections there and the 1st-degree "bridges" who connect him to each. Runs the 2nd-degree People search, and — critically — when a result card truncates its mutuals ("…and N other mutual connection(s)") drills into that person's shared-connections page to enumerate ALL bridges (a hidden one may be the warmest). Matches every bridge to config/connections.csv ratings and appends rows to config/company_connections.csv. Company-first and cheap: one search per company plus one drill-in only per truncated card.
---

# /map-company-connections — who do I know at company X, and how warm is the path

You take a company, find the ~N (LinkedIn caps at ~100) 2nd-degree connections who work
there, and for each one record **which of Jay's 1st-degree connections bridges to them** and
**how warm that bridge is** (rated in `config/connections.csv`). The output is a company-keyed
table Jay can sort by "strongest referral path."

This is the **company-first** replacement for the older `/network-check` (Tier-1 counts) and
`/research-network` (Tier-2 profile-visiting) skills. Prefer this one. It answers the question
Jay actually cares about: *"for a company I want to work at, who can give me a real, warm intro?"*

**Why the drill-in matters (the whole point).** A 2nd-degree result card only *names* 2–3 mutual
connections and then says "…and 21 other mutual connections." One of those hidden others may be a
**High-rated** bridge — Jay's actual best path. Naming only the visible 2–3 undersells or misreads
the connection. So when a card is truncated, we open that person's shared-connections page to get
the complete bridge list.

This is not hypothetical — it was field-verified on a real company scan, where two separate cards
each hid a better-rated bridge than the ones they named. Illustrated with the example persona from
`config/connections.example.csv`: a card reading "Felipe Marquez, Casey Lin +1" — both rated **Low**
— hides Mira Sato, rated **High**. Read the card at face value and you'd chase your two weakest
paths while your best one sat in the "+1."

---

## Parameters

- **company** — a company name (e.g. `On`, `Adidas`) matched against `config/target_companies.csv`,
  OR a LinkedIn numeric company id directly (e.g. `2912258`), OR the company from a specific job
  listing Jay is considering.
- If no argument, list High-priority rows from `config/target_companies.csv` that don't yet have a
  `company_connections.csv` block and ask via `AskUserQuestion` which to map.

---

## Step 0 — Preflight

1. **Files.** Confirm `config/connections.csv` exists (needed for bridge ratings) and
   `config/target_companies.csv` exists. If `connections.csv` is missing, warn that all bridges will
   read "Unrated" and suggest `/build-connections` first (but you can still proceed).
2. **Chrome.** Assume the Claude-in-Chrome extension is connected and Jay is logged into LinkedIn
   (per Jay's standing preference — don't pre-flight nag). If a browser tool later reports "not
   connected," reconnect with `tabs_context_mcp{createIfEmpty:true}` and retry; the drop is usually
   transient and `localStorage` survives it.
3. **Today.** Compute `today_iso = YYYY-MM-DD` for the `scanned_date` column.

Pace all LinkedIn navigation at human speed (~2–3s between actions). If you hit a CAPTCHA,
security-check, or rate-limit page, **stop**, save whatever's in `localStorage`, and report honestly.
Never solve a CAPTCHA.

---

## Step 1 — Resolve the company's LinkedIn numeric id

If given a numeric id, skip to Step 2. Otherwise:

1. If the company row in `target_companies.csv` already has a `linkedin_company_id`, use it.
2. Else find it: navigate to `https://www.linkedin.com/search/results/companies/?keywords=<company>`,
   pick the right entity by name + industry + HQ (e.g. On = slug `on-ag`, Retail, Zurich).
3. Navigate to `https://www.linkedin.com/company/<slug>/people/` and extract the **dominant**
   `fsd_company:<digits>` id (the company's own id appears far more often than "similar companies"):

   ```javascript
   (function(){
     const html=document.documentElement.innerHTML, counts={};
     let m; const re=/(?:fsd_company|company):(\d{3,})/g;
     while((m=re.exec(html))) counts[m[1]]=(counts[m[1]]||0)+1;
     return Object.entries(counts).sort((a,b)=>b[1]-a[1]).slice(0,6);
   })()
   ```

   The top id (by a wide margin) is the company id. Write it back to `target_companies.csv` in Step 6.

---

## Step 2 — Scrape the 2nd-degree People search

2nd-degree search URL (paginate with `&page=<n>`; LinkedIn caps at ~10 pages / ~100 results):

```
https://www.linkedin.com/search/results/people/?currentCompany=%5B%22<id>%22%5D&network=%5B%22S%22%5D&page=<n>
```

**Setup once** (defines the scraper, stores its source in `localStorage` so each page can replay it
with a tiny `eval`, and clears any prior run). Run this as a standalone `javascript_tool` call:

```javascript
localStorage.removeItem('mcc');
window.scrapeCo=function(){
  const COMPANY='<Company Name>';                          // <-- set per run
  const main=document.querySelector('main');
  const lines=main.innerText.split('\n').map(s=>s.trim()).filter(Boolean);
  const noise=/mutual connection|followers|^view |^connect$|^message$|^follow$|results$/i;
  const cards=[];
  for(let i=0;i<lines.length;i++){
    const dm=lines[i].match(/•\s*(1st|2nd|3rd)\b/i); if(!dm) continue;   // handles "Name • 2nd" AND split "• 2nd"
    const before=lines[i].slice(0,dm.index).trim();
    const name=before?before:lines[i-1];
    if(!name||noise.test(name)) continue;
    const h=(lines[i+1]||'').slice(0,180);
    let loc=''; const cand=lines[i+2]||'';
    if(cand&&!noise.test(cand)&&!/•\s*(1st|2nd|3rd)/i.test(cand)) loc=cand;
    let mutual='';
    for(let j=i+1;j<Math.min(i+8,lines.length);j++){ if(/mutual connection/i.test(lines[j])){mutual=lines[j];break;} }
    let other=0; const om=mutual.match(/and\s+(\d+)\s+other\s+mutual connections?/i); if(om) other=parseInt(om[1]);
    cards.push({n:name,h:h,loc:loc,m:mutual,other:other});
  }
  const anchors=Array.from(main.querySelectorAll('a[href*="/in/"]'));
  for(const c of cards){
    let slug=null, anchorEl=null;
    for(const a of anchors){const at=(a.innerText||'').trim();
      if(at===c.n||at.startsWith(c.n+'\n')||at.startsWith(c.n+' ')){const mm=a.href.match(/\/in\/([^\/?]+)/); if(mm){slug=mm[1];anchorEl=a;break;}}}
    c.u=slug; c.urn=null;
    if(anchorEl){let card=anchorEl;for(let i=0;i<6&&card;i++){const t=(card.innerText||'').length;if(t>60&&t<600)break;card=card.parentElement;}
      if(card){const um=(card.outerHTML||'').match(/ACoAA[A-Za-z0-9_-]{10,}/); if(um)c.urn=um[0];}}  // exactly one URN per card = the person's
  }
  const store=JSON.parse(localStorage.getItem('mcc')||'[]');
  const seen=new Set(store.map(e=>e.u));
  let added=0;
  for(const c of cards){if(!c.u||seen.has(c.u))continue;seen.add(c.u);
    store.push({co:COMPANY,n:c.n,h:c.h,loc:c.loc,u:c.u,urn:c.urn,m:c.m,other:c.other,bridges:null});added++;}
  localStorage.setItem('mcc',JSON.stringify(store));
  return {added,total:store.length,hasNext:lines.indexOf('Next')>-1};
};
localStorage.setItem('mccrun','('+window.scrapeCo.toString()+')()');
'setup ok';
```

**Then crawl the pages.** Use `browser_batch` to chain, per page: `navigate` → `computer.wait(3)` →
`javascript_tool` running `eval(localStorage.getItem('mccrun'))`. Do ~5 pages per batch. Notes:

- **No scrolling / screenshots needed.** Navigate + a ~3s wait renders all 10 cards; `main.innerText`
  captures them. (Scrolling was only needed on the infinite-scroll *connections* page, not search.)
- Each `eval` returns a small `{added,total,hasNext}` — no truncation. Stop when `hasNext:false` or a
  page returns `added:0`.
- Because navigation resets `window`, the scraper is replayed from `localStorage['mccrun']` each page.

Example single-page action inside a batch:
`{"name":"javascript_tool","input":{"action":"javascript_exec","tabId":<id>,"text":"eval(localStorage.getItem('mccrun'))"}}`

---

## Step 3 — Drill only the truncated cards

Find the people whose card hid mutuals (`other > 0`) and get their **full** bridge list:

```javascript
(function(){
  const store=JSON.parse(localStorage.getItem('mcc')||'[]');
  return store.filter(e=>e.other>0).map(e=>({n:e.n,other:e.other,urn:e.urn}));
})()
```

Inline returns **redact** the `ACoAA…` URN (base64 filter). To get the drill URLs in plaintext,
render them into the page and read with `get_page_text` (see Step 4 for why this channel works):

```javascript
const trunc=JSON.parse(localStorage.getItem('mcc')||'[]').filter(e=>e.other>0);
document.body.innerHTML='';
const p=document.createElement('pre');
p.textContent=trunc.map(e=>e.n+' ||| https://www.linkedin.com/search/results/people/?network=%5B%22F%22%5D&connectionOf=%5B%22'+e.urn+'%22%5D').join('\n');
document.body.appendChild(p); 'rendered';
```

`get_page_text` → copy each person's shared-connections URL. For each truncated person, navigate to
their URL (this is the "X mutual connections" page = people who are connections of BOTH Jay and them
= all Jay's 1st-degree bridges), wait ~3s, and scrape the 1st-degree names:

```javascript
(function(){
  const main=document.querySelector('main');
  const lines=main.innerText.split('\n').map(s=>s.trim()).filter(Boolean);
  const names=[];
  for(let i=0;i<lines.length;i++){
    const bi=lines[i].indexOf('•'); if(bi<0) continue;
    const after=lines[i].slice(bi+1).trim(); if(after.indexOf('1st')!==0) continue;
    const before=lines[i].slice(0,bi).trim(); const name=before?before:lines[i-1];
    if(name && name.toLowerCase().indexOf('mutual')<0) names.push(name);
  }
  return {count:names.length, names:names};
})()
```

Write each drilled person's `bridges` array back into `localStorage['mcc']` (match by name or slug).
If a drill returns 0 names, the person hides their connections — leave `bridges` null and fall back
to the card's named mutuals; note it.

> Only drill cards with `other > 0`. Cards that already show all their mutuals (no "+N other") need
> no drill — their `bridges` stay null and Step 5 parses the card's mutual line instead.

---

## Step 4 — Get the full dataset out of the browser

Two hard constraints, both field-learned — do NOT fight them:

- `javascript_tool` **inline returns truncate at ~1 KB** and redact base64-looking strings.
- Chrome **blocks a site's 2nd+ automatic Blob download**, so the download trick is one-shot per page
  load and unreliable here.

**Reliable channel:** render the whole array into a `<pre>` and read it with `get_page_text`
(returns the full text, unredacted):

```javascript
document.body.innerHTML='';
const p=document.createElement('pre');
p.textContent=localStorage.getItem('mcc');
document.body.appendChild(p); 'rendered '+JSON.parse(localStorage.getItem('mcc')).length;
```

Then call `get_page_text` and write the returned JSON **verbatim and complete** to
`jobs/.mcc-raw.json`. **Never abbreviate, summarize, or invent rows** when transcribing — copy every
record exactly, then delete the scratch file after Step 5.

---

## Step 5 — Match bridges to ratings and append to company_connections.csv

Run a Python pass (via `Bash`) that:

1. Loads `config/connections.csv` into `name -> rating` (High/Medium/Low/Unrated).
2. For each person: `bridges = drilled list if present, else parse the card's mutual line`.
   Parse the mutual line with a **dictionary-first** match so credential-suffixed names survive
   (e.g. "Priya Natarajan, MBA", "Greta Lindqvist, PHR, SHRM-CP"): scan the known connection names
   (longest-first) as substrings and remove them, then split the leftover on `,` / ` and ` for any
   names not in `connections.csv` (those get "Unrated"). Splitting on commas first would turn
   "Priya Natarajan, MBA" into two people, one of them named "MBA".
3. Per bridge: look up rating (default "Unrated"). `bridge_best_rating` = max (High>Medium>Low>Unrated).
4. Emit rows to `config/company_connections.csv` with schema:
   `company, person_name, title, degree, bridge_connections, bridge_best_rating, person_linkedin_url, notes`
   - `bridge_connections` = semicolon list, ordered best-rating-first.
   - `notes` = `full bridge list via drill-in (card hid +N)` when drilled; `rated: <name (rating), …>`;
     optionally `ex-Nike` if the headline flags it. Keep the raw headline verbatim in `title`.
5. **Idempotent append:** read the existing file, drop any rows where `company == <this company>`, add
   the new rows (sorted `-rating, bridge, name`), write back. This makes re-runs safe.

`config/company_connections.csv` is gitignored (via `config/*`), so it's safe for real network data.

---

## Step 6 — Write-back and report

1. If Step 1 resolved a new company id, write it into that company's `linkedin_company_id` in
   `config/target_companies.csv`, and add a short note (row count + warmest bridges + date).
2. Delete `jobs/.mcc-raw.json`.
3. Report to Jay (a results report — audience is non-technical, keep it plain): total people mapped,
   count of High / Medium / Low / Unrated best-bridge paths, the top bridges by reach, and the list
   of High/Medium paths (person — title — via bridge). Call out any bridges that surfaced but are
   missing from `connections.csv` (worth adding), and any people whose connections were hidden.

---

## Failure modes

- **"No results found" on the 2nd-degree search** → Jay has 0 2nd-degrees at that company, or the
  company id is wrong. Re-verify the id from `/company/<slug>/people/`.
- **Drill-in returns 0** → that person hides their connections. Fall back to the card's named mutuals;
  note "connections hidden."
- **Chrome disconnects mid-crawl** → reconnect (`tabs_context_mcp{createIfEmpty:true}`); `localStorage`
  persists, so resume from the next page. Dedup by slug makes re-scraping harmless.
- **CAPTCHA / rate-limit** → stop, extract what's in `localStorage`, report honestly, recommend a
  cool-down. Do not retry in-session.
- **A URN won't extract from a card** (rare) → visit that person's profile and pull the base `ACoAA…`
  id from the page HTML as a fallback.

---

## LinkedIn UI dependencies (as of mid-2026 — update here when LinkedIn changes)

1. **2nd-degree search URL** — `.../search/results/people/?currentCompany=%5B%22<id>%22%5D&network=%5B%22S%22%5D`, pages via `&page=<n>`, ~100 cap.
2. **Shared-connections URL** — `.../search/results/people/?network=%5B%22F%22%5D&connectionOf=%5B%22<person_member_urn>%22%5D`. `network=["F"]` + `connectionOf=[X]` = people who are 1st-degree to Jay AND connections of X = the mutual/bridge set.
3. **Company id** — dominant `fsd_company:<digits>` on `/company/<slug>/people/`.
4. **Result card** — name line may be `Name • 2nd` (combined) or `Name` then `• 2nd` (split); handle both. Card holds **exactly one** `ACoAA…` member URN (the person's); mutual avatars carry none.
5. **Mutual-line shapes** — `"<A> is a mutual connection"`, `"<A> and <B> are mutual connections"`, `"<A>, <B> and N other mutual connections"`, `"<A>, <B>, and <C> are mutual connections"`.
6. **Extraction** — inline `javascript_tool` returns cap ~1 KB and redact base64; use `localStorage` + `<pre>` + `get_page_text`. One automatic Blob download per page load only.

All browser interaction is confined to Steps 1–4. Nothing browser-related lives elsewhere.

## Relationship to other skills — division of labor

This skill does not replace `/network-check` or `/research-network`; it splits the work. It is the
**find-and-document** step; the other two **consume** what it produces:

- **`/map-company-connections` (this skill) — FIND & DOCUMENT.** The canonical way to discover Jay's
  2nd-degree connections at a company and the bridges to them, drill-in-complete. Writes the
  source-of-truth table `config/company_connections.csv`. Run it per company (from the target list, or
  when a newly-scanned job posting is at a not-yet-mapped company).
- **`/network-check` — SCORE.** Computes the network-proximity sub-score `/score-jobs` needs. Going
  forward it should read this skill's `company_connections.csv` block for the company (more complete
  than its own page-1 scrape, which misses the "+N other" truncation) and only scrape live if the
  company hasn't been mapped yet.
- **`/research-network` — PLAN OUTREACH.** For a job Jay is actively pursuing, uses
  `company_connections.csv` as its candidate pool, then does the per-profile area-relevance judgment
  and the Top-3 outreach plan — the expensive work this skill deliberately doesn't do.

So: **map = find/document, network-check = score, research-network = plan outreach.** The table is the
shared hand-off between them.

- Consumes `config/connections.csv` (1st-degree, rated — built by `/build-connections`, rated by
  `/rate-connections`). A cached `member_id` column exists there if a bridge's URN is ever needed.
- Deliberately company-first. The alternative "crawl each warm 1st-degree's whole network to discover
  companies" was tried and rejected: capped at 100/person, mostly irrelevant companies, and many
  1st-degrees hide their connections.
