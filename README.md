# buradayaşadık

A counter-map of Armenian memory in Üsküdar, Istanbul: a browser prototype placing testimonies about lost and surviving places on a map, and letting some of those places refuse to stay put. Counter-map here means the map is built to argue with the official record rather than reproduce it, so it withholds the usual cartographic instruments (no zoom buttons, compass or scale bar) and gives absence and uncertainty their own visual states. It is a working prototype used in conversation about the project, not a finished thesis artifact.

**Content status.** Fieldwork has not happened. 17 of the 18 seed narratives are placeholder text (lorem ipsum). Only story 18 is real testimony: the 1942 Varlık Vergisi, an Armenian blacksmith in Üsküdar whose workshop was confiscated and who was exiled to a labour camp. Titles are real in all three languages, including Western Armenian (Ներսէսեան Վարժարան). Do not read the placeholder narratives as data.

**Running it.** `python3 -m http.server 8000`, then open `http://localhost:8000`. Do not double-click `index.html`: under `file://` the admin login fails, because `crypto.subtle` needs a secure context.

**The three marker states**, as shown in the sidebar legend:
- *Mevcut mekân* / place still exists — steady marker.
- *Artık mevcut değil* / no longer exists — the ghost markers, which fade out and back on a timed cycle (visible roughly 6–20 s, hidden 5–18 s) and can be caught and held still.
- *Konum belirsiz* / location uncertain — the marker drifts and its particle cluster is wider. Uncertainty is spatial spread, not a caveat in text.

**Where the data lives.** Narratives start as the `SEED_STORIES` array in `index.html`, are copied on first load into `localStorage` under the key `uskudar-stories`, and `SEED_VERSION` drives migration when the seed changes. Anything added or edited through the admin panel lives in that one browser on that one device and is **not saved to git**; clearing site data destroys it. Use the admin panel's JSON export if an edit needs to survive.

**Earlier iterations** are git tags now, not folders. The two dated folders were byte-identical duplicates of content already in git and were removed on 2026-07-25.

- `git show iteration-12-may:index.html` — reproduces the deleted `12 May/` copy exactly. This is a genuinely different earlier version: it loaded Mapbox from a CDN and used the stock `mapbox://styles/mapbox/light-v11` basemap.
- `git show iteration-9-june:index.html` — the state of the project on 9 June 2026. Note that this is *not* what the deleted `9 June/` folder contained: despite its name that folder held a copy of the then-current July `index.html`, which is why it was byte-identical to the working file. Those exact bytes are at `git show 'pre-audit-fixes:9 June/index.html'`.

**Mapbox dependency.** `mapbox-gl.js` / `mapbox-gl.css` (v2.15.0) and the `Noto Serif Armenian` woff2 are vendored here, but the map is *not* offline-capable. Two comments in `index.html`, at the `mapbox-gl.css` link and the `mapbox-gl.js` script tag, claim "works offline"; that is false. The style document, all vector tiles, all label glyphs and the sprites are fetched from `api.mapbox.com` at runtime under a personal token. If that token is rotated without updating the source, if the account lapses, or if there is no network, the library parses but `map.on('load')` never fires and the page is blank. The map is the only route to the content, so that failure is total. Story 18's recording is a YouTube embed and can be taken down independently.

**Stale docs.** `docs/superpowers/` is kept for the record but has drifted: the 2026-06-05 plan says "No git: this directory is not a git repository" (it is one, 55+ commits), and the 2026-05-18 plan cites line numbers that no longer correspond to anything. The specs are still accurate about intent; the plans are archaeology.

**Licences.** `LICENSE` (MIT) covers the source code only. The testimony is covered separately, on different terms, by `DATA-LICENSE.md`.
