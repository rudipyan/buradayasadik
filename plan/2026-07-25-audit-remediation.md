# Audit remediation plan — buradayaşadık

**Source:** `25July_audit_dtes544.md` (2026-07-24, + live appendix 2026-07-25)
**Written:** 2026-07-25
**Artifact type:** prototype shown to people while talking about the project. Not a
thesis-committee submission. Fieldwork has not happened; placeholder content is a
known state, not a defect.

---

## Scope boundary — agreed with Rudi

### Governing logic — DO NOT TOUCH

Any change that violates one of these is out of scope, regardless of what the audit says.

1. **The map is the only route to content.** No register, no list view, no
   "accessible version", no non-map fallback that renders testimony. This kills the
   audit's Direction B and its top recommendation (A6.2 / "if you do only 5 things" #3).
2. **The OKLCH equal-lightness palette** (`index.html:1282-1293`). No emotion may
   visually outweigh another. Hues, L=0.62, C=0.14 are fixed.
3. **Ghost haunting, the catch mechanic, drift-as-uncertainty**, the wider particle
   cluster for `locationKnown: false`.
4. **No zoom buttons / compass / scale bar / geocoder.** The withholding of
   cartographic instruments is deliberate.
5. **Three scripts co-present**, Armenian with its own designed face.

### Design implementation — open to change

Type scale, greys, chip rendering, spacing/radii tokens, responsive behaviour, popup
styling, entrance, basemap styling.

### Decisions taken

| Question | Answer |
|---|---|
| Direction A / B / C | All dropped. Map stays primary. |
| Register | Dropped — violates logic #1. |
| Real testimony + provenance (A4.1/A4.2) | Deferred until after fieldwork. |
| A1 file split | `styles.css` only (justified by the CSS work below). No `data/stories.json`, no `src/admin.js`. |
| Greys and type sizes | In scope. `#aaa`/`#bbb` → `#5A554C`; 10.5px labels → 13px. |
| Emotion chips | **Build both variants, screenshot side by side, Rudi picks.** Not applied unilaterally. |
| Basemap | Download locally **and** strip 50 layers → ~8. |
| Admin login | Leave the code alone. No backend, blast radius near zero. Rudi changes the password wherever else it was used. |
| Duplicate folders | Tag the milestones, then remove. Recommendation — Rudi can veto. |

### CORRECTION — A3.1 is VOID (verified 2026-07-25)

**The audit's #1 finding is wrong. The Mapbox token IS URL-restricted.**

The audit concluded "no URL restriction" from two tests that cannot detect one:
a `GET` on the **styles** endpoint, and `GET /tokens/v2`. The styles endpoint
answers `200` regardless of restrictions, and `/tokens/v2` does not return an
`allowedUrls` field at all. I repeated both tests and reached the same wrong
conclusion before testing the endpoint that actually enforces.

Enforcement is on the **tiles** endpoint — which is also the endpoint that
consumes the account's map-load quota. Measured:

| Referer | tiles |
|---|---|
| `https://evil-scraper.example.com/` | **403** |
| `https://rudipyan.github.io/buradayasadik/` | 200 |
| `https://rudipyan.github.io/` | 200 |
| `http://localhost:8802/` | 200 |
| *(none)* | **403** |

Dashboard confirms three allowed URLs on token `…BKoqAQ` (the one in HEAD),
under the token name `buradayasadik-web`. Same token as `index.html` (sha256
of both compared, identical).

**Consequences:** no urgent rotation. The "restrict then rotate" item in
"IF YOU DO ONLY 5 THINGS" #1 is already done. Do not repeat the
unrestricted-token claim when presenting — it is checkable and false.

**Standing caveat, not a finding:** a `Referer` can be spoofed with `curl`, so
URL restriction deters casual and browser-based abuse rather than guaranteeing
anything. That is true of every Mapbox `pk.` token; it is the documented
mechanism and it is correctly applied here. A usage alert on the account is the
sensible backstop.

A3.2 (admin password in public history) is unaffected and still stands.

