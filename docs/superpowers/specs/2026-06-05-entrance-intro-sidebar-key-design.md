# Design Spec: Entrance Affordance Layer & Sidebar Standing Key
Date: 2026-06-05
Project: Burada Yaşadık — Üsküdar Oral History Map

---

## 0. Origin

Advisor feedback (MAA meeting):

1. *"Interface girişine affordance katmanı eklenmeli: isim, tagline, ve 4–5 cümlelik bir açıklama. Kullanıcının arayüzün mantığını kendiliğinden anlayabileceği bir giriş."*
2. *"Sınıflandırma sistemi daha self-explanatory olmalı — kullanıcının mantığı kendi başına çözmesi beklenememeli."*

This spec addresses both with the lightest touch that still satisfies them, preserving the project's deliberate aesthetic of disappearance (fading ghost markers, drifting uncertain-location markers, particles that scatter from the cursor).

---

## 1. Design Decisions (locked in brainstorming)

| Decision | Choice |
|---|---|
| Overall stance | **Minimal / poetic.** Clarity is an on-ramp at the entrance, not a permanent overlay. Ambiguity stays intact once inside. |
| Where the classification "key" lives | **The sidebar is the standing key.** The intro explains things once; the left sidebar is the durable reference. No separate re-open affordance. |
| People labels | **Unchanged** — bare `1, 2, 3, 4, Mikail`. The intro notes interviewees are numbered for anonymity. |
| Marker-state legend | **Quiet one-line caption** in the sidebar (the three states the colour chips can't carry). |
| Entrance treatment | **B→C blend** — a cinematic fading title page as the door that settles into the open sidebar as the key. |
| Intro frequency | **Once ever per device** (localStorage flag), then never again. |
| Sidebar default | **Open by default on every load** (changed from the current collapsed default). |
| Copy | **All placeholder for now.** Heading = `DTES544`; body = lorem ipsum; caption labels = `DTES544`. Real copy is filled in later by the author. |

---

## 2. Entrance Title Page

### Trigger & persistence
- On load, read `localStorage.getItem('uskudar-intro-seen')`.
- If **absent**: show the intro overlay.
- On dismiss: set `localStorage.setItem('uskudar-intro-seen', '1')` so it never shows again on that device.
- If **present**: skip entirely — page loads straight to the map + open sidebar.

### Appearance (B — cinematic)
- Full-viewport overlay (`#intro-overlay`), `position: fixed`, above the map and sidebar (`z-index` above `.controls`, below nothing else of consequence).
- Background: the map shows through, faintly blurred/dimmed (`backdrop-filter: blur(...)` over a low-opacity scrim). No card, no chrome.
- Centered content:
  - Heading: `DTES544` (placeholder; large, can be serif to match the title-page feel).
  - Body: a short lorem ipsum paragraph (~4 sentences) beneath the heading.
  - A quiet dismiss cue (e.g. a small `↓` or "gir" hint). No tagline.

### Dismiss (C blend + disappearance gesture)
- Dismiss triggers: click anywhere on the overlay, the dismiss cue, or `Esc` (consistent with existing Esc handlers).
- On dismiss: the overlay **text fades out** (opacity transition, ~0.8s) — the explanation itself disappearing — then the overlay is removed (`display:none` / detached).
- After removal the live map is fully interactive and the sidebar is already open (see §3).
- Respect `prefers-reduced-motion`: when set, skip the fade and remove immediately.

---

## 3. Sidebar as Standing Key

- **Default state changed to open.** Today the markup is `<div class="controls hidden">` with `#sidebar-toggle class="collapsed"` (the ☰ visible). New default: `.controls` is **not** hidden and `#sidebar-toggle` is **not** collapsed (☰ hidden until the user collapses the panel).
- The existing **People chips** (`1,2,3,4,Mikail`) and **Emotion chips** (colour swatch + label) already serve as the people/emotion key — unchanged.
- The user can still collapse/expand with the existing ✕ / ☰ controls. `toggleSidebar()` is unchanged.

---

## 4. Marker-State Caption

- A single subtle line added near the bottom of the sidebar body (after the stats block, or beneath the Emotions section).
- Placeholder content: `● DTES544 · ◌ DTES544 · ∿ DTES544`.
- Glyphs are the functional, durable part (`●` present place · `◌` no longer exists · `∿` location uncertain); the `DTES544` labels are placeholders the author replaces.
- Styled muted/small to stay quiet (matches `.stats-detail` weight).
- Wired with `data-i18n="markerKey"`.

---

## 5. i18n

New keys added to the `tr`, `en`, `hy` objects, wired via `data-i18n` so real copy drops in without code changes:

| Key | tr | en | hy |
|---|---|---|---|
| `introHeading` | `DTES544` | `DTES544` | `DTES544` |
| `introBody` | lorem ipsum | lorem ipsum | `[ՀՅ]` |
| `markerKey` | `● DTES544 · ◌ DTES544 · ∿ DTES544` | `● DTES544 · ◌ DTES544 · ∿ DTES544` | `[ՀՅ]` |

- The intro overlay heading and body carry `data-i18n` attributes so `applyLang()` keeps them current (relevant once real copy exists; harmless while placeholder).
- `applyLang()` already walks `[data-i18n]` / `[data-i18n-placeholder]` — no change to its logic needed.

---

## 6. Code Touch Points (single `index.html`, additive)

| Area | Change |
|---|---|
| HTML — map area | Add `#intro-overlay` block (sibling of `#map`) with heading + body + dismiss cue. |
| HTML — sidebar | Change `class="controls hidden"` → `class="controls"`; remove `collapsed` from `#sidebar-toggle`. Add the marker-state caption line in the sidebar body. |
| CSS | `#intro-overlay` (full-screen, centered, blur/scrim, opacity transition, reduced-motion guard); caption styling. |
| JS — config | Add `introHeading`, `introBody`, `markerKey` to the `i18n` object. |
| JS — load | On load: if `uskudar-intro-seen` absent, show overlay; wire dismiss (click / cue / Esc) → fade → remove → set flag. |

---

## 7. Out of Scope

- Real copy (heading, body, caption labels) — placeholders only; author fills later.
- Western Armenian (`hy`) intro/caption strings — stay `[ՀՅ]` per existing policy.
- People relabeling — stays `1–4` + Mikail.
- Persistent on-map legend (was previously removed; not reintroduced).
- Any re-open affordance for the intro.
- Security items M1–M5 from the prior audit (parked).

---

## 8. Open Items

- [ ] Real intro heading + body copy (replaces `DTES544` / lorem ipsum).
- [ ] Real marker-state caption labels (replaces `DTES544`).
- [ ] `hy` translations for `introBody`, `markerKey`.
- [ ] Confirm exact placement of the caption (under Emotions vs under Stats) during implementation review.
