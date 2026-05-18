# Admin Panel Redesign, Ghost Layer & i18n — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the admin panel to support editing/deleting all stories and bulk-adding, add ghost markers for lost places, and add a TR/EN/ՀՅ language switcher across the entire site.

**Architecture:** All code lives in a single `index.html` file (~1675 lines). Changes are additive — new functions and data structures are inserted without restructuring existing ones. The `stories` array is migrated to `localStorage` on first admin login and becomes the sole runtime data source. A global `i18n` object and `applyLang()` function drive all string switching via `data-i18n` attributes.

**Tech Stack:** Vanilla HTML/CSS/JS, Mapbox GL JS 2.15, localStorage

**Spec:** `docs/superpowers/specs/2026-05-18-admin-i18n-ghost-design.md`

---

## File map

| Section | Lines (approx) | What changes |
|---|---|---|
| CSS | 1–630 | Add styles for: ghost checkbox, lang pill, admin tabs, bulk-add cards, edit form |
| HTML — left panel header | 642–658 | Add `data-i18n` to panel title; add lang switcher pill |
| HTML — filter section headers | 659–680 | Add `data-i18n` to "Kişiler" / "Duygular" headings |
| HTML — admin login modal | 696–710 | Add `data-i18n` to labels/buttons |
| HTML — story management modal | 712–753 | Full replacement of add-story modal with tabbed management modal |
| JS — config block | 759–969 | Add `i18n` object; add `currentLang`; add `desaturate()` helper |
| JS — seed migration | 972–980 | Replace `loadStoredStories` IIFE with full seed+load function |
| JS — `buildParticleMarker` | 1226–1332 | Add `ghost` parameter; ghost path uses breathing alpha + desaturated colors |
| JS — `addMarkerForStory` | 1334–1395 | Pass `story.isGhost` to `buildParticleMarker`; update popup HTML (add period, remove sensory) |
| JS — `applyLang` | after config | New function; called on load and on lang switch |
| JS — admin functions | 1517–1675 | Replace `openAddStory`, `closeAddStory`, `submitNewStory`, `deleteStoredStory` with new management panel functions |
| JS — map legend | after map load | Inject legend DOM element into map container |

---

## Task 1: i18n data structure + `applyLang`

**Files:**
- Modify: `index.html` — JS config block (~line 959, after `emotionData`)

- [ ] **Step 1: Insert the `i18n` object and `currentLang` after the `emotionData` block (around line 968)**

```js
// ========================================
// i18n
// ========================================
const i18n = {
    tr: {
        panelTitle:        'Burada Yaşadık',
        peopleHeader:      'Kişiler',
        emotionsHeader:    'Duygular',
        toggleNone:        'Hiçbirini seçme',
        searchPlaceholder: 'Ara…',
        filterAll:         'Tümü',
        storyCount:        n => `${n} tanıklık`,
        editBtn:           'Düzenle',
        deleteBtn:         'Sil',
        backBtn:           '← Listeye dön',
        saveBtn:           'Değişiklikleri Kaydet',
        cancelBtn:         'İptal',
        addCardBtn:        '+ Tanıklık ekle',
        saveAllBtn:        n => `Tümünü Kaydet (${n} tanıklık)`,
        tabStories:        'Tanıklıklar',
        tabBulk:           'Toplu Ekle',
        modalTitle:        'Tanıklık Yönetimi',
        personLabel:       'Kişi',
        titleLabel:        'Başlık',
        narrativeLabel:    'Anlatı',
        periodLabel:       'Dönem (isteğe bağlı)',
        periodPlaceholder: '1980ler, Çocukluk…',
        latLabel:          'Enlem',
        lngLabel:          'Boylam',
        ghostLabel:        'Mevcut değil',
        emotionLabel:      'Duygular',
        videoLabel:        'YouTube URL (isteğe bağlı)',
        bulkPersonLabel:   'Görüşmeci:',
        bulkPersonHint:    '← tüm kartlara uygulanır',
        errorRequired:     'Zorunlu alanları doldurun ve en az bir duygu seçin.',
        legendActive:      'Mevcut mekan',
        legendGhost:       'Artık mevcut değil',
        watchBtn:          '▶ Tanıklığı İzle',
        popupPeriod:       'Dönem',
        adminLoginTitle:   'Yönetici Girişi',
        adminUser:         'Kullanıcı Adı',
        adminPass:         'Şifre',
        adminError:        'Kullanıcı adı veya şifre hatalı.',
        adminLoginBtn:     'Giriş',
    },
    en: {
        panelTitle:        'We Lived Here',
        peopleHeader:      'People',
        emotionsHeader:    'Emotions',
        toggleNone:        'Deselect all',
        searchPlaceholder: 'Search…',
        filterAll:         'All',
        storyCount:        n => `${n} stories`,
        editBtn:           'Edit',
        deleteBtn:         'Delete',
        backBtn:           '← Back to list',
        saveBtn:           'Save Changes',
        cancelBtn:         'Cancel',
        addCardBtn:        '+ Add story',
        saveAllBtn:        n => `Save All (${n} stories)`,
        tabStories:        'Stories',
        tabBulk:           'Bulk Add',
        modalTitle:        'Story Management',
        personLabel:       'Person',
        titleLabel:        'Title',
        narrativeLabel:    'Narrative',
        periodLabel:       'Time period (optional)',
        periodPlaceholder: '1980s, Childhood…',
        latLabel:          'Latitude',
        lngLabel:          'Longitude',
        ghostLabel:        'No longer exists',
        emotionLabel:      'Emotions',
        videoLabel:        'YouTube URL (optional)',
        bulkPersonLabel:   'Person:',
        bulkPersonHint:    '← applies to all cards',
        errorRequired:     'Fill in required fields and select at least one emotion.',
        legendActive:      'Active place',
        legendGhost:       'No longer exists',
        watchBtn:          '▶ Watch testimony',
        popupPeriod:       'Period',
        adminLoginTitle:   'Admin Login',
        adminUser:         'Username',
        adminPass:         'Password',
        adminError:        'Incorrect username or password.',
        adminLoginBtn:     'Login',
    },
    hy: {
        // Western Armenian — all placeholders except where provided by user
        panelTitle:        '[ՀՅ]',
        peopleHeader:      '[ՀՅ]',
        emotionsHeader:    '[ՀՅ]',
        toggleNone:        '[ՀՅ]',
        searchPlaceholder: '[ՀՅ]',
        filterAll:         '[ՀՅ]',
        storyCount:        n => `[ՀՅ] ${n}`,
        editBtn:           '[ՀՅ]',
        deleteBtn:         '[ՀՅ]',
        backBtn:           '[ՀՅ]',
        saveBtn:           '[ՀՅ]',
        cancelBtn:         '[ՀՅ]',
        addCardBtn:        '[ՀՅ]',
        saveAllBtn:        n => `[ՀՅ] (${n})`,
        tabStories:        '[ՀՅ]',
        tabBulk:           '[ՀՅ]',
        modalTitle:        '[ՀՅ]',
        personLabel:       '[ՀՅ]',
        titleLabel:        '[ՀՅ]',
        narrativeLabel:    '[ՀՅ]',
        periodLabel:       '[ՀՅ]',
        periodPlaceholder: '[ՀՅ]',
        latLabel:          '[ՀՅ]',
        lngLabel:          '[ՀՅ]',
        ghostLabel:        'Այլեւս չկայ',
        emotionLabel:      '[ՀՅ]',
        videoLabel:        '[ՀՅ]',
        bulkPersonLabel:   '[ՀՅ]',
        bulkPersonHint:    '[ՀՅ]',
        errorRequired:     '[ՀՅ]',
        legendActive:      '[ՀՅ]',
        legendGhost:       'Այլեւս չկայ',
        watchBtn:          '[ՀՅ]',
        popupPeriod:       '[ՀՅ]',
        adminLoginTitle:   '[ՀՅ]',
        adminUser:         '[ՀՅ]',
        adminPass:         '[ՀՅ]',
        adminError:        '[ՀՅ]',
        adminLoginBtn:     '[ՀՅ]',
    },
};

let currentLang = localStorage.getItem('uskudar-lang') || 'tr';

function t(key) {
    return i18n[currentLang][key] ?? i18n.tr[key] ?? key;
}

function applyLang(lang) {
    currentLang = lang;
    localStorage.setItem('uskudar-lang', lang);

    // Static text nodes
    document.querySelectorAll('[data-i18n]').forEach(el => {
        const key = el.dataset.i18n;
        if (i18n[lang][key] !== undefined && typeof i18n[lang][key] === 'string') {
            el.textContent = i18n[lang][key];
        }
    });
    // Placeholder attributes
    document.querySelectorAll('[data-i18n-placeholder]').forEach(el => {
        const key = el.dataset.i18nPlaceholder;
        if (i18n[lang][key] !== undefined) {
            el.placeholder = i18n[lang][key];
        }
    });

    // Lang pill: update active state
    document.querySelectorAll('.lang-option').forEach(btn => {
        btn.classList.toggle('active', btn.dataset.lang === lang);
    });

    // Rebuild dynamic lists that include translated strings
    if (typeof buildPeopleFilter === 'function') buildPeopleFilter();
    if (typeof buildEmotionFilter === 'function') buildEmotionFilter();
    if (typeof renderStoryList === 'function') renderStoryList();
}
```

