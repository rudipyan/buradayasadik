# Audit — "buradayaşadık" counter-map prototype (DTES544)

**Project root:** `/Users/rudis/Library/Mobile Documents/iCloud~md~obsidian/Documents/1-Academic/3-Courses/DTES544/4.codes`
**Repo:** `git@github.com:rudipyan/buradayasadik.git` — **PUBLIC** (verified via `gh repo view`), 55 commits, single branch `main`
**Audited:** 2026-07-24. Read-only. No project file was modified.
**Scope of evidence:** every claim below is anchored to a file and line I opened, or to a command I ran. Where I could not verify something, I say so.

---

## Executive summary

1. **A live, URL-unrestricted Mapbox token is committed in a public repo** (`index.html:1018`). I verified it is still valid and carries no referrer restriction: an unauthenticated `curl` with no `Referer` header returned `HTTP 200` and `{"code":"TokenValid","usage":"pk","user":"rudipyan"}`. Anyone can lift it and bill tiles to the account. Two older tokens remain in history. **High.**
2. **The admin password is recoverable in plaintext from public git history** — commit `39766cd` ("security: replace plaintext admin password with SHA-256 hash") shows the removed literal in its own diff, and the replacement hash is an unsalted SHA-256 of that same short string. The app has no server so the blast radius is local, but the credential looks reusable. **High.**
3. **The argument has almost no data behind it.** 17 of the 18 seed narratives are lorem ipsum (`index.html:1040, 1053, 1066, 1079, 1093, …`). Only story 18 (`:1272-1277`) carries real testimony. There is no provenance field anywhere in the schema — no interview date, transcript ID, consent record, or note on who assigned the emotion tags. A thesis reader cannot trace a dot on the map back to a source. **High.**
4. **There is no non-map representation of the content for a visitor.** The only list view of the testimonies sits inside the admin modal (`:927-939`), behind `Ctrl+Shift+A` and a password. Screen-reader users, keyboard-first users, and anyone whose device can't run WebGL get nothing. For a project arguing about erasure, this is the finding that most undercuts itself. **High.**
5. **The entire interface is built inside `map.on('load')`** (`:1663-2319`). If Mapbox is unreachable, the token is revoked, or the privately-hosted style 404s, the visitor sees a blank page — `map.on('error')` only writes to the console (`:2321-2323`). This is both a fieldwork risk (conference wifi) and the central archival risk. **High.**
6. **123 KB of tracked duplication:** `9 June/index.html` is byte-identical to `index.html` (both `md5 5a467e05cb58f8874883b95c49e0e1e7`) and both are tracked in git. `12 May/` is a third, older copy, untracked and unreferenced. **High** (as a maintenance and archival hazard, not a runtime one).
7. **The basemap is the thing the project is supposed to be arguing against.** The custom style `mapbox://styles/rudipyan/cmr9mbchq000m01qv1mf9gbmm` is a 50-layer derivative of Mapbox "Light" with 14 label layers, `poi-label` on from z6, admin boundaries, and — decisively — `text-field: ["coalesce", ["get","name_en"], ["get","name"]]`. **The substrate labels Üsküdar in English first, Turkish second, and Armenian never**, including when the UI is switched to `hy`. **High** as a design finding.
8. **All six emotion chips fail WCAG AA** (3.42–3.91:1 for white 10.5px text on the fills at `:1288-1293`), and secondary text runs at `#aaa` = 2.32:1 and `#bbb` = 1.92:1 throughout. The chip failure is a *direct consequence* of the deliberate equal-OKLCH-lightness decision documented at `:1282-1286` — a good political choice that produced a uniform accessibility failure. **Medium.**
9. **Zero responsive breakpoints.** Both `@media` blocks in the file are `prefers-reduced-motion` (`:702`, `:783`). `#video-panel` is `width: 480px` with no `max-width` (`:409`) and renders off-screen on any phone; `.controls` is a fixed 288px open by default (`:827`), covering 77% of a 375px viewport at load. Fieldwork demos happen on phones. **Medium.**
10. **It will not run in five years as-is.** Vendoring `mapbox-gl.js`/`.css`/the Armenian woff2 was the right instinct and deserves credit, but the style, tiles, glyphs and sprites are all still fetched from `api.mapbox.com`, so the two "works offline" comments (`:8`, `:1010`) are false. Add a missing README, a token that will rotate, and a YouTube embed (`:1277`) that can be taken down. **High** for an artifact meant to accompany a thesis.

---

# PART A — CODE AUDIT

## A1. Structure

### The measurement

`index.html` is 2,846 lines / 120.3 KB. Broken down by boundary (`grep -n '<style\|</style>\|<script\|</script>'`):

| Region | Lines | Count |
|---|---|---|
| `<head>` + meta + vendored CSS link | 1–10 | 10 |
| **Inline CSS** | 12–787 | **777** |
| **Body markup** | 790–1010 | **221** |
| **Inline JS** | 1014–2841 | **1,829** |
| closing tags | 2842–2846 | 4 |

Supporting counts across the file: 29 top-level `function` declarations, one ~660-line `map.on('load')` closure holding another 12 nested functions, 77 `document.getElementById` calls, 19 inline `onclick=` attributes, 8 inline `style=` attributes in markup.

### Responsibilities tangled in one file

Twelve, and they are genuinely distinct concerns, not slices of one:

1. Seed data (`:1027-1279`, ~250 lines)
2. i18n dictionaries for tr/en/hy (`:1309-1511`, ~200 lines)
3. localStorage seed/load/migrate/version (`:1591-1631`, `:2355-2389`)
4. Filter UI construction and isolate-on-click logic (`:1680-1819`)
5. Canvas particle renderer + physics (`:1907-2056`)
6. Ghost haunting scheduler + catch/release mechanic (`:2058-2137`)
7. Marker lifecycle and zoom-bucket rebuild (`:2185-2287`)
8. Popup HTML templating (`:2149-2183`)
9. Floating video panel (`:2332-2349`)
10. Admin authentication (`:2391-2443`)
11. Admin CRUD + bulk add (`:2445-2766`)
12. Entrance intro overlay (`:2789-2841`)

### Proposed split — and an honest verdict on whether to do it

The maximal split would be seven files. I do not recommend the maximal split. This is a 55-commit, single-author artifact with a defined end date; refactoring it into an app skeleton is work that pays no dividend to the thesis.

**Do these three (high value, low risk, ~half a day):**

| New file | Moves from | ~Lines | Why it pays for itself |
|---|---|---|---|
| `data/stories.json` | `SEED_STORIES` `:1027-1279` | 250 | Makes the data **citable and diffable**. You can print it as a thesis appendix, a reviewer can read it without reading code, and `git log -p data/stories.json` becomes a legible record of how the archive changed. This is the single most valuable file split available. |
| `styles.css` | `:12-787` | 777 | Not for tidiness — because the Part B design work is **not tractable** on 777 lines of literal hex. Move it out and hoist the ~13 greys and ~10 radii into `:root` custom properties in the same pass. |
| `src/admin.js` | `:2355-2766` + `:2391-2443` | ~400 | Lets you ship a public build with **no login code in it at all**, which is the correct fix for finding A3.2. |

**Skip these:** splitting `i18n.js`, `marker.js`, `map.js`. The i18n object is a flat dictionary that is easier to scan in one place; the marker renderer and map wiring share the `map.on('load')` closure and separating them means inventing a module boundary and an event bus you don't need. Doing it would cost a day and buy nothing a thesis committee will ever see.

After the three recommended moves, `index.html` drops to roughly 230 lines of markup + ~1,400 lines of JS. That is a file a person can still hold in their head, which is the actual goal.

One caveat if you split: `data/stories.json` requires `fetch()`, which does not work from `file://`. That is already true today for the admin panel (`crypto.subtle` needs a secure context — the code even says so at `:2416-2418` and the plan doc warns about it), so it changes nothing in practice, but the README must say "serve over `python3 -m http.server`, do not double-click the file."

---

## A2. Correctness and bugs

### A2.1 — Duplicated iteration folders, one of them tracked — **High**

**Evidence:** `md5 index.html "9 June/index.html"` → both `5a467e05cb58f8874883b95c49e0e1e7`. `git ls-files` shows `9 June/index.html` **is tracked**; it was added in `38c1c71` and modified again in `72c9b5b`, i.e. it is being edited in lockstep with the real file. `12 May/index.html` (`5a07d10a…`) is untracked, as are `12 May/mapbox-gl.js`, `9 June/mapbox-gl.js`, `9 June/fonts/`, `9 June/docs/` (per `git status --short`). All three copies of `mapbox-gl.js` are identical (`320f6915…`), i.e. ~2.9 MB of the same vendored library on disk.

**Why it matters beyond tidiness:** two byte-identical files under version control, both being modified, is a fork waiting to happen — a future edit lands in one and not the other and there is no signal which is canonical. And `12 May/` is genuinely different code (`grep` shows it loads Mapbox from `https://cdn.jsdelivr.net/npm/mapbox-gl@2.15.0/dist/mapbox-gl.js` at `12 May/index.html:547` and uses the stock `mapbox://styles/mapbox/light-v11` at `:777`), so it is a real historical artifact — it just doesn't belong on the working tree.

**Fix:** the iteration history you want already exists in `git log`. Tag the two milestones (`git tag iteration-12-may <sha>`, `git tag iteration-9-june <sha>`), `git rm --cached "9 June/index.html"`, delete both folders from the working tree, and add `12 May/`, `9 June/` to `.gitignore` if you want to keep local copies. If the dated folders are meant to be exhibitable snapshots for the thesis, the honest version is a `snapshots/` directory with a one-line README saying what each froze and why — not two silent duplicates.

### A2.2 — Everything is inside `map.on('load')`; failure is silent — **High**

**Evidence:** `index.html:1663` opens the closure, `:2319` closes it. Inside it: search wiring (`:1670`), `buildPeopleFilter` (`:1680`), `buildEmotionFilter` (`:1742`), `updateMarkerVisibility` (`:1837`), `updateStats` (`:1859`), the whole particle renderer (`:1907`), all marker creation (`:2252`), the marker key (`:2292`), and the `window._mapApi` export (`:2316`). The only error handling is:

```js
map.on('error', (e) => {
    console.error('Mapbox error:', e.error.message);
});                                                    // :2321-2323
```

**Consequences:** if the style request fails — revoked token, expired card on the Mapbox account, deleted style, captive-portal wifi, offline — `load` never fires. The sidebar renders its static shell but the people chips, emotion chips, marker key and story count stay empty, no markers exist, and the visitor sees a warm-grey rectangle. Nothing tells them why. Secondarily, `e.error.message` will itself throw if an error event arrives without `.error`.