### Corrections to the audit, carried forward

- **C-1:** the basemap labels are **Turkish, not English**. `coalesce` falls through to
  `name` because minor Turkish streets carry no `name_en`. Do not repeat "English
  first, Turkish second" when presenting — it is checkable and wrong. The surviving,
  sharper finding: **Armenian never, in any language mode, including `hy`.**
- **C-2:** `locationKnown` **is** surfaced — it is in the legend as "Konum belirsiz"
  and renders as distinct clusters. The true narrower finding: it is a legend key, not
  a filter, and it cannot be authored from either the edit form or the bulk-add form.
- **N-1 (new):** verified there is no `scrollZoom: false`, no `maxBounds`, no
  `minZoom`/`maxZoom` in the map constructor (`:1654-1661`). The reported dead
  scroll-zoom has no cause in the code. Treat as a likely test-harness artifact;
  confirm by hand, do not "fix".

---

## Priority — ranked by what makes the demo fail in front of an audience

### Tier 0 — Rudi only, cannot be delegated (~1 hour)

| # | Task | Finding |
|---|---|---|
| 0.1 | Mapbox dashboard → add URL restrictions to the token (deploy origin + `http://localhost:8000`), **then** rotate and delete the old one. Restrict-then-rotate, in that order. | A3.1 |
| 0.2 | Change `uskudar2***` anywhere else it has been used. Permanently public at commit `39766cd`. | A3.2 |

Do not rewrite git history. Rotation is the correct remedy for a `pk.` token.

### Tier 1 — sequential, done by the lead session

| # | Task | Finding |
|---|---|---|
| 1.1 | `git tag pre-audit-fixes HEAD`; create branch `audit-remediation`. Escape hatch for everything below. | — |
| 1.2 | Extract `<style>` (`:12-787`) to `styles.css`, link it. Mechanical, verify pixel-identical before proceeding. Unblocks the CSS agent. | A1 |
| 1.3 | `git tag iteration-12-may` / `iteration-9-june`; `git rm --cached "9 June/index.html"`; remove both folders from the working tree; add to `.gitignore`. | A2.1 |

### Tier 2 — parallel agents, one file each

| Agent | Owns | Work | Findings |
|---|---|---|---|
| **A — CSS** | `styles.css` | Greys → `#5A554C`; UI labels 10.5px → 13px; hoist ~13 greys + 10 radii to `:root` tokens; `@media (max-width: 640px)` block; inputs → 16px (stops iOS zoom-on-focus); delete dead CSS. **Plus:** build the outlined-chip variant in a separate stylesheet for comparison — do not apply it. | A6.3, A5.3, A2.7, B1 |
| **B — Basemap** | `style/map.json` | `curl` the style down. Then strip: delete `poi-label`, `airport-label`, all admin boundaries, `state-label`, `country-label`, `continent-label`, and the redundant water/natural label layers. Keep land, water, waterway, one road casing + fill, building, coastline, one `settlement-subdivision-label` with `["get","name"]`. Report the one-line `:1656` change; do **not** edit `index.html`. | A7#2, B2.1 |
| **C — JS** | `index.html` JS only | Drift `cancelAnimationFrame` (A2.3); zoom-bucket widened to 0.5 + rebuild on `zoomend` (A2.4); `locationKnown` checkbox in both forms (A2.5/C-2); bulk-add writes `{[lang]: value}` objects (A2.6); guarded `map.on('error')` + visible "map could not load" message — **a message, not a register** (A2.2); marker `aria-label` from title + narrator (A6.1); `document.documentElement.lang = lang === 'hy' ? 'hyw' : lang` (A6.4); `defer` on the Mapbox tag + `DOMContentLoaded` (A5.1); JSON export button in the admin modal (A4.4); `t('videoPanelTitle')` fallback + `videoUrl` host allowlist + `crypto.randomUUID()` (A2.8). | many |
| **D — Docs** | `README.md`, `LICENSE`, `DATA-LICENSE.md` | README: what this is, `python3 -m http.server 8000` and why `file://` fails, where data lives, what the three marker states mean, note that the dated folders are now git tags. Correct the "works offline" comments at `:8`/`:1010` — they are false. LICENSE for code; DATA-LICENSE for testimony/consent terms. | A7 |