- [ ] **Step 2: Verify the object loads without errors**

Open `index.html` in browser. Open DevTools console. Type:
```js
t('panelTitle')   // → "Burada Yaşadık"
t('ghostLabel')   // → "Mevcut değil"
```
Expected: correct strings, no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add i18n data structure, t() helper, and applyLang()"
```

---

## Task 2: `data-i18n` attributes on existing HTML

**Files:**
- Modify: `index.html` — HTML section (lines 642–710)

- [ ] **Step 1: Add `data-i18n` to the left panel header and filter section labels**

Find the panel title element and add `data-i18n="panelTitle"`. Find the "Kişiler" and "Duygular" section labels and add `data-i18n="peopleHeader"` / `data-i18n="emotionsHeader"`. Find the "Hiçbirini seçme" toggle buttons and add `data-i18n="toggleNone"`.

Exact changes (locate by text content, not line number — file shifts with edits):

```html
<!-- Panel title — find the <h1> or title element in the left panel -->
<h1 data-i18n="panelTitle">Burada Yaşadık</h1>

<!-- People section header -->
<div class="filter-section-label" data-i18n="peopleHeader">Kişiler</div>
<button class="toggle-all-link" id="peopleToggleAll" data-i18n="toggleNone">Hiçbirini seçme</button>

<!-- Emotions section header -->
<div class="filter-section-label" data-i18n="emotionsHeader">Duygular</div>
<button class="toggle-all-link" id="emotionToggleAll" data-i18n="toggleNone">Hiçbirini seçme</button>
```

- [ ] **Step 2: Add `data-i18n` to admin login modal**

```html
<h2 data-i18n="adminLoginTitle">Yönetici Girişi</h2>
<span class="modal-field-label" data-i18n="adminUser">Kullanıcı Adı</span>
<span class="modal-field-label" data-i18n="adminPass">Şifre</span>
<div class="modal-error" id="admin-error" data-i18n="adminError">Kullanıcı adı veya şifre hatalı.</div>
<button class="btn-primary" onclick="submitAdminLogin()" data-i18n="adminLoginBtn">Giriş</button>
```

- [ ] **Step 3: Verify `applyLang('en')` switches panel strings**

In browser console:
```js
applyLang('en')
// Panel title should read "We Lived Here"
// People header should read "People"
applyLang('tr')
// Reverts
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add data-i18n attributes to existing HTML elements"
```

---

## Task 3: Language switcher pill UI

**Files:**
- Modify: `index.html` — CSS block; HTML map area

- [ ] **Step 1: Add CSS for the language pill**

Insert in the CSS block (before `</style>`):

```css
/* ── Language switcher ─────────────────────── */
#lang-switcher {
    position: absolute;
    top: 10px;
    right: 10px;
    z-index: 10;
    display: flex;
    align-items: center;
    background: rgba(255,255,255,0.92);
    border: 1px solid rgba(0,0,0,0.1);
    border-radius: 20px;
    padding: 4px 8px;
    gap: 0;
    box-shadow: 0 1px 4px rgba(0,0,0,0.12);
    overflow: hidden;
    max-width: 32px;
    transition: max-width 0.25s ease;
    cursor: pointer;
}
#lang-switcher:hover,
#lang-switcher:focus-within {
    max-width: 120px;
}
.lang-option {
    font-size: 11px;
    font-weight: 700;
    color: #aaa;
    padding: 2px 6px;
    border-radius: 12px;
    cursor: pointer;
    white-space: nowrap;
    border: none;
    background: none;
    transition: color 0.15s, background 0.15s;
    display: none;
}
#lang-switcher:hover .lang-option,
#lang-switcher:focus-within .lang-option {
    display: inline-block;
}
.lang-option.active {
    display: inline-block;
    color: #1a1a1a;
    background: #f0f0f0;
}
.lang-option:hover {
    color: #1a1a1a;
}
```

- [ ] **Step 2: Add the pill HTML just inside the map container div**

Find `<div id="map">` (or the map wrapper) and insert immediately after its opening tag:

```html
<div id="lang-switcher" role="group" aria-label="Language">
    <button class="lang-option active" data-lang="tr" onclick="applyLang('tr')">TR</button>
    <button class="lang-option" data-lang="en" onclick="applyLang('en')">EN</button>
    <button class="lang-option" data-lang="hy" onclick="applyLang('hy')">ՀՅ</button>
