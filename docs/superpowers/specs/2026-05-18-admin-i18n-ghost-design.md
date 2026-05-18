# Design Spec: Admin Panel Redesign, i18n, Ghost Layer
Date: 2026-05-18
Project: Burada Yaşadık — Üsküdar Oral History Map

---

## 1. Scope

Three interconnected features implemented together because they share the same data model and UI surface:

1. **Admin panel redesign** — edit/delete all stories (including hardcoded), bulk add
2. **Ghost layer** — stories about places that no longer exist render as breathing ghost markers
3. **i18n** — site-wide language switcher: Turkish (TR), English (EN), Western Armenian (ՀՅ)

---

## 2. Storage Layer

### Seed migration
- On first admin login, check `localStorage.getItem('uskudar-seeded')`.
- If absent: copy the hardcoded `stories` array into `localStorage` under key `uskudar-stories`, then set `uskudar-seeded = '1'`.
- From this point, `localStorage` is the single source of truth. The hardcoded array is never read again at runtime.
- All subsequent reads (map rendering, filter, popups) pull from `localStorage` only.

### Story schema additions
Two new fields added to every story object:

```js
{
  // existing fields unchanged
  id, person, title, lat, lng, emotions, narrative, videoUrl, locationKnown,

  // NEW
  period: string,       // optional — e.g. "1980ler", "Childhood years"
  isGhost: boolean,     // true = place no longer exists → ghost marker
}
```

Existing stories without these fields are treated as `period: ''`, `isGhost: false`.

---

## 3. Admin Panel

The existing "Add Story" modal is replaced with a full **Story Management modal** triggered by the same admin login flow.

### Modal structure
- Title: TR "Tanıklık Yönetimi" / EN "Story Management" / ՀՅ [placeholder]
- Two tabs: **Stories** (list + edit) | **Bulk Add**

---

### Tab 1 — Stories (list view, default)

**Search row:**
- Text input: placeholder TR "Ara…" / EN "Search…" / ՀՅ [placeholder]
- Person dropdown: "Tümü / All / [HY placeholder]" + one option per unique person

**Story list:**
- Flat scrollable list, max-height ~260px, overflow-y scroll
- Each row: `[title · person]` — `[✎ Düzenle / Edit / [HY]] [× Sil / Delete / [HY]]`
- Count line below: TR "N tanıklık" / EN "N stories" / ՀՅ [placeholder]
- Filter updates the list live (no submit)

**Edit view (replace in place):**
- Clicking Düzenle/Edit replaces the list area with the edit form
- Back button: TR "← Listeye dön" / EN "← Back to list" / ՀՅ [placeholder]
- Fields (in order):
  1. Person — text input
  2. Title * — text input
  3. Narrative * — textarea
  4. Period — text input, TR "Dönem (isteğe bağlı)" / EN "Time period (optional)" / ՀՅ [placeholder]; e.g. "1980ler"
  5. Lat / Lng * — two number inputs side by side
  6. Ghost checkbox — label: TR "Mevcut değil" / EN "No longer exists" / ՀՅ "Այլեւս չկայ"
  7. Emotions * — pill toggles (at least one required)
  8. Video URL — optional text input
- Action row: Cancel | Save Changes (TR "Değişiklikleri Kaydet" / EN "Save Changes" / ՀՅ [placeholder])
- Save writes the updated story back to `localStorage`, re-renders the map marker in place

---

### Tab 2 — Bulk Add

**Shared person field:**
- Row at top: TR "Görüşmeci:" / EN "Person:" / ՀՅ [placeholder] + text input
- Hint: TR "← tüm kartlara uygulanır" / EN "← applies to all cards" / ՀՅ [placeholder]

**Story cards:**
- Each card contains: Title *, Lat *, Lng *, Narrative *, Period (optional), Ghost checkbox, Emotions *
- × button top-right of each card removes it
- "+ Tanıklık ekle / + Add story / [HY placeholder]" button adds a new blank card below
- Save button at bottom: TR "Tümünü Kaydet (N tanıklık)" / EN "Save All (N stories)" / ՀՅ [placeholder]
- Validation: each card must have title, lat, lng, narrative, ≥1 emotion before saving
- Invalid cards show inline error; valid cards save even if some are invalid (user confirms)