**Fix:** two small changes. (a) Guard the handler and surface it: `map.on('error', e => { console.error(e?.error?.message ?? e); showFallback(); })`. (b) More importantly, **the content should not depend on the map rendering at all** — see A6.4. The register/list view proposed in Part B is the real fix here: build it outside the `load` closure, from `stories`, so the testimonies are present whether or not Mapbox answers.

### A2.3 — Unbounded `requestAnimationFrame` leak on drifting markers — **Medium**

**Evidence:** `addMarkerForStory` starts a drift loop for `locationKnown === false` stories at `:2220-2249`:

```js
function drift() {
    if (marker.getElement().style.display === 'none') { driftRafId = null; return; }
    ...
    marker.setLngLat([lng, lat]);
    driftRafId = requestAnimationFrame(drift);
}
```

`driftRafId` is captured in the closure and **never exposed for cancellation**. The teardown helper only handles the canvas loop:

```js
function destroyMarkerAnim(id) {
    const inner = markerElements[id];
    if (inner) { const canvas = inner.querySelector('canvas');
                 if (canvas?.destroyAnim) canvas.destroyAnim(); }
}                                                          // :2623-2629
```

`deleteStory` (`:2631-2648`), `saveEditedStory` (`:2605-2611`) and `seedStoriesFromHardcoded` (`:2374-2379`) all call `destroyMarkerAnim` then `marker.remove()`. After `remove()`, the detached element's `style.display` is still `''`, so the guard at `:2228` never trips and the loop runs forever, calling `setLngLat` on a removed marker every frame.

**Scope:** only story 18 currently has `locationKnown: false` (`:1269`), so today this leaks exactly one loop, and only after an admin edit or the first admin login (which calls `seedStoriesFromHardcoded`). It is not currently a visitor-facing bug. It becomes one the moment more uncertain-location stories are added — which the design explicitly wants.

**Fix:** hang the canceller on the marker next to the existing `_driftRestart` (`:2245`), and call it from `destroyMarkerAnim`:

```js
markerObjects[story.id]._driftStop = () => {
    if (driftRafId) { cancelAnimationFrame(driftRafId); driftRafId = null; }
};
```

### A2.4 — Zoom rebuild churn — **Medium**

**Evidence:** `updateMarkerScales` (`:2259-2287`) is bound to `map.on('zoom', …)` (`:2288`), which fires every animation frame during a pinch or scroll-zoom. It is bucketed to 0.25 zoom steps (`:2262-2264`), which is the right idea, but each bucket crossing rebuilds **every** marker canvas from scratch (`:2280`), and each rebuild allocates a new `<canvas>` plus 70 particle objects (`N = 70` at `:1914`, `Array.from` at `:1929-1943`).

A single z15→z20 pinch crosses 20 buckets. 20 × 18 markers = ~360 canvas allocations and ~25,200 particle objects, in one gesture. At z20 each canvas is `44 × 1.6⁴ ≈ 288px` square (`SCALE_FACTOR = 1.6` at `:2257`). This is the main GC/jank source on a phone.

**Credit where due:** the rebuild-instead-of-CSS-scale decision is correct and the comment at `:2254-2255` explains why (1:1 pixel density, no blur). The teardown at `:2284` correctly cancels the old loop — that leak was already found and fixed (commit `7afb329`).

**Fix, cheapest first:** (a) widen the bucket to 0.5 or 1.0 zoom steps — the visual difference is negligible and it halves-to-quarters the churn; (b) debounce on `zoomend` rather than `zoom` for the rebuild, and apply a temporary CSS transform during the gesture only; (c) reuse the particle array across rebuilds instead of regenerating it — the home positions just need rescaling by `newSize/oldSize`.

### A2.5 — `locationKnown` is unreachable from the UI — **Medium**

**Evidence:** the sidebar advertises three marker states, including `∿ Konum belirsiz / Location uncertain` (`:885-888`, with a live demo canvas at `:2297`). But `locationKnown` appears in only four places (`grep -n locationKnown`): the hardcoded story 18 (`:1269`), and three read sites (`:2197`, `:2220`, `:2280`). The edit form (`:942-986`) has no control for it, the bulk-add card (`:2668-2701`) has no control for it, and `saveBulkStories` never writes the field (`:2739-2743`).

**Consequence:** the third of the three states the interface teaches can never be produced by anyone using the interface. It exists only as a hardcoded property of one story. For a project whose argument depends on representing uncertain and lost locations, this is the wrong thing to have left un-wired.

**Fix:** add a checkbox next to the existing ghost checkbox in both forms, mirroring `edit-ghost` (`:967-970`), and write `locationKnown: !uncertain` in `saveEditedStory` and `saveBulkStories`.

### A2.6 — Bulk-added stories silently break the trilingual model — **Medium**

**Evidence:** seed stories carry `title` and `narrative` as `{tr, en, hy}` objects (`:1032-1036`, `:1272-1276`). `saveBulkStories` writes them as bare strings (`:2742`: `title, lat, lng, narrative, period, isGhost, emotions`). Every read site does a `typeof … === 'object'` check and falls back (`:1827-1828`, `:2166-2167`, `:2498`), so nothing crashes — but a bulk-added story displays the admin's input language in all three UI modes, permanently, with no UI path to add a translation. `mergeLangField` (`:2452-2458`) correctly protects existing translations on edit, but cannot create the object in the first place.

**Fix:** in `saveBulkStories`, write `title: { [currentLang]: title }` and the same for `narrative`, so `mergeLangField` has an object to merge into later. Two-line change.

### A2.7 — Dead code — **Low**

All confirmed by occurrence count (definition present, zero use sites):

| Dead thing | Location | Why it's dead |
|---|---|---|
| `.chip-dot` rule | `:250-257` | Removed from markup in `d9f55a2`/`4969727`; CSS left behind |
| `.no-results` rule | `:351-357` | No element ever carries the class |
| `.video-player` + `.video-player iframe` | `:391-401` | Superseded by `#video-panel` (`:404-448`) |
| `emotionData[].label` values | `:1288-1293` | Never read — all labels go through `t('emo_*')` (`:1370-1375` etc.) |
| `#ghost-toggle` hidden checkbox | `:864` | Kept only so `:1840` doesn't break; the comment admits it |
| `.emotion-check-label` class | applied at `:2560`, `:2664` | No CSS rule of that name exists; styling comes from the `.emotion-check-grid label` descendant selector (`:544`) |

**Also:** `emotionData.black` is now `#8675D4`, a violet (`:1293`). The key name is a leftover from the pre-OKLCH palette and is now actively misleading — it is the key **persisted into localStorage** and into every story's `emotions` array, so renaming it needs a migration. Flagging rather than recommending: the cost may exceed the benefit.

**Fix:** delete the four CSS/data items. Leave the `black` key alone unless you are already writing a migration.

### A2.8 — Smaller correctness items — **Low**

- `openVideoPanel` falls back to a hardcoded Turkish string (`:2340`) instead of `t('videoPanelTitle')`. Visible in EN/HY when a story has no title.
- `story.videoUrl` goes straight into `iframe.src` (`:2339`) with no scheme allowlist. Admin-only, so impact is near-zero, but a `new URL(u)` + `youtube.com` host check is one line.
- Bulk-added ids are floats: `id: Date.now() + Math.random()` (`:2740`). They round-trip through `Number()` in `seedStoriesFromHardcoded` (`:2375`) and interpolate fine into `onclick="openEditView(1753…4831)"` (`:2514`), so nothing is broken — but it is fragile and `crypto.randomUUID()` is available and better.
- Two mechanisms do the same language-switch job: inline `onclick="applyLang('tr')"` on the pill (`:794-796`) and `addEventListener` on the intro buttons (`:2832-2837`), with duplicate active-state logic in `applyLang` (`:1556-1558`) and `markActiveLang` (`:2829-2830`).
- Two `<h1>` elements: the intro heading (`:803`) and the sidebar title (`:831`).

**Positive findings worth recording:** `escapeHtml` (`:1300-1304`) is applied consistently at every `innerHTML`/`setHTML` site I checked (`:2162`, `:2170-2178`, `:2488`, `:2510-2511`). The delegated click handler for popup video buttons (`:2770-2774`) correctly replaced inline `onclick` string interpolation, with a comment explaining why. Corrupt-localStorage handling (`:1603-1612`) falls back to seed instead of blanking the map. Seed versioning (`:1615-1622`) preserves admin-added stories across a bump. These are all careful, and several are fixes for problems a previous audit found. The codebase is not sloppy; it is under-structured.

---

## A3. Secrets

### A3.1 — Live, unrestricted Mapbox public token in a public repo — **High**

**Location:** `index.html:1018` — `mapboxgl.accessToken = 'pk.eyJ1IjoicnVkaXB…MASKED…BKoqAQ';`
Also present, identically, at `9 June/index.html:1018` (tracked duplicate).

**History:** `git log --oneline -S 'pk.eyJ' --all` returns five commits; three distinct token values exist across history:

| Token (masked) | Status |
|---|---|
| `pk.eyJ1IjoicnVkaXB…MASKED…ilbL7w` | superseded — visible at `12 May/index.html:554` and in history |
| `pk.eyJ1IjoicnVkaXB…MASKED…MeUTvA` | superseded |
| `pk.eyJ1IjoicnVkaXB…MASKED…BKoqAQ` | **currently live, in HEAD** |

Commit `9b14c78` ("chore: rotate Mapbox public token") shows rotation has already happened once. Good instinct — but rotation alone does not fix this.

**Verification I ran (read-only GETs, no writes):**

```
curl -s -o /dev/null -w "%{http_code}" \
  "https://api.mapbox.com/styles/v1/rudipyan/cmr9…?access_token=$TOK"   → 200
curl -s "https://api.mapbox.com/tokens/v2?access_token=$TOK"
  → {"code":"TokenValid","token":{"usage":"pk","user":"rudipyan","authorization":"…"}}
```

Both requests were made **with no `Referer` header**, from a machine that is not the deployment origin. Mapbox enforces URL restrictions via the `Referer` header; a restricted token returns `403` under exactly these conditions. It returned `200`. **This token has no URL restriction.**

**Severity reasoning.** A `pk.` token is designed to be public — that part is fine and is not the finding. The finding is the *combination*: public repo + no URL restriction + a personal Mapbox account with a payment method or free-tier cap. Anyone who finds the repo (or a scraper — GitHub is continuously scraped for exactly this pattern) can point their own site at the token and consume the account's map-load quota. On the free tier that means the thesis prototype goes dark mid-demo when the quota trips. If a card is attached, it means a bill.