</div>
```

- [ ] **Step 3: Call `applyLang(currentLang)` on page load**

At the very bottom of the `<script>` block (after all function definitions, before `</script>`), add:

```js
// Apply saved language on load
applyLang(currentLang);
```

- [ ] **Step 4: Verify pill in browser**

- Load page. Pill shows "TR" collapsed top-right of map.
- Hover: expands to show TR · EN · ՀՅ.
- Click EN: panel title changes to "We Lived Here".
- Reload page: EN is remembered (localStorage).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add language switcher pill (TR/EN/ՀՅ) with hover expand"
```

---

## Task 4: Storage seed migration

**Files:**
- Modify: `index.html` — `loadStoredStories` IIFE (~line 972) and `stories` array usage

- [ ] **Step 1: Replace the `loadStoredStories` IIFE with a seed+load function**

Find the existing `(function loadStoredStories() { … })();` block and replace it entirely:

```js
// ========================================
// STORAGE — seed + load
// ========================================
function seedAndLoadStories() {
    if (!localStorage.getItem('uskudar-seeded')) {
        // First admin login hasn't happened yet — seed will happen on login.
        // For now just load hardcoded stories as runtime source.
    }
    const saved = localStorage.getItem('uskudar-stories');
    if (saved) {
        const parsed = JSON.parse(saved);
        parsed.forEach(s => {
            // Backfill new fields on old objects
            if (s.period === undefined) s.period = '';
            if (s.isGhost === undefined) s.isGhost = false;
            stories.push(s);
        });
    }
}
seedAndLoadStories();
```

- [ ] **Step 2: Add `seedStoriesFromHardcoded()` — called on first admin login**

Insert this function near the admin functions block (~line 1517):

```js
function seedStoriesFromHardcoded() {
    if (localStorage.getItem('uskudar-seeded')) return;
    // SEED_STORIES is the original hardcoded array defined at the top.
    // We copy it to localStorage, adding new fields with defaults.
    const seeded = SEED_STORIES.map(s => ({
        ...s,
        period:  s.period  ?? '',
        isGhost: s.isGhost ?? false,
    }));
    localStorage.setItem('uskudar-stories', JSON.stringify(seeded));
    localStorage.setItem('uskudar-seeded', '1');
    // Reload so the seeded data drives the map
    location.reload();
}
```

- [ ] **Step 3: Rename the `stories` const to `SEED_STORIES`**

At line ~766, change:
```js
const stories = [
```
to:
```js
const SEED_STORIES = [
```

Then change the `stories` variable that the rest of the code uses into a mutable array loaded from storage:

```js
// Runtime story list — filled by seedAndLoadStories()
let stories = [];
```

Insert this line immediately before the `seedAndLoadStories()` call.

- [ ] **Step 4: Add `getAllStories()` helper used by rendering and admin code**

```js
function getAllStories() {
    return stories;
}

function saveStories() {
    localStorage.setItem('uskudar-stories', JSON.stringify(stories));
}
```

- [ ] **Step 5: Call `seedStoriesFromHardcoded()` inside `submitAdminLogin()` before opening the panel**

Find `submitAdminLogin()` (~line 1533). After `closeAdminLogin()`, add:

```js
seedStoriesFromHardcoded(); // no-op after first run; reloads on first run
openStoryManagement();
```

Remove the call to `openAddStory()` from `submitAdminLogin()` — it will be replaced by `openStoryManagement()` in Task 6.

- [ ] **Step 6: Verify seed behaviour**