---

## 4. Ghost Marker

### Trigger
`isGhost: true` on a story object. Set via the ghost checkbox in the add/edit form.

### Visual behaviour
Ghost markers use the same `buildParticleMarker` canvas function with a `ghost` mode:

- **Particle count:** 70 (same)
- **Cluster radius:** same as normal (14px for known location, 20px for drifting)
- **Color:** emotion hex desaturated 85% toward grey — hue is preserved at ~15% saturation
  ```js
  function desaturate([r,g,b], factor = 0.85) {
    const grey = r*0.299 + g*0.587 + b*0.114;
    return [r + (grey-r)*factor, g + (grey-g)*factor, b + (grey-b)*factor];
  }
  ```
- **Alpha:** breathes continuously on ~2.5s cycle per dot, random phase offset per particle
  ```js
  const alpha = 0.08 + 0.37 * (0.5 + 0.5 * Math.sin(t * 0.0025 + p.phase));
  ```
- **Physics:** identical — mouse repulsion and spring-back still active
- **Animation loop:** ghost markers always run `requestAnimationFrame` (never idle), normal markers idle when not hovered

### Map legend
A small legend is added to the map (bottom-right or bottom-left, outside Mapbox controls):
- ● Colored dot — TR "Mevcut mekan" / EN "Active place" / ՀՅ [placeholder]
- ○ Faded dot — TR "Artık mevcut değil" / EN "No longer exists" / ՀՅ "Այլեւս չկայ"

---

## 5. i18n

### Language switcher UI
- Floating pill, top-right of the map area (same position as Mapbox zoom controls but opposite corner)
- **Collapsed:** shows only the active language — `TR`, `EN`, or `ՀՅ`
- **Hover/focus:** expands to show all three; click switches language
- Active language stored in `localStorage` under key `uskudar-lang`; defaults to `'tr'`
- Western Armenian abbreviation: **ՀՅ** (Armenian script, not Latin)

### String coverage
All visible UI strings switch language. Story **narrative, title, person, period** fields are NOT translated — they display as entered.

Strings that switch:
- Panel title, section headers (People, Emotions)
- Filter chip labels (emotion names)
- Story popup field labels (Narrative, Time Period, etc.) — Duyular/Sensory removed
- Admin modal: all labels, buttons, placeholders, error messages
- Ghost checkbox label
- Map legend
- Add/Edit/Delete/Save/Cancel/Back buttons
- Search placeholder, person dropdown options ("All")
- Bulk add: person label, hint text, add card button, save button

### Western Armenian policy
**Do not attempt Western Armenian translations.** All ՀՅ strings are set to a clearly marked placeholder:
```js
hy: '[ՀՅ]'   // needs translation
```
The ghost checkbox label is the only exception — provided by the user:
```js
hy: 'Այլեւս չկայ'
```

### i18n data structure
```js
const i18n = {
  tr: { panelTitle: 'Tanıklık Yönetimi', searchPlaceholder: 'Ara…', /* … */ ghostLabel: 'Mevcut değil', /* … */ },
  en: { panelTitle: 'Story Management',  searchPlaceholder: 'Search…', /* … */ ghostLabel: 'No longer exists', /* … */ },
  hy: { panelTitle: '[ՀՅ]',              searchPlaceholder: '[ՀՅ]',     /* … */ ghostLabel: 'Այլեւս չկայ',    /* … */ },
};
let currentLang = localStorage.getItem('uskudar-lang') || 'tr';
```

A single `applyLang(lang)` function walks all DOM elements with a `data-i18n="key"` attribute and sets their `textContent` / `placeholder`. Called on page load and on language switch.

---

## 6. Out of Scope

- Translating story narrative, title, person, or period text — stays as entered
- Any backend / server — everything remains localStorage + static HTML
- Importing stories from external files (JSON import was considered and rejected in favour of bulk add UI)
- Undo/redo for edits

---

## 7. Open Items

- [ ] Western Armenian UI strings — all marked `[ՀՅ]`, to be filled in by the user
- [ ] Exact position of map legend (bottom-left vs bottom-right — depends on Mapbox control placement)
- [ ] Whether the "period" field appears in the popup card shown to visitors (assumed yes, but confirm)