**Remediation, in order:**
1. **Restrict the token by URL** in the Mapbox dashboard (Account → Tokens → the token → URL restrictions). Add exactly the deployment origin (the GitHub Pages / Vercel host) plus `http://localhost:8000`. This is the actual fix and takes two minutes.
2. **Then rotate** — create a new restricted token, paste it at `:1018`, delete the old one. Order matters: restrict-then-rotate means there is never a live unrestricted token.
3. **Do not rewrite git history for this.** History rewriting on a public repo is disruptive and buys nothing here: the old tokens are already revoked, and the current one will be revoked in step 2. Rotation *is* the remediation for public tokens. (This is the opposite of the advice for a secret `sk.` token — I found no `sk.` token anywhere, which is correct.)
4. Optionally set a monthly quota alert on the Mapbox account so a future leak is noticed.

### A3.2 — Admin password recoverable from public history; current hash is unsalted — **High**

**Location:** current credential object at `index.html:2391-2394`:

```js
const ADMIN_CREDS = {
    username:     'admin',
    passwordHash: 'e2e7ae53…MASKED…cd289a',
};
```

**History:** commit `39766cd` ("security: replace plaintext admin password with SHA-256 hash", 2026-05-18) removes the line `const ADMIN_CREDS = { username: 'admin', password: 'uskudar2***' };` and adds the hash **in the same diff**. The plaintext is therefore permanently readable in the public repo's history to anyone who runs `git show 39766cd`. Independently, an unsalted SHA-256 of a short lowercase-plus-digits string is recoverable from any rainbow table in seconds, so the hash offers no protection even without the history.

**Actual blast radius — stated honestly:** near zero *for this app*. There is no backend. "Logging in" unlocks a modal that writes to the visitor's own `localStorage` (`saveStories` → `localStorage.setItem`, `:2355-2357`). An attacker who "breaks in" gains the ability to edit their own browser's copy of the data. Nothing is exfiltrated, nothing shared is modified.

**Why it is still High:** the password follows a `<project><year>` pattern that people reuse, and it is now permanently public. The risk is not to this app; it is to whatever else that string opens.

**Remediation:**
1. Change that password wherever else it has been used. This is the only urgent action.
2. Then decide what the login is *for*. It currently protects nothing, and its presence implies a data-protection guarantee that does not exist — a visitor could reasonably believe their edits are being submitted somewhere. **Recommended:** remove the login entirely from the public build (this is what the `src/admin.js` split in A1 buys you) and keep the admin panel in a local-only file. If you want to keep it as a demo affordance, replace the credential check with an unauthenticated "editor mode" toggle and a visible banner reading "changes are stored on this device only" — which is both honest and, given the thesis, more interesting than a fake lock.

### A3.3 — Other secret-adjacent checks — clean

- No `sk.` token, no `.env`, no `credentials.json`, no `*.pem`/`*.key` anywhere in the tree.
- `.gitignore` (2 lines) excludes `.superpowers/` and `.claude/`. One session log predates it and is still tracked (`.claude/logs/session-20260518-c3cd1066.md`, 34 lines); I read it and it contains no credentials or tokens.
- The `.superpowers/brainstorm/` HTML mockups are untracked and contain no secrets.

---

## A4. Data — where it lives, and whether it is traceable

### A4.1 — No provenance in the schema — **High**

**Where the data lives.** There is no GeoJSON, no CSV, no external file. All map data is the `SEED_STORIES` array literal at `index.html:1027-1279`, plus whatever a given browser has in `localStorage` under `uskudar-stories` (`:1595`, `:2356`). The runtime source of truth is localStorage once seeded (`:1591-1631`), with `SEED_VERSION = 2` (`:1026`) driving migration.

**The full schema**, assembled from the seed objects and `saveEditedStory` (`:2595-2601`):
`id, person, title, lat, lng, emotions[], narrative, videoUrl?, locationKnown?, period, isGhost`

**What is missing:** interview date. Interviewer. Transcript or recording identifier. Consent record. Language the testimony was given in. Who assigned the emotion tags — the narrator or the researcher. How the coordinate was determined — pointed at on a map by the narrator, geocoded from an address, or inferred. Whether the narrator reviewed the transcription.

For an ethnographic HCI thesis this is the load-bearing gap. A committee member looking at a purple dot at 41.0262, 29.0253 has no way to ask "where did this come from, and who decided it was nostalgia?" The emotion taxonomy in particular is doing heavy interpretive work — six categories, assigned by someone, presented as a property of the testimony — and the interface presents that assignment as fact.

**Fix.** Add a `source` block per story and move the array to `data/stories.json` (per A1):

```json
{
  "id": 18,
  "source": {
    "interview_id": "USK-2026-04",
    "date": "2026-04-12",
    "language": "tr",
    "consent": "written, on file",
    "location_method": "narrator pointed on printed map",
    "emotions_assigned_by": "researcher, from transcript",
    "transcript_ref": "transcripts/USK-2026-04.md#L120-L148"
  }
}
```

Then surface a subset of it in the popup — even one line ("Anlatıcı 4 ile 12 Nisan 2026 · duygular araştırmacı tarafından atandı") converts the interface from *asserting* to *citing*. That single change does more for the thesis argument than any visual move in Part B.

### A4.2 — 17 of 18 narratives are lorem ipsum — **High**

**Evidence:** `:1040`, `:1053`, `:1066`, `:1079`, `:1093`, `:1108`, `:1121`, `:1134`, `:1147`, `:1162`, `:1175`, `:1188`, `:1201`, `:1216`, `:1229`, `:1243`, `:1257` — all Latin filler. Only story 18 (`:1272-1277`) carries real testimony, and it is genuinely strong: the 1942 Varlık Vergisi, an Armenian blacksmith in Üsküdar, workplace confiscated, exiled to a labour camp, with a linked recording. The titles *are* real and trilingual, including Western Armenian (`:1033-1035`, `:1222-1224` "Ներսէսեան Վարժարան").

I take this as a known state rather than an oversight — the specs say copy is placeholder pending real content, and the co-design workshop palette is also pending (`:1282-1283`). But it needs saying plainly in an audit: **the critical-design argument currently rests on one testimony.** Everything in Part B is a proposal about a container that is 94% empty. Fill it before or alongside any redesign; a bold visual system wrapped around lorem ipsum reads as styling, which is the specific failure mode the brief warns about.

### A4.3 — False precision on an explicitly unknown location — **Low, but conceptually sharp**

**Evidence:** story 18 carries `locationKnown: false` (`:1269`) and coordinates `lat: 41.03390715820939, lng: 29.031161149808515` (`:1267-1268`). Fourteen decimal places is sub-micron precision. The other 17 stories use 4–5 decimals (~1–10 m), which is honest.

These numbers came from the draggable-marker feature in the 12 May version (`updateCoordinates`, present in `12 May/index.html`), which dumped raw `lngLat` floats. The artifact is harmless at runtime — the marker drifts ±0.0008° (~89 m) anyway (`:2224`) — but it is exactly the false precision the thesis critiques, reproduced in the thesis's own data file. Round it to 3 decimals and let the drift carry the uncertainty.

### A4.4 — Edits are per-device and unrecoverable — **Medium**

`saveStories()` writes to `localStorage` only (`:2355-2357`). A testimony added during fieldwork on a phone exists in that phone's browser storage and nowhere else — not in git, not in a file, not exportable through any UI. Clearing site data destroys it. There is no export button anywhere in the admin modal (`:910-1008`).

**Fix:** one button in the management modal that does `JSON.stringify(stories, null, 2)` into a `Blob` download. Twelve lines. It converts the admin panel from a demo into something that can actually capture field data, and it is the missing link between "someone typed a story into the prototype" and "the story is in `data/stories.json` in git."

---

## A5. Performance

### A5.1 — Render-blocking 963 KB library; "works offline" is false — **Medium**

**Evidence:** `<link href="mapbox-gl.css">` at `:9` (34.8 KB, render-blocking in `<head>`) and `<script src="mapbox-gl.js"></script>` at `:1011` (962.9 KB, no `defer`, no `async`, no `type="module"`). The inline application script at `:1013` depends on `mapboxgl` being defined at parse time (`:1018`), so the blocking is load-bearing as written.

The comments at `:8` and `:1010` both read "vendored v2.15.0 — **works offline**". This is not true. I confirmed from the fetched style JSON that the runtime still requires `api.mapbox.com` for: the style document itself, all vector tiles (`sources: ['composite']`), and all label glyphs (`"glyphs": "mapbox://fonts/mapbox/{fontstack}/{range}.pbf"`), plus sprites. With no network, the library parses and `map.on('load')` never fires — i.e. A2.2's blank page. The comment is worth correcting because it is currently reassuring the author about an archival property the artifact does not have.

**Rough load budget on a 3G-ish connection (~400 KB/s):** 963 KB JS + 35 KB CSS + 34 KB woff2 ≈ 1.03 MB before a single tile is requested — roughly 2.5–3 s of blocking transfer, then the style round-trip, then tiles. The intro overlay's `backdrop-filter: blur(6px)` (`:717-718`) is compositing over a WebGL canvas that is still loading tiles, which on mid-range Android is the worst moment to ask for a blur.

**Fix, cheapest first:** add `defer` to `:1011` and wrap the inline script in `DOMContentLoaded` (or make it `type="module"`, which defers by default). Precompress: ship `mapbox-gl.js.gz`/`.br` if the host supports it — this library compresses to roughly a quarter of its size and costs nothing. Drop the `backdrop-filter` on small viewports.

### A5.2 — Layer and source count is minimal; markers are the cost — **Low (informational)**

The app adds **zero** Mapbox sources and **zero** Mapbox layers — `grep -c addControl` = 0, no `addSource`, no `addLayer`. Everything is DOM markers (`mapboxgl.Marker` with a custom element, `:2208-2211`). The basemap style contributes 50 layers, 14 of them symbol layers, all from Mapbox's stock Light template.

Marker cost: 18 markers × 70 particles (`:1914`) = 1,260 particle objects, each with its own `<canvas>` and 2D context. Ghost markers run `requestAnimationFrame` permanently by design (`:2017` — `(ghost && !REDUCED_MOTION) || hovering || anyMoving`), so 3 seed ghosts + 1 legend ghost = 4 always-on rAF loops before any interaction. That is acceptable at 18 stories and becomes a problem somewhere around 60–80.

The `no NavigationControl` finding is worth flagging as a **deliberate good decision** rather than an omission — see Part B.

### A5.3 — Mobile layout is broken, not merely unoptimized — **Medium**