1. Clear localStorage: `localStorage.clear()` in console, reload.
2. Log in as admin (admin / uskudar2025).
3. Page reloads. Open console: `JSON.parse(localStorage.getItem('uskudar-stories')).length` → should equal the count of hardcoded stories.
4. Log in again: no reload this time (already seeded).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: migrate hardcoded stories to localStorage on first admin login"
```

---

## Task 5: Ghost marker — `buildParticleMarker` update

**Files:**
- Modify: `index.html` — `buildParticleMarker` function (~line 1226) and `addMarkerForStory` (~line 1334)

- [ ] **Step 1: Add `desaturate` helper before `buildParticleMarker`**

```js
function desaturate(hex, factor) {
    const r = parseInt(hex.slice(1,3), 16);
    const g = parseInt(hex.slice(3,5), 16);
    const b = parseInt(hex.slice(5,7), 16);
    const grey = r*0.299 + g*0.587 + b*0.114;
    return [
        Math.round(r + (grey - r) * factor),
        Math.round(g + (grey - g) * factor),
        Math.round(b + (grey - b) * factor),
    ];
}
```

- [ ] **Step 2: Add `ghost` parameter to `buildParticleMarker`**

Change the function signature from:
```js
function buildParticleMarker(emotions, physicalSize, drifting) {
```
to:
```js
function buildParticleMarker(emotions, physicalSize, drifting, ghost) {
```

- [ ] **Step 3: In the particle-building loop, compute ghost color when `ghost` is true**

Find the particle creation block (inside `Array.from`). Change:
```js
color: emotionData[emotion].hex
```
to:
```js
color: ghost
    ? desaturate(emotionData[emotion].hex, 0.85)  // returns [r,g,b]
    : emotionData[emotion].hex
```

Store whether ghost on each particle:
```js
return {
    x: hx, y: hy, hx, hy,
    vx: 0, vy: 0,
    r: (0.8 + Math.random() * 0.7) * scale,
    color: emotionData[emotion].hex,          // keep for normal
    ghostRgb: desaturate(emotionData[emotion].hex, 0.85),  // precomputed
    phase: Math.random() * Math.PI * 2,       // for breathing
};
```

- [ ] **Step 4: Split the `tick` function into normal and ghost rendering paths**

For **ghost** particles, replace:
```js
ctx.fillStyle = p.color;
ctx.fill();
```
with:
```js
if (ghost) {
    const alpha = 0.08 + 0.37 * (0.5 + 0.5 * Math.sin(performance.now() * 0.0025 + p.phase));
    const [r, g, b] = p.ghostRgb;
    ctx.fillStyle = `rgba(${r},${g},${b},${alpha.toFixed(3)})`;
} else {
    ctx.fillStyle = p.color;
}
ctx.fill();
```

For ghost markers, `rafId` must **never be null** — they always animate. Change the end of `tick`:
```js
rafId = ghost
    ? requestAnimationFrame(tick)
    : ((hovering || anyMoving) ? requestAnimationFrame(tick) : null);
```

- [ ] **Step 5: Pass `ghost` flag from `addMarkerForStory`**

Find:
```js
inner.appendChild(buildParticleMarker(story.emotions, null, story.locationKnown === false));
```
Change to:
```js
inner.appendChild(buildParticleMarker(
    story.emotions,
    null,
    story.locationKnown === false,
    !!story.isGhost
));
```

- [ ] **Step 6: Verify ghost rendering**

In browser console:
```js
// Temporarily mark story id=1 as ghost and rebuild its marker
stories[0].isGhost = true;
markerObjects[1].remove();
delete markerObjects[1];
delete markerElements[1];
addMarkerForStory(stories[0]);
```
Expected: marker for story 1 now pulses slowly in desaturated colours.

Undo: `stories[0].isGhost = false;` and re-add.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: ghost particle marker — breathing alpha, desaturated emotion hue"
```

---

## Task 6: Popup update — add period, remove sensory

**Files:**
- Modify: `index.html` — `addMarkerForStory` popup HTML (~line 1355)

- [ ] **Step 1: Replace the popup HTML template in `addMarkerForStory`**

Find the `popupHTML` template string and replace it:

```js
const periodHTML = story.period
    ? `<div style="font-size:11px;color:#888;margin-bottom:0.5rem;">
           <strong>${t('popupPeriod')}:</strong> ${story.period}
       </div>`
    : '';

const popupHTML = `
    <h3>${story.title}</h3>
    <div class="person-name"><strong>${story.person}</strong></div>
    <div style="margin-bottom:0.75rem;">${emotionTags}</div>
    ${periodHTML}
    <p>${story.narrative}</p>
    ${story.videoUrl ? `<div style="text-align:center; margin-top: 0.75rem;">
        <button class="play-video-btn"
            onclick="openVideoPanel('${story.videoUrl}', '${story.title} — ${story.person}')">
            ${t('watchBtn')}
        </button>
    </div>` : ''}
`;
```

Note: the old `story.sensory` div is simply removed.

- [ ] **Step 2: Verify popup in browser**

Click a marker. Popup should show title, person, emotion chips, narrative. No "Duyular" line. If story has a period, it shows between emotion chips and narrative.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: update story popup — add period field, remove sensory"
```

---

## Task 7: Map legend

**Files:**
- Modify: `index.html` — CSS block; JS map `load` event handler

- [ ] **Step 1: Add legend CSS**

```css
/* ── Map legend ─────────────────────────── */
#map-legend {
    position: absolute;
    bottom: 28px;
    left: 10px;
    z-index: 5;
    background: rgba(255,255,255,0.9);
    border: 1px solid rgba(0,0,0,0.08);
    border-radius: 8px;
    padding: 8px 12px;
    font-size: 10px;
    color: #555;
    pointer-events: none;
}
.legend-row {
    display: flex;
    align-items: center;
    gap: 7px;
    margin-bottom: 4px;
}
.legend-row:last-child { margin-bottom: 0; }
.legend-dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    flex-shrink: 0;
}
```

- [ ] **Step 2: Inject the legend after the map loads**

Inside the `map.on('load', () => { … })` handler, at the very end (before the closing `}`), add:

```js
// Map legend
const legend = document.createElement('div');
legend.id = 'map-legend';
legend.innerHTML = `
    <div class="legend-row">
        <div class="legend-dot" style="background:#4A90E2"></div>
        <span data-i18n="legendActive">${t('legendActive')}</span>
    </div>
    <div class="legend-row">
        <div class="legend-dot" style="background:rgba(100,105,130,0.3);border:1.5px solid rgba(100,105,130,0.4)"></div>
        <span data-i18n="legendGhost">${t('legendGhost')}</span>
    </div>
`;
document.getElementById('map').appendChild(legend);
```

- [ ] **Step 3: Verify legend appears bottom-left of map**

Load page. Small legend with "Mevcut mekan" and "Artık mevcut değil" visible. Switch to EN — legend text updates via `applyLang`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add ghost/active legend to map"
```

---

## Task 8: Story management modal — HTML structure

**Files:**
- Modify: `index.html` — replace add-story modal HTML (lines ~712–753)

- [ ] **Step 1: Replace the entire `add-story-overlay` div with the new management modal**

Remove:
```html
<!-- Add story modal -->
<div class="modal-overlay" id="add-story-overlay">
    … (everything up to and including the closing </div></div>)
```

Replace with:

