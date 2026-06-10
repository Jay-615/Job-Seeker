# Pipeline smoke findings — 2026-06-09

Findings from the LinkedIn-only smoke run of `/scan-jobs` → `/score-jobs` → `/research-network 1` before M7 polish. These are the concrete fixes M7 should fold in before the repo goes public.

---

## scan-jobs.md

### 1. Step 1b card-extraction parser is off-by-one

**Symptom.** Extracted cards had `company = "Product Manager with verification"` and `location = "Instagram"` for an Instagram PM row, where the intended values were the reverse.

**Cause.** LinkedIn's current result-card DOM interleaves a `<title> with verification` alt-label line between the visible title and the company line. The reference snippet assumed `[title, company, location]` at lines `[0, 1, 2]`. The actual order is `[title, "<title> with verification", company, location, ...]`.

**Fix.** In the cards loop, before pairing fields, filter the card's text lines to drop noise:

```javascript
const noise = /^(promoted|easy apply|viewed|be an early applicant|actively reviewing applicants|medical|401\(k\)|· \d+|\d+ applicants?|new|company logo)$/i;
const isVerifLine = (l) => /\bwith verification$/i.test(l);
lines = lines.filter(l => !noise.test(l) && !isVerifLine(l));
const title = lines[0] || ''; const company = lines[1] || ''; const location = lines[2] || '';
```

### 2. Page-1 pagination heuristic misses organic results

**Symptom.** Page 1 of the Portland search returned 7 cards (all `Promoted`). The header said 26 total results. The other 19 are organic and live on pages 2–3.

**Cause.** Skill's heuristic says: "Don't paginate beyond the first page unless the first page returned the maximum result count (usually 25)." Page 1 returned 7, so the skill stops. But Promoted cards occupy the top of page 1 and organic cards are paginated underneath.

**Fix.** Replace the heuristic with: "If page 1 returns fewer than the header total AND a `Page 2` link is present in the pagination strip, paginate to page 2. Stop at page 3 or when no `Next` is visible." Add a pagination URL formatter and reuse the same extractor.

### 3. "Promoted by hirer · Responses managed off LinkedIn" postings have no JD body

**Symptom.** Tried to extract JDs from 3 sampled postings (Instagram, Nike, Roadie); all returned `main.innerText` of ~1.3K chars containing only meta header + footer. No `About the job` section. No `[class*=jobs-description]` div in the DOM.

**Cause.** Cards flagged "Promoted by hirer · Responses managed off LinkedIn" link out to the hirer's external ATS for the description and apply flow. LinkedIn renders the meta header and an Apply button but does not host the JD inline.

**Fix.** In Step 1c, detect the pattern before attempting body extraction:

```javascript
const off_linkedin = /responses managed off linkedin/i.test(main.innerText);
```

If true, skip the body fetch, set `jd_body = ""`, and add a `Notes` line on the inbox file: `"JD body not on LinkedIn page (Promoted by hirer + off-LinkedIn responses). Follow the URL above and click Apply to read the full JD."` Optionally (post-v1): grab the `Apply` button's `href`, navigate to the hirer's ATS, and try to extract the JD there.

### 4. Step 1c h1 assumption is wrong

**Symptom.** `document.querySelector('h1').innerText` returned the **company name** ("Nike", "Instagram", "Roadie") instead of the job title.

**Cause.** LinkedIn rev — the page's h1 is now the company name; the job title lives one line below in `main.innerText`.

**Fix.** Parse `document.title`, which is reliably formatted as `<job title> | <company> | LinkedIn`:

```javascript
const m = document.title.match(/^(.+?)\s\|\s(.+?)\s\|\sLinkedIn$/);
const title = m ? m[1] : null;
const company = m ? m[2] : null;
```

Update **LinkedIn UI dependency #3** in the skill to document this.

---

## research-network.md

### 5. Step 6 candidate URL pairing bug — critical

**Symptom.** First extraction had candidate #1's `profile_url` pointing at `/in/<mutual-a>` — a mutual connection's profile, not the candidate's. Candidate #2 pointed at `/in/<mutual-b>`, a different mutual. And so on: all 9 cards beyond the first had wrong URLs.

**Cause.** Each result card's DOM contains `<a href="/in/...">` anchors not only for the candidate but also for the **named mutual connections** in the "X, Y and N other mutual connections" text. The skill's document-order pairing (`profile_urls[urlIdx]` paired with `cards[urlIdx]`) breaks because the mutual anchors are interleaved with candidate anchors.