- **Zero responsive breakpoints.** `grep -c '@media'` = 2, and both are `prefers-reduced-motion` (`:702`, `:783`). There is no viewport-width media query in the file.
- `#video-panel { width: 480px; right: 32px; }` with no `max-width` (`:404-415`). On a 390px iPhone the panel's left edge lands at −122px — it renders partly off-screen. This is the demo surface for the one real testimony that has video.
- `.controls { width: 288px; top: 20px; left: 20px; }` (`:52-60`), and the sidebar is **open by default** (`:827`, changed deliberately per the 2026-06-05 spec §3). On a 375px viewport that is 77% of the width covering the map at load.
- `.modal-box { width: 440px; max-width: calc(100vw - 40px); }` (`:462-471`) — this one is correct, and shows the pattern was known.
- Inputs at `font-size: 12px` (`:137`) and `13px` (`:495`) trigger iOS Safari's zoom-on-focus, which yanks the viewport every time the search box is tapped. The threshold is 16px.

**Fix:** one `@media (max-width: 640px)` block: video panel to `left: 12px; right: 12px; width: auto;`; sidebar to `width: auto; left: 12px; right: 12px;` and collapsed-by-default on small screens only; inputs to `font-size: 16px`. Perhaps 25 lines. Given that fieldwork demos happen on phones, this is higher practical priority than its severity suggests.

---

## A6. Accessibility

### A6.1 — Markers *are* keyboard-reachable, but only by accident, and they are unlabelled — **Medium**

I need to correct an assumption I started with. I searched the vendored library and found that `Marker.setPopup()` does this (`mapbox-gl.js`, offset ~916325):

```js
this._element.setAttribute("role","button");
this._originalTabIndex = this._element.getAttribute("tabindex");
this._originalTabIndex || this._element.setAttribute("tabindex","0");
this._element.addEventListener("keypress", this._onKeyPress);
this._element.setAttribute("aria-expanded","false");
```

Since `addMarkerForStory` calls `.setPopup(popup)` (`:2210`), **every marker is focusable and Enter/Space opens its popup.** That works today. Credit belongs to Mapbox, not to the code under audit, but it works.

What does not work: the same file sets `aria-label` to the string `"Map marker"` for every marker that doesn't already have one. The code never sets one (`grep -c 'aria-'` in `index.html` = 5, none on markers). So a screen-reader user tabbing the map hears **"Map marker, button"** eighteen times — in English, regardless of the selected language — with no way to tell testimonies apart before opening each one. Tab order is DOM insertion order (`stories.forEach(addMarkerForStory)`, `:2252`), i.e. story-id order, which has no relation to geography.

**Fix:** in `addMarkerForStory`, after creating `el`:
```js
el.setAttribute('aria-label', `${title} — ${displayPersonName(story.person)}`);
```
using the same language resolution as `getPopupHTML` (`:2166-2167`), and re-set it in `applyLang` alongside the popup refresh at `:1569-1577`. Roughly six lines, and it converts eighteen identical "Map marker"s into eighteen readable testimonies.

### A6.2 — No non-map fallback — **High**

This is the most serious accessibility finding and the one most entangled with the thesis argument, so I want to state it plainly.

**Evidence:** there is no `<noscript>` (`grep -c noscript` = 0). There is no list, table, or text rendering of the testimonies anywhere in the visitor-facing UI. The sidebar (`:827-891`) contains only filters, a count, and a three-item marker key. The one list view (`:927-939`, `renderStoryList` at `:2491-2521`) is inside the admin modal, reachable only via `Ctrl+Shift+A` (`:2779`) followed by a password.

**Consequence:** if you cannot use a WebGL canvas — because you use a screen reader, because you are on a device or browser without WebGL, because the token got revoked (A3.1), because the style 404s, or because you are on conference wifi — the content of this project is not available to you in any form. A map-only interface excludes people. Say it plainly: **a project whose argument is about who gets erased from the record should not have an interface that erases readers who cannot see it.**

**Fix:** a list. Not a hidden "accessibility version" — a visible, permanent register of every testimony, rendered from `stories` **outside** the `map.on('load')` closure so it survives map failure. This is finding A2.2's fix, this finding's fix, and the core of Part B's recommended direction, all at once. It is the highest-leverage single change available in this codebase.

### A6.3 — Contrast fails throughout, including as a direct consequence of a good decision — **Medium**

I computed all of these with the WCAG 2.x relative-luminance formula.

**Secondary text on white** (`.controls` background is `#fff`, `:56`):

| Colour | Ratio | Where | Size |
|---|---|---|---|
| `#bbb` | **1.92:1** | `.no-results` `:353`, `.mgmt-story-info span` `:651`, `.mgmt-count` `:655`, `.search-input::placeholder` `:147`, `.bulk-add-card-btn` `:665` | 10–11px |
| `#aaa` | **2.32:1** | `.controls .subtitle` `:106`, `#sidebar-close` `:118`, `.toggle-all-link` `:199`, `.stats-detail` `:346`, `.marker-key-item` `:683`, `.mgmt-tab` `:638` | 11px |
| `#888` | **3.54:1** | `.filter-section h3` `:194`, `.modal-field-label` `:484` | 11px |
| `#555` | 7.46:1 | `.mapboxgl-popup-content p` `:379` | 12px — passes |

AA requires 4.5:1 for text under 18.66px. The 11px `#aaa` labels — which include the entire marker-state key, the thing the 2026-06-05 spec designated as the standing classification key — sit at half the required ratio.

**Emotion chips — white text on the fills** (`.emotion-chip.active { color: #fff }`, `:299-303`, at `font-size: 10.5px`, `:281`):

| Emotion | Hex | vs `#fff` text | vs basemap `#f5f1ea` |
|---|---|---|---|
| blue (Üzüntü) | `#288DD4` | **3.59:1** | 3.29:1 |
| purple (Nostalji) | `#B464AE` | **3.91:1** | 3.58:1 |
| orange (Mutluluk) | `#BE7100` | **3.78:1** | 3.47:1 |
| green (Aidiyet) | `#399D57` | **3.42:1** | 3.14:1 |
| red (Öfke) | `#CD6057` | **3.88:1** | 3.56:1 |
| black (Korku) | `#8675D4` | **3.81:1** | 3.50:1 |

**All six fail, and they fail by almost exactly the same margin.** That is not a coincidence — it is the mathematical consequence of the decision documented at `:1282-1286`: pin every hue to OKLCH `L = 0.62, C = 0.14` so that "no emotion visually outweighs another." The reasoning is excellent and explicitly political (the comment notes the old palette let near-black "fear" dominate every cluster — an editorial weighting nobody decided). The lightness that makes them equal is simply the wrong lightness for white text.

**Fix that keeps the politics.** Do not un-equalise the palette; that would trade an argued design decision for a lint rule. Instead **take the text off the fill**: render the chip as an outlined pill with `#1a1a1a` text (16.1:1) and the emotion colour as a 8px swatch or a 2px left rule. The active state becomes a filled swatch plus a heavier border rather than a colour flood. Equal weighting is preserved — arguably better preserved, since the six chips now differ only in a small colour token and not in a large field of colour. Alternatively, drop to `L ≈ 0.45` uniformly, which keeps equality and clears 4.5:1 against white; but the outlined version is better because it also fixes the marker-vs-basemap ratios in the third column, which no text-colour change can.

**Ghost markers, separately:** desaturating 85% toward grey (`:1894-1904`) turns purple `#B464AE` into `#8c808b` — 3.45:1 against the basemap at full opacity, and the ghost renders at alpha `0.08–0.45` (`:1990`), so effective contrast bottoms out near 1.1:1. That is deliberate and it is the whole point of a ghost. It is defensible **only if the same information exists somewhere non-visual** — which today it does not (A6.2). The register fixes this too.

### A6.4 — `<html lang>` never updates — **Medium**

**Evidence:** `<html lang="tr">` at `:2`. `applyLang` (`:1528-1582`) sets `document.title` (`:1532`), walks `[data-i18n]` (`:1535`), `[data-i18n-placeholder]` (`:1541`), `[data-i18n-title]` (`:1548`), updates the pill (`:1556`), rebuilds filters and popups (`:1561-1577`) — but never touches `document.documentElement.lang`.

**Consequence:** a screen reader announces the English and Western Armenian interfaces using Turkish pronunciation rules. For Armenian this is not a minor mispronunciation; it renders the text close to unintelligible. Given that the co-presence of the three scripts is described in the code itself as "part of the content" (`:589-591`), shipping Armenian that a screen reader cannot pronounce undercuts the point.

**Fix:** one line in `applyLang`: `document.documentElement.lang = lang === 'hy' ? 'hyw' : lang;` (`hyw` is the IETF subtag for Western Armenian specifically, which is the correct tag here and matters for a language whose Eastern variant differs). Consider also marking the Armenian story titles inside popups with `lang="hyw"` when the surrounding UI is `tr` or `en`.

### A6.5 — Reduced motion is handled well — **credit, not a finding**

Unusually thorough. `REDUCED_MOTION` (`:1893`) is checked in five distinct places, each with a reasoned comment: ghost alpha freezes at a static mid-pulse 0.30 (`:1988-1990`), the haunting scheduler is skipped entirely and ghosts stay permanently visible (`:2087-2091`), drift is disabled with uncertainty still carried by the wider particle cluster (`:2218-2220`), the legend's drift keyframe is disabled in CSS (`:702-704`), and the intro fade is skipped (`:783-787`, `:2810-2811`). The distinction drawn at `:1890-1892` — autonomous motion is suppressed, but input-driven motion following the user's own gesture is kept — is the correct reading of the spec and better than most production sites manage.

### A6.6 — Focus management and phantom targets — **Low**

- No focus trap and no focus return on either modal (`:894`, `:910`) or the intro overlay (`:801`). `openAdminLogin` focuses the username field on a 50ms timer (`:2403`); nothing else manages focus. Tab from an open modal walks into the page behind it.
- The intro overlay toggles `aria-hidden` correctly (`:2795`, `:2807`) but never moves focus into itself, so a screen-reader user starts reading the sidebar behind a modal blocking layer.
- **Phantom targets:** ghost markers sit at `opacity: 0` for 5–18 seconds at a time (`:2079-2084`) while remaining focusable (per A6.1) and hit-testable. A sighted mouse user cannot see or reliably click a target that a keyboard user can tab straight into; conversely a low-vision user may never perceive a ghost at all. The mechanic is the best idea in the prototype — see Part B — but its current implementation is only defensible if the ghosts are permanently listed somewhere non-visual.
- Two `<h1>`s (`:803`, `:831`) and no landmark regions (`<nav>`, `<main>`, `<aside>`); the sidebar is a bare `<div class="controls">`.

---

## A7. Reproducibility and archival