```html
<!-- Story management modal -->
<div class="modal-overlay" id="story-mgmt-overlay">
    <div class="modal-box" style="max-height:90vh;overflow-y:auto;width:420px">

        <h2 data-i18n="modalTitle">Tanıklık Yönetimi</h2>

        <!-- Tabs -->
        <div class="mgmt-tabs">
            <button class="mgmt-tab active" id="tab-stories" onclick="switchMgmtTab('stories')"
                data-i18n="tabStories">Tanıklıklar</button>
            <button class="mgmt-tab" id="tab-bulk" onclick="switchMgmtTab('bulk')"
                data-i18n="tabBulk">Toplu Ekle</button>
        </div>

        <!-- ── TAB 1: Stories list + edit ── -->
        <div id="mgmt-stories-pane">

            <!-- List view -->
            <div id="mgmt-list-view">
                <div class="mgmt-search-row">
                    <input type="text" id="mgmt-search"
                        data-i18n-placeholder="searchPlaceholder"
                        placeholder="Ara…"
                        oninput="renderStoryList()">
                    <select id="mgmt-person-filter" onchange="renderStoryList()">
                        <option value="" data-i18n="filterAll">Tümü</option>
                    </select>
                </div>
                <div class="mgmt-story-list" id="mgmt-story-list"></div>
                <div class="mgmt-count" id="mgmt-count"></div>
            </div>

            <!-- Edit view (hidden by default) -->
            <div id="mgmt-edit-view" style="display:none">
                <button class="mgmt-back-btn" onclick="showListView()"
                    data-i18n="backBtn">← Listeye dön</button>

                <span class="modal-field-label" data-i18n="personLabel">Kişi</span>
                <input type="text" id="edit-person" class="modal-input">

                <span class="modal-field-label" data-i18n="titleLabel">Başlık *</span>
                <input type="text" id="edit-title" class="modal-input">

                <span class="modal-field-label" data-i18n="narrativeLabel">Anlatı *</span>
                <textarea id="edit-narrative" class="modal-textarea"></textarea>

                <span class="modal-field-label" data-i18n="periodLabel">Dönem (isteğe bağlı)</span>
                <input type="text" id="edit-period" class="modal-input"
                    data-i18n-placeholder="periodPlaceholder" placeholder="1980ler, Çocukluk…">

                <span class="modal-field-label">Konum *</span>
                <div class="coord-row">
                    <div><input type="number" id="edit-lat" step="0.00001"
                        data-i18n-placeholder="latLabel" placeholder="Enlem"></div>
                    <div><input type="number" id="edit-lng" step="0.00001"
                        data-i18n-placeholder="lngLabel" placeholder="Boylam"></div>
                </div>

                <div class="inline-check" style="margin-top:8px">
                    <input type="checkbox" id="edit-ghost">
                    <label for="edit-ghost" data-i18n="ghostLabel">Mevcut değil</label>
                </div>

                <span class="modal-field-label" data-i18n="emotionLabel">Duygular *</span>
                <div class="emotion-check-grid" id="edit-emotion-checkboxes"></div>

                <span class="modal-field-label" data-i18n="videoLabel">YouTube URL (isteğe bağlı)</span>
                <input type="text" id="edit-video" class="modal-input">

                <div class="modal-error" id="edit-error" style="display:none"
                    data-i18n="errorRequired">Zorunlu alanları doldurun.</div>
                <div class="modal-actions">
                    <button class="btn-secondary" onclick="showListView()"
                        data-i18n="cancelBtn">İptal</button>
                    <button class="btn-primary" onclick="saveEditedStory()"
                        data-i18n="saveBtn">Değişiklikleri Kaydet</button>
                </div>
            </div>
        </div>

        <!-- ── TAB 2: Bulk Add ── -->
        <div id="mgmt-bulk-pane" style="display:none">
            <div class="bulk-person-row">
                <label data-i18n="bulkPersonLabel">Görüşmeci:</label>
                <input type="text" id="bulk-person" class="modal-input" style="flex:1">
                <span class="bulk-person-hint" data-i18n="bulkPersonHint">← tüm kartlara uygulanır</span>
            </div>
            <div id="bulk-cards"></div>
            <button class="bulk-add-card-btn" onclick="addBulkCard()"
                data-i18n="addCardBtn">+ Tanıklık ekle</button>
            <button class="btn-primary bulk-save-btn" id="bulk-save-btn"
                onclick="saveBulkStories()">Tümünü Kaydet</button>
        </div>

        <div class="modal-actions" style="margin-top:12px;border-top:1px solid #f0f0f0;padding-top:12px">
            <button class="btn-secondary" onclick="closeStoryManagement()"
                data-i18n="cancelBtn">İptal</button>
        </div>
    </div>
</div>
```

- [ ] **Step 2: Add CSS for the new modal elements**

```css
/* ── Story management modal ─────────────── */
.mgmt-tabs {
    display: flex;
    border-bottom: 1.5px solid #f0f0f0;
    margin-bottom: 12px;
}
.mgmt-tab {
    background: none;
    border: none;
    border-bottom: 2px solid transparent;
    padding: 6px 14px;
    font-size: 12px;
    color: #aaa;
    cursor: pointer;
    margin-bottom: -2px;
    font-weight: 500;
}
.mgmt-tab.active { color: #1a1a1a; font-weight: 700; border-bottom-color: #1a1a1a; }
.mgmt-search-row { display: flex; gap: 6px; margin-bottom: 8px; }
.mgmt-search-row input { flex: 1; }
.mgmt-story-list { border: 1px solid #eee; border-radius: 7px; overflow: hidden; max-height: 260px; overflow-y: auto; }
.mgmt-story-row { display: flex; align-items: center; justify-content: space-between; padding: 8px 10px; border-bottom: 1px solid #f5f5f5; font-size: 11px; }
.mgmt-story-row:last-child { border-bottom: none; }
.mgmt-story-info { flex: 1; min-width: 0; }
.mgmt-story-info strong { display: block; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.mgmt-story-info span { font-size: 10px; color: #bbb; }
.mgmt-story-actions { display: flex; gap: 4px; flex-shrink: 0; }
.mgmt-btn-edit { background: #f4f4f4; border: none; border-radius: 4px; padding: 3px 8px; font-size: 10px; cursor: pointer; }
.mgmt-btn-del  { background: #fff0f0; border: none; border-radius: 4px; padding: 3px 8px; font-size: 10px; color: #e74c3c; cursor: pointer; }
.mgmt-count { font-size: 10px; color: #bbb; margin-top: 5px; }
.mgmt-back-btn { background: none; border: none; font-size: 11px; color: #888; cursor: pointer; padding: 0; margin-bottom: 10px; }
.mgmt-back-btn:hover { color: #1a1a1a; }
/* Bulk add */
.bulk-person-row { display: flex; align-items: center; gap: 8px; background: #fffbf0; border: 1px solid #ffe4a0; border-radius: 7px; padding: 8px 10px; margin-bottom: 10px; }
.bulk-person-hint { font-size: 9px; color: #bbb; white-space: nowrap; }
.bulk-card { border: 1px solid #e8e8e8; border-radius: 7px; padding: 10px; background: #fff; margin-bottom: 8px; position: relative; }
.bulk-card-num { font-size: 9px; font-weight: 700; color: #bbb; text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 6px; }
.bulk-remove-btn { position: absolute; top: 8px; right: 8px; background: none; border: none; color: #ddd; font-size: 14px; cursor: pointer; line-height: 1; }
.bulk-remove-btn:hover { color: #e74c3c; }
.bulk-add-card-btn { width: 100%; border: 1.5px dashed #ddd; border-radius: 7px; padding: 8px; background: #fafafa; color: #bbb; font-size: 11px; cursor: pointer; margin-bottom: 8px; }
.bulk-add-card-btn:hover { border-color: #aaa; color: #888; }
.bulk-save-btn { width: 100%; }
.modal-input { width: 100%; border: 1px solid #e5e5e5; border-radius: 6px; padding: 7px 10px; font-size: 12px; margin-bottom: 4px; font-family: inherit; box-sizing: border-box; }
.modal-textarea { width: 100%; border: 1px solid #e5e5e5; border-radius: 6px; padding: 7px 10px; font-size: 12px; height: 80px; resize: vertical; font-family: inherit; box-sizing: border-box; margin-bottom: 4px; }
```

