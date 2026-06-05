# Entrance Affordance Layer & Sidebar Standing Key — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a once-per-device cinematic intro title page and turn the always-open left sidebar into the standing classification key (with a quiet marker-state caption), per the approved spec.

**Architecture:** All changes are additive edits to the single `index.html`. A `#intro-overlay` is shown on first visit (gated by a `localStorage` flag) and fades out on dismiss. The sidebar's default state flips from collapsed to open. A new `.marker-key` caption and three i18n keys (`introHeading`, `introBody`, `markerKey`) are added; all copy is placeholder (`DTES544` / lorem ipsum) wired via `data-i18n` for later replacement.

**Tech Stack:** Vanilla HTML/CSS/JS, Mapbox GL JS 2.15, localStorage. No build step, no test runner.

**Spec:** `docs/superpowers/specs/2026-06-05-entrance-intro-sidebar-key-design.md`

---

## Notes for the implementer

- **No git:** this directory is not a git repository. The `git commit` steps are **optional** — skip them, or run `git init` first if you want history. Do not let a missing git stop you.
- **No automated tests:** verification is done in a browser. Serve the folder over `http://localhost` (e.g. `python3 -m http.server` in the project root) and open `index.html` — do **not** use `file://` (the admin login's `crypto.subtle` and some features need a secure context).
- **Anchored edits:** locate each change by the quoted surrounding text, not by line number. The file is ~2380 lines and one big HTML document.
- **Reset between checks:** the intro shows only once per device. To re-test it, run `localStorage.removeItem('uskudar-intro-seen')` in DevTools and reload.

---

## File map

| Section of `index.html` | What changes |
|---|---|
| CSS block (before `</style>`) | Add `#intro-overlay` styles + `.marker-key` caption styles |
| `#sidebar-toggle` button | Remove `class="collapsed"` so the ☰ is hidden while the sidebar is open |
| `.controls` panel | Change `class="controls hidden"` → `class="controls"` (open by default) |
| Sidebar body (after `.stats`) | Add the `.marker-key` caption line |
| After the `#map` block | Add the `#intro-overlay` markup |
| `i18n` object (`tr`/`en`/`hy`) | Add `introHeading`, `introBody`, `markerKey` |
| Bottom of `<script>` | Add the `initIntro()` IIFE after `applyLang(currentLang)` |

---

## Task 1: Add i18n keys

**Files:**
- Modify: `index.html` — the `i18n` object (`tr`, `en`, `hy` blocks)

- [ ] **Step 1: Add keys to the `tr` block**

Find this exact text (end of the `tr` block):

```js
            adminLoginBtn:     'Giriş',
        },
        en: {
```

Replace with:

```js
            adminLoginBtn:     'Giriş',
            introHeading:      'DTES544',
            introBody:         'Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed diam nonummy nibh euismod tincidunt ut laoreet dolore magna aliquam erat volutpat. Ut wisi enim ad minim veniam, quis nostrud exerci tation ullamcorper. Duis autem vel eum iriure dolor in hendrerit.',
            markerKey:         '● DTES544 · ◌ DTES544 · ∿ DTES544',
        },
        en: {
```

- [ ] **Step 2: Add keys to the `en` block**

Find this exact text (end of the `en` block):

```js
            adminLoginBtn:     'Login',
        },
        hy: {
```

Replace with:

```js
            adminLoginBtn:     'Login',
            introHeading:      'DTES544',
            introBody:         'Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed diam nonummy nibh euismod tincidunt ut laoreet dolore magna aliquam erat volutpat. Ut wisi enim ad minim veniam, quis nostrud exerci tation ullamcorper. Duis autem vel eum iriure dolor in hendrerit.',
            markerKey:         '● DTES544 · ◌ DTES544 · ∿ DTES544',
        },
        hy: {
```

- [ ] **Step 3: Add keys to the `hy` block**

Find this exact text (end of the `hy` block):

```js
            adminLoginBtn:     '[ՀՅ]',
        },
    };
```

Replace with:

```js
            adminLoginBtn:     '[ՀՅ]',
            introHeading:      'DTES544',
            introBody:         '[ՀՅ]',
            markerKey:         '[ՀՅ]',
        },
    };
```

- [ ] **Step 4: Verify the keys load**

Reload the page over `http://localhost`. In DevTools console:

```js
t('introHeading')  // → "DTES544"
t('markerKey')     // → "● DTES544 · ◌ DTES544 · ∿ DTES544"
```

Expected: those strings, no errors.

- [ ] **Step 5 (optional): Commit**

```bash
git add index.html
git commit -m "feat: add intro/markerKey i18n keys (placeholder copy)"
```

---

## Task 2: Sidebar open by default + marker-state caption

**Files:**
- Modify: `index.html` — `#sidebar-toggle` button, `.controls` panel, sidebar body, CSS block

- [ ] **Step 1: Hide the ☰ toggle by default (sidebar will be open)**

Find:

```html
<button id="sidebar-toggle" class="collapsed" onclick="toggleSidebar()">☰</button>
```

Replace with:

```html
<button id="sidebar-toggle" onclick="toggleSidebar()">☰</button>
```

(The CSS rule `#sidebar-toggle { display: none; }` already hides it; only `.collapsed` shows it. Removing the class means it stays hidden until the user collapses the panel.)

- [ ] **Step 2: Make the sidebar open by default**

Find:

```html
<!-- Left control panel -->
<div class="controls hidden">
```

Replace with:

```html
<!-- Left control panel -->
<div class="controls">
```

- [ ] **Step 3: Add the marker-state caption in the sidebar body**

Find this exact text (the stats block and the two closing divs that follow it):

```html
        <!-- STATS -->
        <div class="stats">
            <span class="stats-count" id="storyCount">—</span>
            <span class="stats-label">Görünen Anı</span>
            <div class="stats-detail" id="activeFilters"></div>
        </div>
    </div>
</div>
```

Replace with:

```html
        <!-- STATS -->
        <div class="stats">
            <span class="stats-count" id="storyCount">—</span>
            <span class="stats-label">Görünen Anı</span>
            <div class="stats-detail" id="activeFilters"></div>
        </div>

        <!-- Marker-state key (glyphs functional, labels placeholder) -->
        <div class="marker-key" data-i18n="markerKey">● DTES544 · ◌ DTES544 · ∿ DTES544</div>
    </div>
</div>
```

- [ ] **Step 4: Add the `.marker-key` CSS**

Find this exact text (the last CSS rule before the closing style tag):

```css
        .modal-textarea { width: 100%; border: 1px solid #e5e5e5; border-radius: 6px; padding: 7px 10px; font-size: 12px; height: 80px; resize: vertical; font-family: inherit; box-sizing: border-box; margin-bottom: 4px; }
    </style>
```

Replace with:

```css
        .modal-textarea { width: 100%; border: 1px solid #e5e5e5; border-radius: 6px; padding: 7px 10px; font-size: 12px; height: 80px; resize: vertical; font-family: inherit; box-sizing: border-box; margin-bottom: 4px; }

        /* ── Marker-state key ───────────────────── */
        .marker-key {
            margin-top: 10px;
            font-size: 11px;
            color: #aaa;
            text-align: center;
            letter-spacing: 0.2px;
        }
    </style>
```

- [ ] **Step 5: Verify**

Reload over `http://localhost`. Expected:
- The left sidebar is **open** on load (not collapsed), the ☰ button is not visible.
- A quiet line `● DTES544 · ◌ DTES544 · ∿ DTES544` sits at the bottom of the sidebar body.
- Clicking the ✕ collapses the sidebar and the ☰ appears; clicking ☰ reopens it (existing `toggleSidebar()` still works).
- Switch language to EN/HY: the caption updates via `data-i18n` (HY shows `[ՀՅ]`).

- [ ] **Step 6 (optional): Commit**

```bash
git add index.html
git commit -m "feat: open sidebar by default and add marker-state caption"
```

---

## Task 3: Intro overlay markup + CSS

**Files:**
- Modify: `index.html` — after the `#map` block; CSS block

- [ ] **Step 1: Add the overlay HTML after the map**

Find this exact text (the end of the `#map` div and the comment that follows):

```html
<div id="lang-switcher" role="group" aria-label="Language">
    <button class="lang-option active" data-lang="tr" onclick="applyLang('tr')">TR</button>
    <button class="lang-option" data-lang="en" onclick="applyLang('en')">EN</button>
    <button class="lang-option" data-lang="hy" onclick="applyLang('hy')">ՀՅ</button>
</div>
</div>

<!-- Floating video panel -->
```

Replace with:

```html
<div id="lang-switcher" role="group" aria-label="Language">
    <button class="lang-option active" data-lang="tr" onclick="applyLang('tr')">TR</button>
    <button class="lang-option" data-lang="en" onclick="applyLang('en')">EN</button>
    <button class="lang-option" data-lang="hy" onclick="applyLang('hy')">ՀՅ</button>
</div>
</div>

<!-- Entrance intro overlay (shown once per device) -->
<div id="intro-overlay" aria-hidden="true">
    <div id="intro-inner">
        <h1 id="intro-heading" data-i18n="introHeading">DTES544</h1>
        <p id="intro-body" data-i18n="introBody">Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed diam nonummy nibh euismod tincidunt ut laoreet dolore magna aliquam erat volutpat. Ut wisi enim ad minim veniam, quis nostrud exerci tation ullamcorper. Duis autem vel eum iriure dolor in hendrerit.</p>
        <div id="intro-cue" aria-hidden="true">↓</div>
    </div>
</div>

<!-- Floating video panel -->
```

- [ ] **Step 2: Add the overlay CSS**

Find this exact text (the `.marker-key` rule added in Task 2, before the closing style tag):

```css
        /* ── Marker-state key ───────────────────── */
        .marker-key {
            margin-top: 10px;
            font-size: 11px;
            color: #aaa;
            text-align: center;
            letter-spacing: 0.2px;
        }
    </style>
```

Replace with:

```css
        /* ── Marker-state key ───────────────────── */
        .marker-key {
            margin-top: 10px;
            font-size: 11px;
            color: #aaa;
            text-align: center;
            letter-spacing: 0.2px;
        }

        /* ── Entrance intro overlay ─────────────── */
        #intro-overlay {
            position: fixed;
            inset: 0;
            z-index: 10000;
            display: none;
            align-items: center;
            justify-content: center;
            text-align: center;
            padding: 24px;
            background: rgba(245,245,245,0.55);
            backdrop-filter: blur(6px);
            -webkit-backdrop-filter: blur(6px);
            cursor: pointer;
            transition: opacity 0.8s ease;
        }
        #intro-overlay.visible { display: flex; }
        #intro-overlay.dismissing { opacity: 0; }
        #intro-inner { max-width: 560px; }
        #intro-heading {
            font-family: Georgia, 'Times New Roman', serif;
            font-size: 40px;
            font-weight: 600;
            color: #1a1a1a;
            letter-spacing: 0.5px;
        }
        #intro-body {
            margin-top: 18px;
            font-size: 15px;
            line-height: 1.8;
            color: #555;
        }
        #intro-cue {
            margin-top: 28px;
            font-size: 18px;
            color: #aaa;
            letter-spacing: 2px;
            animation: introCueBob 2.2s ease-in-out infinite;
        }
        @keyframes introCueBob {
            0%, 100% { transform: translateY(0);   opacity: 0.5; }
            50%      { transform: translateY(4px); opacity: 1;   }
        }
        @media (prefers-reduced-motion: reduce) {
            #intro-overlay { transition: none; }
            #intro-cue { animation: none; }
        }
    </style>
```

- [ ] **Step 3: Verify the overlay renders when forced visible**

Reload over `http://localhost`. The overlay should be hidden by default (no JS yet). In DevTools console, force it on:

```js
document.getElementById('intro-overlay').classList.add('visible');
```

Expected: a centered title page — `DTES544` heading, lorem ipsum paragraph, a bobbing `↓` — over a blurred map. Then hide it again:

```js
document.getElementById('intro-overlay').classList.remove('visible');
```

- [ ] **Step 4 (optional): Commit**

```bash
git add index.html
git commit -m "feat: add entrance intro overlay markup and styles"
```

---

## Task 4: Intro show/dismiss logic

**Files:**
- Modify: `index.html` — bottom of the `<script>` block

- [ ] **Step 1: Add the `initIntro` IIFE after `applyLang(currentLang)`**

Find this exact text (the very end of the script):

```js
    // Apply saved language on load
    applyLang(currentLang);
</script>
```

Replace with:

```js
    // Apply saved language on load
    applyLang(currentLang);

    // ── Entrance intro (shown once per device) ──────────────
    (function initIntro() {
        const overlay = document.getElementById('intro-overlay');
        if (!overlay) return;
        if (localStorage.getItem('uskudar-intro-seen')) return; // already seen — skip

        overlay.classList.add('visible');
        overlay.setAttribute('aria-hidden', 'false');

        let dismissed = false;
        function dismissIntro() {
            if (dismissed) return;
            dismissed = true;
            localStorage.setItem('uskudar-intro-seen', '1');

            const reduce = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
            const remove = () => {
                overlay.classList.remove('visible', 'dismissing');
                overlay.setAttribute('aria-hidden', 'true');
                overlay.removeEventListener('transitionend', remove);
            };
            if (reduce) {
                remove();
            } else {
                overlay.classList.add('dismissing');   // opacity → 0 via transition
                overlay.addEventListener('transitionend', remove);
                setTimeout(remove, 1000);              // fallback if transitionend doesn't fire
            }
        }

        overlay.addEventListener('click', dismissIntro);
        document.addEventListener('keydown', e => {
            if (e.key === 'Escape' && overlay.classList.contains('visible')) dismissIntro();
        });
    })();
</script>
```

- [ ] **Step 2: Verify first-visit behavior**

In DevTools console, clear the flag and reload:

```js
localStorage.removeItem('uskudar-intro-seen'); location.reload();
```

Expected: the intro title page appears on load over a blurred map, sidebar open behind it.

- [ ] **Step 3: Verify dismiss + fade + persistence**

- Click anywhere on the overlay (or press `Esc`). Expected: the text/overlay fades out (~0.8s) and is removed, revealing the live map with the open sidebar.
- In console: `localStorage.getItem('uskudar-intro-seen')` → `"1"`.
- Reload the page. Expected: **no** intro this time — straight to the map.

- [ ] **Step 4: Verify reduced-motion path**

In DevTools, emulate reduced motion (Rendering tab → "Emulate CSS prefers-reduced-motion: reduce"), then:

```js
localStorage.removeItem('uskudar-intro-seen'); location.reload();
```

Click to dismiss. Expected: the overlay disappears immediately with no fade, and the flag is still set.

- [ ] **Step 5 (optional): Commit**

```bash
git add index.html
git commit -m "feat: show intro once per device with fade dismiss + reduced-motion"
```

---

## Task 5: Full smoke test

**Files:** none (verification only)

- [ ] **Step 1: Syntax check the inline script**

Run from the project root:

```bash
node -e '
const fs=require("fs"),vm=require("vm");
const html=fs.readFileSync("index.html","utf8");
const re=/<script(?![^>]*\bsrc=)[^>]*>([\s\S]*?)<\/script>/g;let m,n=0;
while((m=re.exec(html))){const b=m[1];if(!b.trim())continue;n++;try{new vm.Script(b,{filename:`inline-${n}.js`});}catch(e){console.error("FAIL",e.message);process.exit(1);}}
console.log(`OK: ${n} inline script(s) parsed cleanly.`);
'
```

Expected: `OK: 1 inline script(s) parsed cleanly.`

- [ ] **Step 2: End-to-end in the browser** (served over `http://localhost`)

1. `localStorage.clear()` in console, reload → intro title page shows over open sidebar.
2. Dismiss (click / `Esc`) → fades out, map interactive, sidebar open, caption visible at the bottom.
3. Reload → no intro (flag set).
4. Collapse sidebar with ✕ → ☰ appears; reopen with ☰.
5. Switch TR → EN → ՀՅ: sidebar caption and (after a flag reset) the intro heading/body update via `data-i18n`; HY shows `[ՀՅ]` for the body and caption.
6. Confirm existing features still work: clicking a marker opens its popup; the "Watch" button plays Mikail's video; admin login (Ctrl+Shift+A) still opens.

- [ ] **Step 3 (optional): Final commit**

```bash
git add index.html docs/superpowers/
git commit -m "feat: entrance intro + sidebar standing key"
```

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| Intro once ever per device (`uskudar-intro-seen`) | Task 4 |
| Cinematic overlay (heading + body, blurred map, no chrome) | Task 3 |
| Dismiss via click / cue / Esc, text fades out | Task 4 |
| `prefers-reduced-motion` skips the fade | Tasks 3 (CSS) + 4 (JS) |
| Sidebar open by default | Task 2 |
| ☰ hidden while open, existing toggle preserved | Task 2 |
| Quiet marker-state caption (glyphs + placeholder labels) | Task 2 |
| i18n keys `introHeading` / `introBody` / `markerKey`, wired via `data-i18n` | Task 1 + markup in Tasks 2–3 |
| HY stays `[ՀՅ]` for body + caption | Task 1 |
| Placeholder copy (`DTES544` / lorem ipsum) | Tasks 1–3 |
| People labels unchanged (`1–4` + Mikail) | (no task — untouched, as required) |

All spec sections map to a task. No persistent map legend, no re-open affordance, no people relabel — consistent with "Out of Scope."

**Placeholder scan:** The only `DTES544`/lorem ipsum strings are the intentional content placeholders the spec calls for; there are no `TBD`/`TODO`/"handle later" gaps in the steps themselves.

**Type/name consistency:** `#intro-overlay`, `#intro-heading`, `#intro-body`, `#intro-cue`, `.marker-key`, classes `visible`/`dismissing`, flag `uskudar-intro-seen`, keys `introHeading`/`introBody`/`markerKey` — used identically across CSS, HTML, JS, and i18n in every task.