**Verdict: no, this will not run in five years as-is.** But the gap is smaller than it looks, and the instincts are right.

**What is already correct and should be credited:**
- `mapbox-gl.js` and `mapbox-gl.css` are vendored at v2.15.0 (`:9`, `:1011`) instead of CDN-linked. The `12 May` version used `https://cdn.jsdelivr.net/npm/mapbox-gl@2.15.0/...` (`12 May/index.html:547`) and moving off it was a deliberate improvement.
- `NotoSerifArmenian-wght400-700.woff2` is self-hosted (`:22-29`, 33.8 KB), with `font-display: swap` and a `unicode-range` scoping it to Armenian glyphs. This is careful work.
- No build step, no `node_modules`, no framework. A single HTML file and three assets is genuinely the most archivable web architecture available.

**What breaks it:**

| Dependency | Where | Failure mode |
|---|---|---|
| Hosted style `mapbox://styles/rudipyan/cmr9mbchq000m01qv1mf9gbmm` | `:1656` | Style JSON is fetched at runtime. I confirmed it is marked `"visibility": "private"` and last modified 2026-07-06. **It dies with the Mapbox account.** |
| Vector tiles (`sources: ['composite']`) | style JSON | All map geometry comes from `api.mapbox.com`. |
| Glyphs `mapbox://fonts/mapbox/{fontstack}/{range}.pbf` | style JSON | All basemap labels come from Mapbox. |
| Access token | `:1018` | Will be rotated (correctly — A3.1). A rotated token in an archived copy = blank map. |
| YouTube embed | `:1277` | The one real testimony's recording is a third-party embed that can be removed, age-gated, or region-blocked. |
| **No README** | — | `ls | grep -i readme` → nothing. No run instructions, no license, no data dictionary. |

**Documentation drift:** the 2026-06-05 plan states "**No git:** this directory is not a git repository." It is now, and has been for 55 commits. The 2026-05-18 plan's file map cites line numbers ("CSS 1–630", "JS config block 759–969") that no longer correspond to anything. The specs remain accurate about *intent*; the plans have decayed into archaeology. That is normal and fine — but a reader five years out will trust them.

**Fix, in order of value per hour:**

1. **Write a README.** Ten lines: what this is, `python3 -m http.server 8000` then open `http://localhost:8000` (and why `file://` fails), where the data lives, where the token comes from, what the three marker states mean, and a one-paragraph note that the dated folders are frozen iterations. This is thirty minutes and it is the difference between an artifact and a folder.
2. **Download the style locally.** `curl "https://api.mapbox.com/styles/v1/rudipyan/cmr9…?access_token=$TOK" > style/map.json`, then `style: 'style/map.json'` at `:1656`. Removes the account dependency for the style document. Tiles and glyphs still come from Mapbox, so this is partial — but it is a ten-minute change that also becomes the editable substrate for every Part B direction, since you cannot restructure a basemap you don't have a local copy of.
3. **For real five-year archival: self-host the tiles.** An OSM-derived PMTiles extract of the Üsküdar bbox at z0–z16 is roughly 5–20 MB. Combined with a self-hosted glyph set and `protomaps` or `maplibre-gl` in place of `mapbox-gl`, the folder becomes genuinely self-contained: no token, no account, no network, no expiry. At that point the "works offline" comments at `:8` and `:1010` become true. This is perhaps two days of work, and it is the same work that Direction A in Part B requires anyway — which is the argument for doing them together.
4. **Archive the video.** Download the recording, commit it or deposit it in the university repository, and keep the YouTube link as a convenience rather than the only copy.
5. **Add a LICENSE.** For a public repo containing oral-history testimony, the license question is not boilerplate — code and testimony almost certainly want different terms, and the narrators' consent almost certainly constrains reuse. A `LICENSE` for the code plus a `DATA-LICENSE.md` stating the consent terms is the honest structure.

---

# PART B — DESIGN CRITIQUE AND BOLD DIRECTIONS

## B1. The current design system, reconstructed from the code

Not from the specs — from what is actually in `index.html`.

### Typography

**Stack** (`:37`): `'Noto Serif Armenian', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`.

The consequential detail is at `:28`: the `@font-face` carries `unicode-range: U+0308, U+0530-058F, U+2010, U+2024, U+25CC, U+FB13-FB17` — Armenian block only. So **Armenian renders in a designed serif book face; Turkish and English render in the operating system's UI sans.** The comment at `:18-21` frames this as the two scripts coexisting rather than Armenian falling back to an afterthought, which is right as far as it goes. But the actual result is asymmetric in an undeclared way: Armenian is the only script in this project that has been given a typeface. Turkish and English wear San Francisco or Segoe UI — the chrome of the device.

That asymmetry can be read two ways, and the project has not decided which: either Armenian is honoured while the colonial languages are demoted to system furniture (strong, and worth making explicit), or the Armenian is content while Turkish and English are interface (weak, and accidental). Right now it reads as the second, because the sizes and weights are identical across all three.

**Scale.** Thirteen distinct sizes between 9px and 40px, with no ratio: 40 (`:727`), 20 (`:329`), 18 (`:436`), 15 (`:474`, `:733`), 14 (`:96`, `:369`, `:771`), 13 (`:61`, `:495`, `:522`, `:747`), 12.5 (`:551`), 12 (`:135`, `:377`, `:425`, `:583`), 11.5, 11 (`:105`, `:190`, `:324`, `:384`, `:480`, `:682`), 10.5 (`:227`, `:281`), 10 (`:337`, `:653`), 9 (`:660`, `:662`).

The centre of gravity is 10–11px. That is below the practical floor for sustained reading and it is where nearly all the interpretive labels live — the emotion names, the people chips, the marker-state key, the stats. **The classification system that the 2026-06-05 spec designated as the interface's standing key is set at 10.5–11px in `#aaa`.** The project's own taxonomy is rendered as fine print.

**Weights:** 400 / 500 / 600 / 700, applied without a rule — 700 appears on 9px bulk-card numbers (`:662`) and on 11px lang-switcher glyphs (`:608`).

### Colour roles

**Greys — thirteen, all literal hex, no tokens:** `#1a1a1a` (`:98`), `#333` (`:39`), `#444`, `#555` (`:379`), `#666` (`:229`), `#888` (`:194`), `#aaa` (`:106`), `#bbb` (`:353`), `#ccc` (`:235`), `#ddd` (`:67`), `#e5e5e5` (`:225`), `#eee`, `#f0f0f0` (`:314`), `#f5f5f5` (`:38`), `#fafafa` (`:139`), `#fff`.

**Accent: none.** The primary action colour is `#1a1a1a` (`:518`, `:774`) — near-black. Every button, every active chip, the Enter CTA. The interface has no colour of its own; the only colour in the system belongs to the emotions.

That is, quietly, the single best colour decision in the project. The chrome refuses to compete with the data. It should be protected in any redesign.

**Emotion palette** (`:1287-1294`), six hues pinned to OKLCH `L = 0.62, C = 0.14`:

`#288DD4` blue/Üzüntü · `#B464AE` purple/Nostalji · `#BE7100` orange/Mutluluk · `#399D57` green/Aidiyet · `#CD6057` red/Öfke · `#8675D4` "black"/Korku

The comment at `:1282-1286` is the most articulate design reasoning in the file: the previous ad-hoc palette let near-black "fear" dominate every cluster and saturated orange glow, "an editorial weighting nobody decided." Equalising lightness and chroma is a *political* correction, not an aesthetic one. It is the clearest evidence in the codebase that this is critical design and not a map app. (Its accessibility cost is A6.3; the fix there preserves the politics.)

**Off-system colour:** `#e74c3c` for errors (`:506`, `:654`). Flat-UI-2013 red. It is the one colour in the file that comes from neither the greyscale nor the OKLCH ramp.

**Basemap** (from the fetched style JSON): background `#f5f1ea`, landuse/national-park `#efeade`, water `#dfe4e6`, buildings `#ece5d8` from z15, POI label text `hsl(220,1%,62%)`, settlement-subdivision labels `#8a8274`. Warm paper. Chosen deliberately in commit `5830610` ("swap basemap to custom warm-paper Mapbox style"), replacing the stock `mapbox://styles/mapbox/light-v11` still visible at `12 May/index.html:777`.

### Spacing, radii, elevation

No scale. Literal values in use: 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 32 px.

**Ten distinct border-radii:** 4 (`:394`), 5, 6 (`:116`, `:582`), 7 (`:646`), 8 (`:136`, `:325`, `:521`), 10 (`:162`, `:363`), 12 (`:411`, `:610`), 14 (`:57`, `:464`), 20 (`:224`, `:601`), 22 (`:776`), 50% (`:252`).

Ten radii on one surface is the clearest available proof that there is no design system — only a sequence of local decisions, each reasonable, none reconciled. Same for the four shadow recipes (`:58`, `:163`, `:365`, `:413`, `:465`).

### Motion

This is where the project is genuinely distinctive, and it is worth cataloguing precisely because it is the part worth keeping.

- **UI transitions:** 0.15s on hover states (throughout); 0.28s `cubic-bezier(0.4,0,0.2,1)` on the sidebar (`:65`) — Material Design's standard easing curve, imported wholesale.
- **Particle physics** (`:1956-1981`): spring constant 0.1, friction 0.8, repulsion radius `13 × scale` px, repulsion force 1.4, 70 particles per marker, cluster radius 14px (known location) or **20px (uncertain location)** — uncertainty is encoded as spatial spread (`:1912`).
- **Ghost breath** (`:1990`): `alpha = 0.08 + 0.37 × (0.5 + 0.5 × sin(t × 0.0025 + phase))` — a ~2.5s cycle, per-particle random phase, alpha ranging 0.08–0.45.
- **Ghost haunting** (`:2065-2084`): visible 6–20s, hidden 5–18s, driven by a per-session `sessionStorage` `ghostBias` so every visit has a different overall hauntedness, plus a per-marker phase so they don't blink in unison. Initial appearance staggered 1–6s (`:2107`). Fades at 1.8s (`:2099`).
- **Catch mechanic** (`:2116-2136`): hovering a ghost freezes its disappearance schedule and materialises it — a 600ms exponential brightness flash (`:1996`) over an 800ms blend from desaturated to full colour (`:1999`). Release restores the haunting cycle after a 1.2–3.2s grace period (`:2130`).
- **Drift** (`:2224, :2239-2240`): `DRIFT_R = 0.0008°` (~89m N-S, ~67m E-W at 41°N), two-frequency Lissajous on each axis, frozen while hovered (`:2233-2237`).
- **Entrance:** 0.8s opacity fade (`:720`), `backdrop-filter: blur(6px)` (`:717`).