- [ ] **Step 3: Verify modal HTML renders**

Temporarily call `document.getElementById('story-mgmt-overlay').classList.add('visible')` in console. Modal should appear with two tabs and empty list.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: story management modal HTML structure and CSS"
```

---

## Task 9: Story management JS — list, edit, delete

**Files:**
- Modify: `index.html` — JS admin functions block (~line 1517)

- [ ] **Step 1: Replace `openAddStory`, `closeAddStory`, `submitNewStory` with management functions**

Remove those three functions entirely. Add:

```js
let editingStoryId = null;

function openStoryManagement() {
    populatePersonFilter();
    renderStoryList();
    switchMgmtTab('stories');
    showListView();
    document.getElementById('story-mgmt-overlay').classList.add('visible');
    applyLang(currentLang);
}

function closeStoryManagement() {
    document.getElementById('story-mgmt-overlay').classList.remove('visible');
    editingStoryId = null;
}

function switchMgmtTab(tab) {
    document.getElementById('mgmt-stories-pane').style.display = tab === 'stories' ? '' : 'none';
    document.getElementById('mgmt-bulk-pane').style.display    = tab === 'bulk'    ? '' : 'none';
    document.getElementById('tab-stories').classList.toggle('active', tab === 'stories');
    document.getElementById('tab-bulk').classList.toggle('active', tab === 'bulk');
}

function populatePersonFilter() {
    const sel = document.getElementById('mgmt-person-filter');
    const unique = [...new Set(stories.map(s => s.person))].sort();
    sel.innerHTML = `<option value="">${t('filterAll')}</option>` +
        unique.map(p => `<option value="${p}">${p}</option>`).join('');
}

function renderStoryList() {
    const query  = (document.getElementById('mgmt-search')?.value || '').toLowerCase();
    const person = document.getElementById('mgmt-person-filter')?.value || '';
    const list   = document.getElementById('mgmt-story-list');
    const count  = document.getElementById('mgmt-count');

    const filtered = stories.filter(s =>
        (!person || s.person === person) &&
        (!query  || s.title.toLowerCase().includes(query) || s.person.toLowerCase().includes(query))
    );

    list.innerHTML = filtered.length === 0
        ? `<div style="padding:12px;text-align:center;color:#bbb;font-size:11px">—</div>`
        : filtered.map(s => `
            <div class="mgmt-story-row">
                <div class="mgmt-story-info">
                    <strong>${s.title}${s.isGhost ? ' 👻' : ''}</strong>
                    <span>· ${s.person}</span>
                </div>
                <div class="mgmt-story-actions">
                    <button class="mgmt-btn-edit" onclick="openEditView(${s.id})">${t('editBtn')}</button>
                    <button class="mgmt-btn-del"  onclick="deleteStory(${s.id})">${t('deleteBtn')}</button>
                </div>
            </div>`
        ).join('');

    count.textContent = t('storyCount')(filtered.length);
}

function showListView() {
    document.getElementById('mgmt-list-view').style.display = '';
    document.getElementById('mgmt-edit-view').style.display = 'none';
    editingStoryId = null;
}

function openEditView(id) {
    const story = stories.find(s => s.id === id);
    if (!story) return;
    editingStoryId = id;

    document.getElementById('edit-person').value    = story.person;
    document.getElementById('edit-title').value     = story.title;
    document.getElementById('edit-narrative').value = story.narrative;
    document.getElementById('edit-period').value    = story.period || '';
    document.getElementById('edit-lat').value       = story.lat;
    document.getElementById('edit-lng').value       = story.lng;
    document.getElementById('edit-ghost').checked   = !!story.isGhost;
    document.getElementById('edit-video').value     = story.videoUrl || '';
    document.getElementById('edit-error').style.display = 'none';

    // Build emotion checkboxes
    const wrap = document.getElementById('edit-emotion-checkboxes');
    wrap.innerHTML = '';
    Object.entries(emotionData).forEach(([key, {hex, label}]) => {
        const cb  = document.createElement('input');
        cb.type   = 'checkbox';
        cb.value  = key;
        cb.id     = `edit-em-${key}`;
        cb.checked = story.emotions.includes(key);
        const lbl = document.createElement('label');
        lbl.htmlFor = cb.id;
        lbl.textContent = ' ' + label;
        lbl.className = 'emotion-check-label' + (cb.checked ? ' checked' : '');
        if (cb.checked) { lbl.style.backgroundColor = hex; lbl.style.borderColor = hex; }
        cb.addEventListener('change', () => {
            lbl.classList.toggle('checked', cb.checked);
            lbl.style.backgroundColor = cb.checked ? hex : '';
            lbl.style.borderColor     = cb.checked ? hex : '';
        });
        wrap.appendChild(cb);
        wrap.appendChild(lbl);
    });

    document.getElementById('mgmt-list-view').style.display = 'none';
    document.getElementById('mgmt-edit-view').style.display = '';
}