The skill's UI-dependency block already says: *"verify that cards.length matches profile_urls.length and bail with a parsing error if not."* That check was documented but **not enforced** in the reference snippet — and even if enforced, the check would fail noisily without actually fixing the pairing.

**Fix.** Replace document-order pairing with **name-match anchor scoping**. For each text-parsed card name, walk `main.querySelectorAll('a[href*="/in/"]')` once and pick the first anchor whose `innerText` matches the card name (`startsWith` to tolerate trailing degree/verification badges inside the anchor):

```javascript
const all_anchors = Array.from(main.querySelectorAll('a[href*="/in/"]'));
const enriched = text_cards.map(c => {
  let url = null;
  for (const a of all_anchors) {
    const atext = (a.innerText || '').trim();
    if (atext === c.name || atext.startsWith(c.name + '\n') || atext.startsWith(c.name + ' ')) {
      url = a.href.split('?')[0].replace(/\/$/, '');
      break;
    }
  }
  return { ...c, profile_url: url };
});
```

Verified in the field — produced 9/9 correct URLs on Nike's 2nd-degree People search where the original snippet produced 1/9.

Update **LinkedIn UI dependency #2** in the skill to describe name-match pairing instead of document-order pairing. Keep a fallback `missing_url_count` count and bail if any cards lack URLs.

### 6. Step 7 top-card headline skip-list misses `· 2nd` / `· 3rd`

**Symptom.** On one candidate's profile, the parser returned `headline: "· 2nd"` instead of the actual headline.

**Cause.** Profile pages render the candidate's degree marker (`· 2nd`, `· 3rd`) right below the name. The skill's skip-list excludes pronouns and "Verified" but doesn't skip the degree marker.

**Fix.** Add the degree-marker forms to the skip-list:

```javascript
if (/^•?\s*(1st|2nd|3rd)$/i.test(s)) continue;
if (/^·\s*(1st|2nd|3rd)$/i.test(s)) continue;
if (/^·$/.test(s)) continue;
```

### 7. Step 7 About-section detector is unscoped

**Symptom.** One candidate's profile has no About section. The unscoped `text.findIndex(/^About$/i)` matched the LinkedIn footer's "About" link and pulled in `["Accessibility", "Talent Solutions", "Community Guidelines", ...]` as the "About" block.

**Cause.** The About-section detector searches the entire `main.innerText`. When the profile has no About section, the global search falls through to the LinkedIn footer (which contains an "About" link). When the profile DOES have an About section, the global search works correctly because the profile's About appears before the footer.

**Fix.** Scope the About search to before the Experience section. The profile body's "About" always appears above Experience; the footer's "About" appears far below:

```javascript
const expIdx = text.findIndex(l => /^Experience$/i.test(l));
const aboutSearchSlice = expIdx >= 0 ? text.slice(0, expIdx) : text.slice(0, 30);
const aboutLocal = aboutSearchSlice.findIndex(l => /^About$/i.test(l));
const about_block = aboutLocal >= 0 ? aboutSearchSlice.slice(aboutLocal + 1, aboutLocal + 8) : [];
```

---

## Pipeline-level observations (no code fix; for the README / docs)

- **Smoke scope.** This run did LinkedIn only, page 1, Portland Product Manager — 7 cards → 5 inbox files → 5 scored → research on rank 1 (Nike). All three skills exercised end-to-end. Per-source token cost on this slice was modest (~80K tokens estimated).
- **Network cache is doing real work.** Nike's Tier 1 cache (from 2026-06-08) was hit cleanly during `/score-jobs` and reused during `/research-network` to seed candidate mutuals — saved fresh LinkedIn navigation for the most-relevant company.
- **`connections.csv` ratings are the next big lever.** All Tier 1 Nike mutuals are still `Unrated`. Rating just the three mutuals who recur across every Nike candidate's list lifts Nike's Tier 1 score from 70 to 100. The product workflow should nudge the user toward `/rate-connections` early.
- **Promoted-off-LinkedIn coverage gap.** All 7 cards in the page-1 sample were "Promoted · off LinkedIn." If this pattern persists across Jay's regular search slice, the inbox files will frequently be metadata-only and experience-match scoring will be thinner than the README implies. Either (a) build the apply-URL-follow path in fix #3, or (b) call out the limitation honestly in the scoring methodology doc.