The motion language has one consistent idea: **things here do not hold still, and attention is what temporarily stabilises them.** That is a real thesis about memory expressed in easing curves, and it is the strongest design work in the file.

### Interaction model and entry

**Entry** (`:801-812`, `:2789-2841`): full-viewport scrim `rgba(245,245,245,0.55)` + 6px blur; 40px heading; a four-sentence paragraph; three language buttons; one `Haritaya gir →` CTA. Shown once per device (`uskudar-intro-seen`, `:2792`).

**Persistent chrome:** 288px sidebar, open by default (`:827`) — search box first (`:838-840`), then People chips, then Emotion chips, then a 20px count, then the three-item live marker key (`:876-889`).

**Map controls:** `grep -c addControl` = **0**. No zoom buttons, no compass, no scale bar, no geocoder, no geolocate. Only Mapbox's mandatory logo and attribution.

**Markers:** canvas particle clusters → click for a popup; hover to scatter; ghosts to catch.

---

## B2. Where it defaults to map-app convention — and why that undercuts the argument

A counter-map is an argument about erasure, not an information display. The test for each element below is not "is this good UX" but "does this element assert that the map is a neutral record of what is there?" Where it does, it is working against the thesis.

**First, credit what already refuses convention.** Three moves in this prototype are genuinely counter-cartographic and must survive any redesign:

1. **No zoom controls, no compass, no scale bar, no geocoder** (`addControl` count: 0). The interface withholds the instruments of cartographic mastery. This was almost certainly a decision, and it is the right one.
2. **The ghost haunting scheduler** (`:2065-2084`). A marker that is *absent for 5–18 seconds at a time* is a genuinely radical thing to put in a map interface. Most critical-design maps represent absence with a symbol for absence; this one makes the interface itself intermittently fail to show you the thing. The per-session `ghostBias` means no two visitors get the same map, which quietly denies the map's reproducibility.
3. **Drift for `locationKnown: false`** (`:2220-2249`) plus the wider particle cluster (`:1912`). Uncertainty is expressed as motion and spread rather than as an error bar. The marker refuses to hold a coordinate.

Those three are the seed of everything in B3. Now the problems.

### B2.1 — The basemap is the state's map, in English, and it is treated as ground truth

This is the central failure and it is fully evidenced. The style at `:1656` is a 50-layer derivative of Mapbox Light. Its symbol layers: `road-label-simple`, `waterway-label`, `natural-line-label`, `natural-point-label`, `water-line-label`, `water-point-label`, `poi-label` (from z6), `airport-label`, `settlement-subdivision-label` (z10), `settlement-minor-label`, `settlement-major-label`, `state-label`, `country-label`, `continent-label`. Plus `admin-0-boundary`, `admin-1-boundary`, `admin-0-boundary-disputed`.

And the text field: `["coalesce", ["get","name_en"], ["get","name"]]`.

**English first. Turkish as fallback. Armenian never — including when the interface is switched to `hy`.** A user reading the site in Western Armenian sees an Armenian sidebar, Armenian story titles, Armenian buttons, floating above a substrate that names the neighbourhood in the language of neither the community nor even the state. The one layer in the composition that presents itself as fact is the one layer with no Armenian in it.

Worse, `poi-label` renders **current commercial POIs** — today's cafés, pharmacies, shops — from zoom 6 upward, at a density that far exceeds 18 testimonies. At the default zoom 15 (`:1658`) the visitor sees a handful of memory-dots scattered across a field of contemporary business names. The composition reads: *here is Üsküdar as it is (authoritative, labelled, complete), and here are some people's feelings about it (sparse, coloured, subjective).* That is precisely the epistemological hierarchy a counter-map exists to invert.

The warm-paper recolouring (`#f5f1ea`) was a real improvement over stock Light and shows the instinct is right. But recolouring a cadastre does not stop it being a cadastre. **The basemap was styled; it was never interrogated.**

### B2.2 — Search-first promises retrieval from an archive that is mostly gone

The search input is the **second** element in the sidebar, above the taxonomy and above everything else (`:838-840`). A search field is a promise: *what you are looking for is in here; type it and I will find it.* For an archive of a community that was removed, that promise is false in a specific and important way — the most significant fact about this archive is what is not in it.

And the failure state makes it worse. `renderStoryList` returns a bare em-dash on no results (`:2506`), and `updateMarkerVisibility` just drops the count to 0 (`:1856`). A visitor who searches for a family name and gets nothing learns nothing — the interface stays silent, so the absence reads as a failed query rather than as the subject matter. `.no-results` was even styled (`:351-357`) and then never wired up (A2.7). The one place where the interface could have spoken about absence is dead code.

### B2.3 — The filter panel presents grief as a complete, closed taxonomy

Six emotions, five people, every chip `active` by default (`:1645-1646`, `:1685`, `:1749`). The opening state of the interface is **"everything is shown."** The stats block confirms it in a 20px bold numeral: `18 · Görünen Anı` (`:869-873`).

Two problems. First, six categories asserted without attribution — nothing anywhere says who decided that a testimony about the 1942 Wealth Tax is `black`/Korku and `purple`/Nostalji (this is the schema gap from A4.1 surfacing as a design problem). The interface presents an interpretive act as a property of the data.

Second, and more important: a filter panel is a **completeness claim**. Its entire semantics is "here is the full set; narrow it." The correct default for this project is the opposite — the visitor should arrive understanding that they are looking at a residue, and that the filters subdivide a fragment, not a corpus.

### B2.4 — The popup is a business listing

`mapboxgl.Popup({ offset: 15, maxWidth: '420px', closeButton: true })` (`:2202-2206`), styled at `:360-389`: white card, 10px radius, `0 4px 20px rgba(0,0,0,0.15)` shadow, 14px semibold title, small grey subtitle, coloured tag chips, body text, a black CTA button.

This is, component for component, the Google Maps place card. Title, category chips, description, primary action. The form says *establishment, verified, currently operating.* The content at `:1273-1275` says *this man's workshop was taken from him by the 1942 Wealth Tax and he was sent to a labour camp.*

The chrome is not neutral. A testimony about dispossession delivered in the visual grammar of a restaurant listing is domesticated by that grammar. And the `closeButton: true` is a small tell: the interface offers a tidy dismissal of an account that does not resolve.

### B2.5 — Zooming in makes memory more solid

`updateMarkerScales` (`:2259-2287`) grows markers by `1.6^(z - 16)` above zoom 16 (`:2256-2257`). At z20 a marker is ~6.5× its base size — a large, dense, confident cluster of colour.

The convention being borrowed is *zoom reveals detail*, which is the founding promise of slippy maps. But this project's subject is a place where the closer you get, the less there is. Approaching an erased building should not yield more marker; it should yield the discovery that there is nothing to resolve. As built, the interaction rewards zooming with visual mass, which is an argument that memory becomes more substantial under scrutiny. The project believes the opposite.

### B2.6 — The entrance explains the interface instead of establishing the stakes

The intro (`:803-810`, copy at `:1353`) does three things: names the project, describes the research, and **teaches the controls** — "Click on the markers to examine the testimonies, and use the filters to search by person and emotion" (`:1420`).

The last clause is an onboarding tooltip. It frames what follows as a tool to be operated. Meanwhile the genuinely disorienting facts — that markers vanish and return, that one location is unknown and drifts, that the people are numbered because they are anonymous, that most of what was here is not represented at all — arrive either as 11px `#aaa` fine print (`:876-889`) or not at all.

The `once ever per device` gate (`:2792`) compounds this: the single moment where the project can state its argument is spent on instructions, and then never returns.

### B2.7 — The interface never admits its own gaps

Nowhere does the artifact acknowledge that 17 of its 18 narratives are placeholder (A4.2), that the emotion palette is explicitly interim pending a co-design workshop (`:1282-1283`), that the third marker state cannot be authored (A2.5), that edits are device-local and unrecoverable (A4.4), or that the `hy` translations of the story content are partial. All of that is known — it is *documented in the source comments*, which are more honest than the interface they produce.

For a project about disappearance, the gap between what the code admits and what the interface admits is itself the most interesting unexploited material in the repository.

---

## B3. Three bold directions

Each is a different structural answer to B2. They are not variations on a theme and you should not merge all three.

---

### DIRECTION A — «Silinmiş» / *Erased*

> **The official map cannot coexist with the testimony: wherever someone remembers, the state's cartography is eaten away, and it grows back the moment you look elsewhere.**