function saveEditedStory() {
    const person    = document.getElementById('edit-person').value.trim();
    const title     = document.getElementById('edit-title').value.trim();
    const narrative = document.getElementById('edit-narrative').value.trim();
    const period    = document.getElementById('edit-period').value.trim();
    const lat       = parseFloat(document.getElementById('edit-lat').value);
    const lng       = parseFloat(document.getElementById('edit-lng').value);
    const isGhost   = document.getElementById('edit-ghost').checked;
    const videoUrl  = document.getElementById('edit-video').value.trim();
    const emotions  = [...document.querySelectorAll('#edit-emotion-checkboxes input:checked')]
                        .map(i => i.value);

    if (!person || !title || !narrative || isNaN(lat) || isNaN(lng) || emotions.length === 0) {
        document.getElementById('edit-error').style.display = 'block';
        return;
    }

    const idx = stories.findIndex(s => s.id === editingStoryId);
    if (idx === -1) return;

    stories[idx] = { ...stories[idx], person, title, narrative, period, lat, lng, isGhost, emotions, videoUrl };
    saveStories();

    // Rebuild the map marker
    if (markerObjects[editingStoryId]) {
        markerObjects[editingStoryId].remove();
        delete markerObjects[editingStoryId];
        delete markerElements[editingStoryId];
    }
    addMarkerForStory(stories[idx]);

    // Update people list if new person name
    if (!people.includes(person)) people.push(person);
    buildPeopleFilter();
    updateMarkerVisibility();
    showListView();
    renderStoryList();
}

function deleteStory(id) {
    if (!confirm('Bu tanıklığı silmek istediğinize emin misiniz?')) return;
    const idx = stories.findIndex(s => s.id === id);
    if (idx === -1) return;
    stories.splice(idx, 1);
    saveStories();
    if (markerObjects[id]) {
        markerObjects[id].remove();
        delete markerObjects[id];
        delete markerElements[id];
    }
    renderStoryList();
    updateStats();
}
```

- [ ] **Step 2: Update `deleteStoredStory` reference**

Remove the old `deleteStoredStory` function — it is replaced by `deleteStory` above.

- [ ] **Step 3: Verify list, edit, and delete**

1. Log in as admin → modal opens showing all 20 stories.
2. Filter by person "Mikail" → shows only Mikail's stories.
3. Search "Kilise" → shows matching story.
4. Click Düzenle on a story → edit form appears pre-filled.
5. Change the title, click Save → story updates on map and in list.
6. Click × Sil → story removed from list and map.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: story list, inline edit, and delete in management modal"
```

---

## Task 10: Bulk add JS

**Files:**
- Modify: `index.html` — JS admin block

- [ ] **Step 1: Add bulk add functions**

```js
function addBulkCard() {
    const cards = document.getElementById('bulk-cards');
    const idx   = cards.children.length + 1;
    const card  = document.createElement('div');
    card.className = 'bulk-card';
    card.dataset.cardIdx = idx;

    // Build emotion pills for this card
    const emotionPills = Object.entries(emotionData).map(([key, {hex, label}]) =>
        `<input type="checkbox" id="bc-em-${idx}-${key}" value="${key}"
            style="display:none"
            onchange="bcEmotionToggle(this,'${hex}')">
         <label for="bc-em-${idx}-${key}"
            class="emotion-check-label"
            style="font-size:10px;padding:2px 8px;">${label}</label>`
    ).join('');

    card.innerHTML = `
        <button class="bulk-remove-btn" onclick="this.closest('.bulk-card').remove(); updateBulkSaveBtn()">×</button>
        <div class="bulk-card-num">Tanıklık ${idx}</div>

        <span class="modal-field-label" style="margin-top:0" data-i18n="titleLabel">Başlık *</span>
        <input type="text" class="modal-input bc-title" placeholder="">

        <div style="display:grid;grid-template-columns:1fr 1fr;gap:6px">
            <div>
                <span class="modal-field-label" data-i18n="latLabel">Enlem *</span>
                <input type="number" step="0.00001" class="modal-input bc-lat" placeholder="41.028">
            </div>
            <div>
                <span class="modal-field-label" data-i18n="lngLabel">Boylam *</span>
                <input type="number" step="0.00001" class="modal-input bc-lng" placeholder="29.033">
            </div>
        </div>

        <span class="modal-field-label" data-i18n="narrativeLabel">Anlatı *</span>
        <textarea class="modal-textarea bc-narrative" style="height:60px"></textarea>

        <span class="modal-field-label" data-i18n="periodLabel">Dönem (isteğe bağlı)</span>
        <input type="text" class="modal-input bc-period" placeholder="">

        <div class="inline-check">
            <input type="checkbox" class="bc-ghost" id="bc-ghost-${idx}">
            <label for="bc-ghost-${idx}" data-i18n="ghostLabel">Mevcut değil</label>
        </div>

        <span class="modal-field-label" data-i18n="emotionLabel">Duygular *</span>
        <div class="emotion-check-grid bc-emotions">${emotionPills}</div>

        <div class="modal-error bc-error" style="display:none" data-i18n="errorRequired">Zorunlu alanları doldurun.</div>
    `;
    cards.appendChild(card);
    applyLang(currentLang); // translate new card's data-i18n nodes
    updateBulkSaveBtn();
}

function bcEmotionToggle(cb, hex) {
    const lbl = cb.nextElementSibling;
    lbl.classList.toggle('checked', cb.checked);
    lbl.style.backgroundColor = cb.checked ? hex : '';
    lbl.style.borderColor     = cb.checked ? hex : '';
}

function updateBulkSaveBtn() {
    const n   = document.getElementById('bulk-cards').children.length;
    const btn = document.getElementById('bulk-save-btn');
    btn.textContent = t('saveAllBtn')(n);
    btn.disabled = n === 0;
}

function saveBulkStories() {
    const person = document.getElementById('bulk-person').value.trim();
    const cards  = [...document.getElementById('bulk-cards').querySelectorAll('.bulk-card')];
    let allValid = true;

    const toSave = cards.map(card => {
        const title     = card.querySelector('.bc-title').value.trim();
        const lat       = parseFloat(card.querySelector('.bc-lat').value);
        const lng       = parseFloat(card.querySelector('.bc-lng').value);
        const narrative = card.querySelector('.bc-narrative').value.trim();
        const period    = card.querySelector('.bc-period').value.trim();
        const isGhost   = card.querySelector('.bc-ghost').checked;
        const emotions  = [...card.querySelectorAll('.bc-emotions input:checked')].map(i => i.value);
        const errEl     = card.querySelector('.bc-error');
        const valid     = !!title && !isNaN(lat) && !isNaN(lng) && !!narrative && emotions.length > 0;
        errEl.style.display = valid ? 'none' : 'block';
        if (!valid) allValid = false;
        return valid ? { id: Date.now() + Math.random(), person: person || 'Unknown',
                         title, lat, lng, narrative, period, isGhost, emotions } : null;
    }).filter(Boolean);

    if (toSave.length === 0) return;

    toSave.forEach(s => {
        stories.push(s);
        addMarkerForStory(s);
        if (!people.includes(s.person)) people.push(s.person);
    });
    saveStories();
    buildPeopleFilter();
    updateMarkerVisibility();
    updateStats();

    // Clear bulk pane
    document.getElementById('bulk-cards').innerHTML = '';
    document.getElementById('bulk-person').value = '';
    updateBulkSaveBtn();

    if (!allValid) alert('Some cards had errors and were skipped. Valid stories were saved.');
}
```