**Contention control:** after 1.2 the CSS lives in its own file, so A and C never touch
the same file. B produces a JSON file and *reports* the `:1656` edit; the lead session
applies it. D only creates new files.

### Tier 3 — lead session, after agents return

| # | Task |
|---|---|
| 3.1 | Apply B's one-line `:1656` change. |
| 3.2 | Serve on `python3 -m http.server` and verify in Chrome: desktop 1440×900 and mobile 390×844. Console clean. Markers, ghosts, particles, popups, catch mechanic, all three languages still work. |
| 3.3 | Screenshot current chips vs. outlined chips side by side → Rudi picks → apply or discard. |
| 3.4 | Confirm scroll-zoom by hand (N-1). |
| 3.5 | Conventional Commits, one commit per concern. Do not push without asking. |

---

## Explicitly not doing

- Building a register / list view / non-map fallback for content — violates logic #1.
- Splitting `data/stories.json` or `src/admin.js` — the audit justified both by design
  work that is now out of scope.
- Filling in real testimony or provenance — fieldwork has not happened.
- Rewriting git history for the leaked tokens — rotation is the correct remedy.
- Renaming the misleading `emotionData.black` key — it is persisted in localStorage and
  in every story's `emotions` array; the migration costs more than the benefit.
- Self-hosting tiles / PMTiles migration — two days, and only pays off for archival,
  which is not what this artifact is for yet.
- Un-equalising the emotion palette — violates logic #2.

---

## OPEN ISSUE — local basemap not wired in (2026-07-25)

`style/map.json` (8 layers, stripped) and `style/map.original.json` (untouched 50-layer
download) both exist. **Neither is wired into `index.html`** — the Studio-hosted style
is still the default, because I could not verify the local style reliably.

**Symptom.** With `style: 'style/map.json'`, `map.on('load')` did not fire in the
automated browser: no markers, empty people/emotion filters, story count `—`. No
console errors. `isStyleLoaded()` true, `areTilesLoaded()` true, sprite 200,
glyphs 200 for all three fontstacks.

**Why the diagnosis is not trustworthy.** A controlled trial run late in the session
reported `load` never firing for *four* styles including the Studio-hosted one that
demonstrably works in a normal window. The automated tab appears to throttle
rendering, and Mapbox fires `load` on first render. So the earlier bisect
(“any symbol layer stalls it; zero symbol layers works”) rests on single
observations taken through a measurement that is now known to be unreliable.
Do not treat it as established.

**What is actually known:**
- The hosted Studio style works. This is the committed default.
- `index.html` is byte-identical to the pre-test snapshot apart from the style line.
- Nothing about the local style files is known to be malformed: valid JSON,
  `sources`/`glyphs`/`sprite` unchanged from the original, no dangling source refs.

**Next step.** Open the prototype in a normal Chrome window, swap the style line to
`style: 'style/map.json'`, hard-reload, and see whether the 18 markers appear.
That single manual check settles it. If markers appear, the local stripped basemap
is fine and the automation was the problem. If they do not, bisect by re-adding
`settlement-subdivision-label` from `map.original.json`.

Also unverified for the same reason: the marker `aria-label` work (task 7). One probe
showed the Mapbox default `"Map marker"` still in place, but that probe may have run
before `addMarkerForStory` executed. Re-check in a normal window.

## Verification

The site must load and function after **every** tier. Checks at 3.2:

- No console errors.
- 18 markers render; three visual states present.
- Hover scatters particles; ghosts appear/disappear; catch mechanic freezes a ghost.
- Popups open, video panel opens and is fully on-screen at 390px wide.
- All three language buttons switch the UI; Armenian renders with no tofu.
- Filters and search still narrow the set.
- `git diff --stat` reviewed before any commit.