**Basemap approach.** Download the style (per A7 fix #2) to `style/map.json` and strip it from 50 layers to **eight**: `land`, `water`, `waterway`, one road casing, one road fill, `building`, coastline, and a single `settlement-subdivision-label`. Delete `poi-label`, `airport-label`, all admin boundaries, `state-label`, `country-label`, `continent-label`. On the one surviving label layer, change the text field from `["coalesce", ["get","name_en"], ["get","name"]]` to `["get","name"]` — the local name, never English.

**Palette.** Paper `#EFE9DF`. Water `#E3E7E4`. Roads as 0.5px hairlines in `#D8D0C2`. Buildings `#E7E0D2` with no outline. Ink `#1B1A17`. The six emotion hues retained exactly as they are — the OKLCH equalisation is not up for renegotiation — but rendered per A6.3's fix (colour as swatch/rule, never as a field behind white text).

**Type.** Keep `Noto Serif Armenian` (already vendored, already correct) and **give the Latin script a real face too**, so the asymmetry noted in B1 becomes a decision rather than an accident: **Newsreader** (Production Type, SIL Open Font License, self-hostable as woff2) at 400/500 for testimony, paired at 17px/1.75. UI labels in **Inter Tight** 500 at 13px — up from the current 10.5–11px, which is the single cheapest legibility win available. If budget exists, **Arek** (Khajag Apelian, TypeTogether) is the Armenian face actually designed for this and carries a matched Latin; it would let one family carry all three scripts, which is a stronger statement than two families.

**Signature interaction — the erasure mask.** Precisely enough to build:

1. On load, generate a GeoJSON `FeatureCollection` of one `Polygon` per story — a 32-vertex circle centred on `[lng, lat]`, radius stored as a feature property `r` initialised to `0`. Add it as a source `id: 'erasure'`.
2. `map.addLayer({ id:'erasure-mask', type:'fill', source:'erasure', paint:{ 'fill-color':'#EFE9DF', 'fill-opacity':0.94 }})` with **no `beforeId`** — so it sits above every basemap layer but below the DOM markers (which are not layers and always render on top).
3. Opening a testimony eases that feature's radius `0 → 180 m` over 1200 ms (`easeOutCubic`), rewriting the polygon each frame via `source.setData()`. Roads, buildings and the one label beneath it are consumed by paper.
4. Closing eases it back `180 → 0` over **4000 ms** — deliberately four times slower. The state's map returns more slowly than it left, but it always returns.
5. Ghosts get a permanent baseline radius of 40 m that never fully closes. Where a place is gone, the official map is permanently thinned.

**Motion language.** One-directional, no springs, no bounce. Erasure is not playful. Every ease is `cubic-bezier(0.22, 1, 0.36, 1)` out-only, durations 1200 ms opening / 4000 ms closing. The existing particle physics stay untouched inside the markers — the contrast between soft, springy memory and slow, irreversible ground is the composition.

**Why it advances the thesis.** It stages the incompatibility directly: the interface can render the state's record or the community's, never both in the same square metre. The visitor's attention is the agent of erasure — and the asymmetric return timing means the official map is structurally advantaged, which is the historical claim. It also answers B2.1 at the root rather than by restyling.

**Build cost.** 3–5 days. Circle-polygon generation is ~20 lines of trigonometry (no turf dependency needed); one source, one layer, one rAF loop; the style strip is a JSON edit. The prerequisite style download is A7 fix #2, which you want regardless.

**Accessibility risk. Low–moderate.** The mask removes *basemap* information only; no testimony content is ever obscured, and markers render above it. Risks: (a) the animation is motion — must be disabled under `prefers-reduced-motion`, where the mask should snap to its final state instead; (b) reduced basemap contrast could disorient users relying on landmarks — mitigate with an opt-out control. Defensible.

---

### DIRECTION B — «Yoklama» / *Roll Call*

> **The register is the artifact and the map is a subordinate witness: you read the testimonies in sequence, and the map's job is only to admit whether it can locate each one.**

**Structure.** Invert the hierarchy the project currently assumes. Two columns: **left 62% is a scrolling register** of every testimony; **right 38% is a fixed map strip**. On mobile the register is the whole screen and the map becomes a 180px sticky band at the top. The map is never the primary surface, at any breakpoint.

**Basemap approach.** The same eight-layer strip as Direction A, plus a further reduction: **zero label layers**, and the whole style flattened to four values — `#F2EFE9` land, `#E5E0D6` landuse, `#D3CCBE` roads/buildings, `#B9B0A0` water edge. The map becomes texture, not reference. It cannot be read as authority because there is nothing on it to read.

**Palette.** Paper `#F7F4EE`, ink `#1B1A17` (16.3:1), secondary ink `#5A554C` (7.1:1 — replacing every `#aaa`/`#bbb` in the current file). Six emotion hues retained, used **only** as a 3px left rule on each register entry and as the marker fill. No coloured fills behind text anywhere, which resolves A6.3 by construction.

**Type.** `Noto Serif Armenian` 400 at **19px / 1.75** for testimony — this is the change that matters most, since the project currently sets its content at 12px and its taxonomy at 10.5px. Entries separated by 96px of vertical space. Register metadata (narrator, date, provenance) in **Inter Tight** 500 at 13px, letter-spacing 0.01em. Entry titles at 24px.

**Signature interaction — scroll-linked location, and its failure.** Precisely:

1. An `IntersectionObserver` with `rootMargin: '-45% 0px -45% 0px'` fires when a register entry crosses the viewport centre.
2. For a located testimony: `map.easeTo({ center:[lng,lat], zoom:16.5, duration:900, easing: t => 1-Math.pow(1-t,3) })`, and that entry's marker fades from 0.35 to 1.0 opacity.
3. **For `locationKnown: false`:** the map does *not* move to a point. It eases out to the neighbourhood bounding box, every marker drops to 0.15 opacity, and a line appears over the strip in the active language — `bu noktada durulamıyor` / *this cannot be stood on* / `հոս կարելի չէ կանգնիլ`. The map is made to state its failure, in the first person, as a first-class result rather than a footnote.
4. **For a ghost:** the map arrives at the coordinate and the marker is *not drawn* — instead the erasure treatment from Direction A applies at a fixed 60 m radius. You arrive and there is a hole.
5. Scrolling is the only navigation. No search box above the register — a filter row sits *below* the fold, after the visitor has read at least one testimony.

**Motion language.** Entirely scroll-driven. Nothing animates on its own except the ghosts' existing breath (`:1990`), which is retained. The interface has no autonomous life; it responds only to reading.

**Why it advances the thesis.** It refuses the premise that a map is the right container for testimony, while still using a map — which is a sharper critical position than either abandoning cartography or accepting it. Western Armenian gets typographic primacy at a readable size for the first time. The map's inability to locate becomes a designed, spoken state rather than a drifting dot most visitors will never notice. And it fixes A6.2 structurally: the non-map representation is not an accommodation bolted on for screen-reader users, it is **the default experience for everyone**, which is the only version of accessibility that is not condescending.

**Build cost.** 5–8 days — the highest of the three. Real layout work, an IntersectionObserver controller, three languages across a long-form reading surface, mobile reflow. But roughly two of those days are the register itself, which is already the top recommendation from Part A (A6.2) and would have to be built anyway.

**Accessibility risk. Very low — it is the accessibility fix.** Semantic `<article>` elements in a real reading order, keyboard-native, works with the map dead (A2.2), all text above 4.5:1. The only risk is scroll-hijack feel, avoided by never intercepting scroll — only *observing* it. Nothing about this trade needs defending.

---

### DIRECTION C — «Verilmedi» / *Withheld*

> **The archive is depleted by being consumed: every testimony you open spends some of the map's remaining colour, and after enough looking there is nothing left to look at.**

**Basemap approach.** Keep the current vendored style; drive it at runtime with `map.setPaintProperty()`. A session counter `spent` drives: `background-color` interpolating `#F4F1EA → #E8E7E6` as spent rises; `poi-label` `text-opacity` `1 → 0`; road layers `line-opacity` `1 → 0.3`. The more you take, the less map remains to take it from.

**Palette.** Start at the current warm paper; end desaturated. Emotion hues start at OKLCH `L 0.62 / C 0.14` and lose 20% chroma per opening of that specific testimony, persisted in `sessionStorage` so it does not reset on pan.

**Type.** `Noto Serif Armenian` 400 at 17px/1.8 for testimony; **Inter Tight** 13px/1.45 for UI, no uppercase (the current `text-transform: uppercase` at `:192`, `:338`, `:482` is bureaucratic register and works against the material).

**Signature interaction — the fourth open.** Each testimony tracks an open count in `sessionStorage`. Opens 1–3 render normally with progressively reduced chroma. **On the fourth open the popup does not show the narrative.** It shows one line, centred, in the active language: `bu kadarı anlatıldı` / *this much was told* / `այսքանը պատմուեցաւ`. The marker drops permanently to the ghost's desaturated state for the remainder of the session. There is no way to get the text back without reloading, and reloading resets everything — which is itself the point about what archives do and do not preserve.

**Motion language.** Slow (2400 ms), one-directional, never reversible within a session.

**Why it advances the thesis.** It converts the extractive relationship between visitor/researcher and survivor community from a caption into a mechanic. Nobody who uses this interface can avoid noticing that their attention costs something. It is the most conceptually aggressive of the three.

**Build cost.** 2–3 days — the cheapest. Mostly a state counter plus `setPaintProperty` calls; no new layers, no layout work.

**Accessibility risk. HIGH — and this is the one to be honest about.** Deliberately reducing contrast over time is a direct, unambiguous WCAG conflict: a user with low vision experiences the "argument" as the interface simply becoming unusable, and cannot distinguish critique from malfunction. Withholding content on the fourth open also means a screen-reader user who re-navigates a list (a normal, necessary behaviour, not consumption) burns through the limit without ever having "consumed" anything. See B5.

---

## B4. Ranking and recommendation

**Recommended: DIRECTION B — «Yoklama» / Roll Call.** Then fold in Direction A's erasure mask, scoped to the map strip, as a second phase.

**Ranking:**

| | Direction | Argument strength | Cost | A11y | Verdict |
|---|---|---|---|---|---|
| 1 | **B — Yoklama** | Highest — attacks the form itself | 5–8 d | Fixes it | **Build this** |
| 2 | **A — Silinmiş** | High — attacks the basemap | 3–5 d | Low risk | Phase 2, inside B |
| 3 | **C — Verilmedi** | Sharpest single idea, weakest as a whole system | 2–3 d | High risk | Don't build as primary |

**Why B, tied to the thesis argument specifically.**

"Designing for Disappearance" is a critique of representational systems that decide what counts as present. The current prototype accepts the most consequential of those systems — the slippy map — and then decorates it with critique. The ghosts, the drift, the equalised palette are all excellent moves *within* a frame the project never challenges. Direction B challenges the frame: it uses a map while denying it primacy, and makes the map's failure to locate into content rather than noise. That is a harder and more defensible position than either abandoning cartography (which forfeits the argument's terrain) or restyling it (which is what has been done so far).

Three further reasons, in order of weight:

1. **It resolves the artifact's worst self-contradiction.** A map-only interface (A6.2) excludes people from a project about who gets excluded from the record. That is not an accessibility footnote; it is an argumentative failure a committee member could reasonably raise. B fixes it by making the readable register the default for everyone, which is the only fix that does not create a second-class "accessible version" — a fix that would reproduce the exact hierarchy the thesis critiques.
2. **It gives Western Armenian typographic primacy.** Today Armenian is the only script with a designed face (`:22-29`) but is rendered at 12px inside a popup card, beneath an English-labelled basemap (B2.1). B puts it at 19px/1.75 on the primary surface. The gap between those two facts is currently the loudest unintended statement the artifact makes.
3. **Roughly half the work is work you already owe.** The register is Part A's top recommendation (A6.2) and simultaneously fixes the blank-page failure (A2.2). You are not choosing between "fix the audit findings" and "do the redesign" — for this direction they are largely the same task.

**Why not A as primary.** A is the more spectacular direction and the erasure mask is the best single interaction in this document. But it leaves the fundamental structure untouched: still map-first, still no non-map fallback, still 11px `#aaa` fine print. It would produce a striking artifact with the same exclusion problem. It belongs inside B, where the map strip is already stripped and the mask has something to eat — build it in phase 2 and it costs perhaps two days on top.

**Why not C.** C has the sharpest single idea in this document and I would be sorry to see it disappear entirely. But as a whole system it fails the brief's own test. Its accessibility cost is not incidental — it is load-bearing, since the mechanic *is* the degradation. And there is a deeper problem: an interface that punishes the visitor for reading testimony makes a claim about the visitor's extractive relationship that the project has not earned with only one real testimony in the file (A4.2). Make the archive real first. **A salvageable fragment:** keep the "fourth open" sentence and drop the contrast decay entirely — a testimony that says `bu kadarı anlatıldı` after repeated visits, with no visual degradation, delivers most of the idea at none of the cost. Consider it as a small move inside B.