- [ ] **Step 2: Initialise bulk pane with one empty card when tab is switched**

In `switchMgmtTab`, when switching to `'bulk'`, add a card if the container is empty:

```js
function switchMgmtTab(tab) {
    document.getElementById('mgmt-stories-pane').style.display = tab === 'stories' ? '' : 'none';
    document.getElementById('mgmt-bulk-pane').style.display    = tab === 'bulk'    ? '' : 'none';
    document.getElementById('tab-stories').classList.toggle('active', tab === 'stories');
    document.getElementById('tab-bulk').classList.toggle('active', tab === 'bulk');
    if (tab === 'bulk' && document.getElementById('bulk-cards').children.length === 0) {
        addBulkCard();
    }
}
```

- [ ] **Step 3: Verify bulk add**

1. Switch to "Toplu Ekle" tab → one empty card appears.
2. Set person "5". Fill in card 1: title, lat, lng, narrative, one emotion.
3. Click "+ Tanıklık ekle" → second card appears.
4. Leave card 2 blank, click "Tümünü Kaydet (2 tanıklık)".
5. Card 2 shows error. Card 1 saves — new marker appears on map.
6. `JSON.parse(localStorage.getItem('uskudar-stories')).slice(-1)` shows the new story.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: bulk add stories with stacked cards and shared person field"
```

---

## Task 11: Wire up `submitAdminLogin` + remove old modal references

**Files:**
- Modify: `index.html` — `submitAdminLogin` function; any HTML referencing old modal

- [ ] **Step 1: Update `submitAdminLogin`**

```js
function submitAdminLogin() {
    const u = document.getElementById('admin-username').value.trim();
    const p = document.getElementById('admin-password').value;
    if (u === ADMIN_CREDS.username && p === ADMIN_CREDS.password) {
        closeAdminLogin();
        seedStoriesFromHardcoded(); // seeds and reloads on first run; no-op after
        openStoryManagement();
    } else {
        document.getElementById('admin-error').style.display = 'block';
    }
}
```

Note: `seedStoriesFromHardcoded()` calls `location.reload()` only on first run. On subsequent runs it returns immediately, so `openStoryManagement()` is reached.

- [ ] **Step 2: Remove any HTML buttons or links still referencing `openAddStory`**

Search the HTML for `openAddStory` and replace with `openStoryManagement`.

- [ ] **Step 3: Full end-to-end test**

1. `localStorage.clear()`, reload.
2. Click admin button → login modal.
3. Login → page reloads (seed). Login again → management modal opens.
4. 20 stories listed. Edit one, save, marker updates. Delete one, gone.
5. Bulk add 2 stories for person "5" → both appear on map.
6. Switch language to EN → all modal strings in English.
7. Switch to ՀՅ → `[ՀՅ]` placeholders visible, ghost label shows "Այլեւս չկայ".

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: wire admin login to story management, remove old add-story modal"
```

---

## Task 12: Ghost marker filter toggle (visitor-facing)

The ghost layer is a separate visual concern — visitors should be able to toggle it.

**Files:**
- Modify: `index.html` — left panel HTML (filter section); JS filter logic

- [ ] **Step 1: Add a ghost toggle below the emotion filter section**

```html
<!-- Ghost layer toggle -->
<div class="filter-section" style="margin-top:8px">
    <div class="inline-check">
        <input type="checkbox" id="ghost-toggle" checked onchange="updateMarkerVisibility()">
        <label for="ghost-toggle" data-i18n="legendGhost">Artık mevcut değil</label>
    </div>
</div>
```

- [ ] **Step 2: Update `updateMarkerVisibility` to respect the ghost toggle**

Find `updateMarkerVisibility` (~line 1175). Add to the visibility condition:

```js
const showGhosts = document.getElementById('ghost-toggle')?.checked ?? true;

const isVisible =
    activePeople.has(story.person) &&
    story.emotions.some(e => activeEmotions.has(e)) &&
    storyMatchesSearch(story) &&
    (showGhosts || !story.isGhost);
```

- [ ] **Step 3: Verify**

Check the ghost toggle → ghost markers disappear. Uncheck → they reappear. Normal markers unaffected.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: ghost layer toggle in visitor filter panel"
```

---

## Task 13: Final cleanup + push

- [ ] **Step 1: Remove the old `sensory` field references throughout**

Search for `sensory` in the JS. Remove any reference in popup HTML (already done in Task 6). Check `submitNewStory` was fully removed (Task 11). Check `closeAddStory` was removed.

- [ ] **Step 2: Smoke test all features together**

1. Load fresh page (clear localStorage).
2. Language switches correctly across all three.
3. Admin login → seed → management modal.
4. Edit, delete, bulk add all work.
5. Ghost checkbox on a story → marker becomes breathing dots.
6. Ghost toggle in panel hides/shows ghost markers.
7. Period field appears in popup for stories that have it.
8. Map legend visible bottom-left.

- [ ] **Step 3: Push to GitHub**

```bash
git push
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Seed migration on first login | Task 4 |
| `period` + `isGhost` schema fields | Task 4 |
| Admin modal: two tabs | Task 8 |
| Search + person filter | Task 9 |
| Inline edit (replace in place) | Task 9 |
| Delete story | Task 9 |
| Bulk add with shared person | Task 10 |
| Ghost marker — breathing alpha | Task 5 |
| Ghost marker — desaturated hue | Task 5 |
| Ghost checkbox in forms | Tasks 8, 10 |
| Period in public popup | Task 6 |
| Sensory removed | Task 6 |
| i18n data structure + applyLang | Task 1 |
| data-i18n on existing elements | Task 2 |
| Language switcher pill | Task 3 |
| ՀՅ placeholders except provided strings | Task 1 |
| Map legend | Task 7 |
| Ghost layer visitor toggle | Task 12 |

**Placeholder scan:** No TBD, TODO, or "implement later" present.

**Type consistency:** `saveStories()`, `getAllStories()`, `stories` array, `emotionData`, `markerObjects`, `markerElements`, `people` — all consistent with existing codebase naming.