---

## B5. Accessibility and legibility trades — flagged explicitly

The brief asks me to name every proposed move that trades away accessibility or legibility, and to say whether the trade is defensible here. In order of severity:

**1. Direction C's progressive contrast decay — NOT DEFENSIBLE as specified.**
Deliberately driving text and marker contrast downward over a session is a direct WCAG 1.4.3 violation with no user-facing escape. A low-vision user cannot distinguish "this interface is making an argument about extraction" from "this interface is broken," and the argument therefore lands only on users who can see well enough to perceive it as intentional — an argument about erasure that is legible only to the unimpaired. That is self-refuting. **Only defensible if:** the decay is hard-capped so no text falls below 4.5:1 and no marker below 3:1; it is disabled entirely under `prefers-reduced-motion`; and a visible, persistent control on the entrance says "this map degrades as you use it — turn that off." With all three, it survives. Without them, cut it.

**2. The existing ghost haunting — currently NOT DEFENSIBLE; becomes defensible under B.**
This is a live finding, not a proposal. Ghosts sit at `opacity: 0` for 5–18 seconds (`:2079-2084`) while remaining focusable and clickable (A6.1, A6.6). A low-vision user may never perceive a ghost at all; a keyboard user tabs into targets nobody can see. It is the best idea in the prototype and it currently excludes people from the testimonies most central to the argument. **Defensible the moment the register exists**, because every ghost is then permanently listed, readable and reachable in text — the map's intermittency becomes an expressive layer over a stable substrate rather than the sole channel. Until then, at minimum set `aria-hidden` / `tabindex="-1"` on a ghost while it is at zero opacity so the visual and non-visual states agree.

**3. Direction A's erasure mask — DEFENSIBLE.**
It removes basemap information only. No testimony text is ever obscured; markers render above the mask; the register (under B) is unaffected. Conditions: snap rather than animate under `prefers-reduced-motion`, and provide an opt-out. The information destroyed is exactly the information the thesis argues should not be treated as ground truth, which is the definition of a trade that serves the argument.

**4. Direction B's removal of all basemap labels — DEFENSIBLE, with one carve-out.**
Stripping labels removes orientation cues some users rely on. But B moves the wayfinding burden onto the register, which is text, ordered, and screen-reader native — so orientation improves overall. **Carve-out:** keep one `settlement-subdivision-label` layer with `["get","name"]` so the neighbourhood remains nameable, and put the place name in each register entry as text. Nobody should have to read a map to know where they are.

**5. Direction B's removal of search from above the fold — DEFENSIBLE but watch it.**
Demoting search below the first testimony (B2.2) is deliberate friction against the retrieval promise. Fine for a visitor. **Not fine for a returning researcher or a committee member trying to find a specific testimony.** Keep search fully keyboard-reachable via a skip link and `/` shortcut even while it sits below the fold. Friction as argument must never become friction as obstruction for the people who need to audit the work.

**6. Raising type sizes — no trade at all.**
Moving UI labels from 10.5px to 13px and testimony from 12px to 17–19px costs nothing but layout time. The current sizes are not a design position; nothing in the specs or commit history argues for them. This is pure gain and should happen regardless of direction.

**7. The equal-lightness palette — a trade already made, and correctly.**
The OKLCH equalisation (`:1282-1286`) knowingly costs contrast to buy political neutrality between emotions. Defensible and well-argued. The fix in A6.3 (colour as swatch/rule, ink for text) keeps the politics and recovers the contrast, so this trade need not stand. **Do not resolve it by un-equalising the palette** — that would trade a reasoned design decision for a lint rule, which is the wrong direction of travel for this project.

---

# IF YOU DO ONLY 5 THINGS

Spanning the whole audit, in order. The first is measured in minutes; the rest are the spine of both halves.

### 1. Restrict the Mapbox token, then rotate it — and change that password elsewhere
**~1 hour. Findings A3.1, A3.2.**
In the Mapbox dashboard, add URL restrictions to the token at `index.html:1018` (deployment origin + `http://localhost:8000`), *then* issue a replacement and delete the old one. Restrict-then-rotate, in that order, so there is never a live unrestricted token. Separately: the plaintext admin password is permanently readable in public history at commit `39766cd` — change it anywhere else it has been used. Do not rewrite git history; rotation is the correct remedy. Nothing else on this list matters if the map goes dark mid-demo because a scraper found the token.

### 2. Put real testimony in the file, with provenance
**Findings A4.1, A4.2.**
17 of 18 narratives are lorem ipsum. Move `SEED_STORIES` (`:1027-1279`) to `data/stories.json`, fill it with real content, and add a `source` block per story: interview id, date, language, consent, how the coordinate was determined, and **who assigned the emotion tags**. Then surface one line of it in the popup. This is the only item on the list that the thesis argument cannot survive without — and it converts the interface from asserting to citing, which is a bigger critical-design move than anything in Part B.

### 3. Build the register — the non-map, keyboard-reachable list of every testimony
**Findings A6.2, A2.2; Direction B phase 1.**
Render every story as semantic HTML from `stories`, **outside** the `map.on('load')` closure, present by default rather than hidden behind an accessibility toggle. One change fixes three things at once: the exclusion of anyone who cannot use a WebGL canvas, the blank-page failure when Mapbox is unreachable, and the first and largest phase of the recommended design direction. Highest leverage per hour in the entire audit.

### 4. Take the basemap seriously — download it, strip it, and de-anglicise it
**Findings A7, B2.1; Directions A and B.**
`curl` the style to `style/map.json` and point `:1656` at the local copy (ten minutes, and it removes the Mapbox-account dependency for the style document). Then cut 50 layers to ~8, delete `poi-label` and the admin boundaries, and change the surviving label layer's text field from `["coalesce",["get","name_en"],["get","name"]]` to `["get","name"]`. Right now a counter-map of Armenian Üsküdar labels the neighbourhood in English underneath an Armenian interface. That is the sharpest contradiction in the artifact and it is a JSON edit away from being fixed.

### 5. Fix the type sizes and the contrast without touching the palette's politics
**Findings A6.3, A6.4, B1.**
Take white text off the emotion fills — outlined chips, `#1a1a1a` text, colour as an 8px swatch — which clears all six WCAG failures while *preserving* the equal-lightness decision at `:1282-1286`. Replace every `#aaa` (2.32:1) and `#bbb` (1.92:1) with `#5A554C` (7.1:1). Move UI labels from 10.5px to 13px and testimony from 12px to 17px. Add the one-line `document.documentElement.lang = lang === 'hy' ? 'hyw' : lang` to `applyLang` so screen readers stop pronouncing Armenian as Turkish. Perhaps a day, no design position surrendered.

**Deliberately not in the top 5, but cheap and worth doing in the same sitting:** the `@media (max-width: 640px)` block (A5.3 — the video panel currently renders off-screen on every phone, and fieldwork demos happen on phones); the drift `cancelAnimationFrame` (A2.3); `defer` on the Mapbox script tag (A5.1); a ten-line README (A7); and untracking `9 June/index.html` (A2.1).


---

# APPENDIX — Live browser verification (2026-07-25)

Added after the original audit, which was a static source read. This records what was
observed running the prototype in Chrome (`python3 -m http.server 8802`, desktop
1440×900). **Where live evidence contradicts the audit above, the appendix wins.**

## CONFIRMED

| Finding | Live evidence |
|---|---|
| Placeholder data (High) | The KİŞİLER filter renders exactly five narrators: **"Anlatıcı 1", "Anlatıcı 2", "Anlatıcı 3", "Anlatıcı 4", "Mikail"**. Four generically-numbered placeholders and one real name. The counter reads **"18 GÖRÜNEN ANI"**. Confirms 17-of-18 placeholder. |
| Trilingual entry | The entry screen offers Türkçe / English / **Հայերէն**, and the Armenian renders correctly with no tofu. |
| The map does load | Style, tiles and glyphs all fetched successfully; markers render in three visual states. The blank-page risk in A2.2 is a *failure-mode* risk, not a current defect. |

## CORRECTIONS

**C-1 — The basemap does NOT label Üsküdar in English. It labels it in Turkish.**

This is the audit's single decisive design finding and it is **wrong as stated**.

The style expression is quoted correctly:
`text-field: ["coalesce", ["get","name_en"], ["get","name"]]`. But `coalesce` returns
`name_en` *only where that property exists*. In Mapbox Streets, minor Turkish streets
carry no `name_en`, so they fall through to `name`. Rendered at neighbourhood zoom,
every visible label is Turkish:

> Bahanakkaş Sokağı · Meriç Sokağı · Kozanoğlu Sokağı · Gazi Caddesi ·
> Trablus Sokağı · Vakıf Sokağı · Secaat Sokağı · Koncagül Sokağı · Arzu Ayak…

English would surface only for features that *have* a distinct `name_en` — countries,
major cities, seas, major arterials — none of which dominate this view.

**What survives, and it is still the real finding:** the substrate renders
**Armenian never**, in any language mode, including `hy`. A counter-map of Armenian
Üsküdar is drawn on a basemap that cannot name a single Armenian place. That argument
does not need the English claim, and it is weaker for having been overstated — do not
repeat "English first, Turkish second" in the thesis, because a reader can open the
map and see Turkish.

**C-2 — `locationKnown` IS surfaced in the UI.**
A2.5 calls the flag "unreachable from the UI." The map legend in fact exposes three
marker states — **"Mevcut mekân" / "Artık mevcut değil" / "Konum belirsiz"** — and
markers in the third state render as distinct purple clusters visible on the map. The
narrower true statement is that it is a *legend key, not a filter*: you cannot filter
by it the way you can by narrator or emotion. Correct the finding to that.

This matters for Part B: Direction B («Yoklama») is described as turning an unreachable
flag into the load-bearing element. In fact the state is already visible, so the
direction is **cheaper than estimated** — it promotes an existing surfaced state rather
than exposing a hidden one.

## NEW FINDING

**N-1 — Scroll-zoom does not respond. *(Medium, unresolved)***
Eight wheel ticks over the map produced no zoom change and a pixel-identical
screenshot. Either scroll-zoom is disabled, or the camera is constrained by
`minZoom`/`maxBounds`. This was not investigated further. It is worth resolving
deliberately rather than by accident, because B2.5 ("zooming in makes memory more
solid") assumes zoom is a live axis of the argument — if zoom is already locked, part
of that critique may not apply, and a deliberate zoom lock could instead become a
design statement.
