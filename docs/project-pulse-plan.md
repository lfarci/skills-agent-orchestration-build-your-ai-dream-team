# Project Pulse — Implementation Plan

## 1. Overview

**Project Pulse** is a small, static, client-side dashboard web app that visualizes a portfolio of projects at a glance. It renders a responsive grid of project cards showing:

- Project name
- Status (`on-track`, `at-risk`, `blocked`, `done`) with a color-coded badge
- Owner
- Progress percentage (visualized as a bar)
- Last-updated date

**Goals:**

- Zero backend, zero build tooling — plain HTML, CSS, and vanilla JS.
- Data-driven: cards are rendered from a local `project-data.json` file via `fetch`.
- Debuggable directly inside VS Code via a `.vscode/launch.json` configuration.
- Accessible, responsive, and visually polished, using deterministic CSS hooks so markup, styling, and rendering logic can be developed in parallel.

**Non-goals (initial version):**

- No authentication, persistence, or write operations.
- No framework (React/Vue/etc.), no bundler, no package.json.
- No automated test suite (validation is manual + deterministic-by-construction).

---

## 2. File Assignments

The app lives in `app/`. A small JS file is introduced (`app/app.js`) rather than an inline script, so the Coder and Designer can own cleanly separated files with no merge overlap.

| File | Primary Owner | Secondary / Review | Purpose |
|---|---|---|---|
| `app/index.html` | **Designer** | Coder (reviews script tag + data-* hooks) | Semantic markup skeleton, header, empty `.dashboard` container, `<template id="project-card-template">` for a single card, `<script src="app.js" defer>` include, accessible landmarks. |
| `app/styles.css` | **Designer** | — | Visual design, responsive grid layout for `.dashboard`, `.project-card` styling, `.status-badge` variants (`.status-badge--on-track`, `--at-risk`, `--blocked`, `--done`), `.progress-bar` styling, typography, focus states, reduced-motion support. |
| `app/project-data.json` | **Coder** | Designer (reviews sample content for realism) | Sample dataset conforming to an agreed schema (see §4). Must include at least one entry per status to exercise all badge variants, plus edge cases (0% progress, 100% progress, long names). |
| `app/app.js` | **Coder** | Designer (reviews class names actually assigned) | Fetches `project-data.json`, validates each record, clones the `<template>`, populates fields, applies the correct `.status-badge--<status>` modifier, appends to `.dashboard`. Handles fetch/parse errors and empty/malformed rows. |
| `.vscode/launch.json` | **Coder** | — | VS Code debug configuration to launch `app/index.html` in a browser (Chrome/Edge) with a workspace-relative URL, so `fetch` works (i.e. served over `http://`, not `file://`). See §4 for approach. |

**Explicit collaboration points:**

- The **card DOM structure** (the contents of `<template id="project-card-template">`) is authored by the Designer, but its `data-field` attributes / class hooks are consumed by the Coder in `app.js`. These must be agreed in Phase 1 (see §6) — captured as a short "DOM contract" comment inside `index.html`.
- The **JSON schema** (§4) is authored by the Coder and consumed by both `app.js` and the Designer's sample-card mockups.
- The Designer must not edit `app.js` or `project-data.json`. The Coder must not edit `styles.css` or restructure `index.html` markup — only append/adjust the `<script>` tag and (if needed) `data-*` attributes previously agreed with the Designer.

---

## 3. Designer Responsibilities

Concrete deliverables owned by the Designer:

1. **`app/index.html` markup**, including:
   - `<!doctype html>`, `<html lang="en">`, proper `<meta charset>` and `<meta name="viewport">`.
   - Semantic landmarks: `<header>` with app title + short description, `<main>` containing the dashboard.
   - `<section class="dashboard" aria-label="Projects">` as the render target (must be empty at load; populated by JS).
   - `<template id="project-card-template">` containing the canonical `.project-card` markup with the deterministic hooks below.
   - An empty-state element (`.dashboard-empty`, hidden by default) and an error-state element (`.dashboard-error`, hidden by default) that `app.js` can toggle.
   - `<script src="app.js" defer></script>` at the end of `<head>` or before `</body>`.
2. **Deterministic CSS / DOM hooks** (contract with Coder):
   - Container: `.dashboard`
   - Card root: `.project-card`
   - Card fields (each with a `data-field` for JS targeting):
     - `.project-card__name` `data-field="name"`
     - `.project-card__owner` `data-field="owner"`
     - `.project-card__updated` `data-field="lastUpdated"` (rendered inside a `<time>` element)
     - `.project-card__progress-bar` `data-field="progress"` (with an inner `.project-card__progress-fill` whose `width` is set by JS)
     - `.project-card__progress-label` `data-field="progressLabel"`
     - `.status-badge` `data-field="status"` — JS will additionally apply one of: `.status-badge--on-track`, `.status-badge--at-risk`, `.status-badge--blocked`, `.status-badge--done`.
3. **`app/styles.css`**:
   - Responsive grid on `.dashboard` using CSS Grid `auto-fill` + `minmax(260px, 1fr)`.
   - Breakpoints validated at ~360px, 768px, 1200px.
   - Distinct, accessible color treatments per status badge variant (contrast ≥ 4.5:1 for text).
   - Progress bar styling with a filled portion controlled via inline `width` set by JS.
   - Visible focus rings on any interactive element (cards are non-interactive in v1, but headings/links must still have focus styles).
   - `@media (prefers-reduced-motion: reduce)` disables non-essential transitions.
   - No dependency on external fonts or CDNs (fully offline-capable).
4. **Accessibility:**
   - Cards use `role="article"` (or a semantic `<article>` element) with an `aria-labelledby` pointing at the name.
   - Status conveyed by both color *and* text (badge text = human-readable status).
   - Dates rendered inside `<time datetime="…">`.

---

## 4. Coder Responsibilities

Concrete deliverables owned by the Coder:

1. **`app/project-data.json` — schema and sample data.**

   Top-level shape: a JSON array of project objects. Schema per project:

   | Field | Type | Required | Notes |
   |---|---|---|---|
   | `id` | string | yes | Stable unique key (used for `data-project-id` on the card). |
   | `name` | string | yes | Non-empty, trimmed. |
   | `status` | string enum | yes | One of `"on-track" \| "at-risk" \| "blocked" \| "done"`. |
   | `owner` | string | yes | Display name. |
   | `progress` | number | yes | Integer 0–100 inclusive. |
   | `lastUpdated` | string | yes | ISO 8601 date (`YYYY-MM-DD`). |

   Sample dataset must include: at least one project per status value, one project at `progress: 0`, one at `progress: 100`, and one with a deliberately long name to stress the layout.

2. **`app/app.js` — rendering logic.** Must:
   - Run after DOM is ready (script uses `defer`).
   - `fetch('./project-data.json')`, then `.json()`.
   - Validate the payload is an array; validate each entry against the schema (type checks, status enum check, progress clamped/rejected outside 0–100).
   - For each valid entry: clone `#project-card-template`, populate elements by `data-field`, set `width` on `.project-card__progress-fill`, add the correct `.status-badge--<status>` modifier class, set `data-project-id` on the card root, append to `.dashboard`.
   - Skip invalid entries but log a single grouped `console.warn` listing which entries were skipped and why (does not throw).
   - Show `.dashboard-empty` if the array is empty or all entries are invalid.
   - Show `.dashboard-error` if `fetch` fails or JSON parsing throws; keep the message generic (no stack traces in UI).
   - Deterministic ordering: render cards in the order provided by the JSON file (no implicit sorting in v1).
   - No global leaks — wrap in an IIFE or `type="module"` is acceptable if the launch config serves over HTTP.
3. **`.vscode/launch.json` — debug configuration.**
   - Use the built-in `chrome` (or `msedge`) debug type.
   - Because `fetch` on `file://` is blocked in most browsers, the recommended approach is to run a lightweight static server first. Two acceptable options — Coder picks one and documents it in a top-of-file JSON comment:
     - **Option A (preferred, zero-install):** a `preLaunchTask` in `.vscode/tasks.json` that runs `python3 -m http.server 5500 --directory app` (Python 3 is present in the devcontainer — Coder must confirm), and a `launch` config with `"url": "http://localhost:5500/index.html"` and `"webRoot": "${workspaceFolder}/app"`.
     - **Option B:** rely on the "Live Server" VS Code extension and document the requirement; launch config points at `http://127.0.0.1:5500/app/index.html`.
   - Include a second, `"compound"`-free fallback config of type `"chrome"` with `"request": "launch"` that opens the URL without a preLaunchTask, for users who start the server manually.
   - Do **not** modify the existing `.vscode/tasks.json` entry (Copilot CLI terminal task); only append a new task if Option A is chosen.

4. **Error handling & determinism guarantees:**
   - Same input JSON always produces the same DOM output (no `Date.now()`, no `Math.random()` in render path).
   - All string interpolation uses `textContent` (never `innerHTML`) to avoid XSS from data.
   - `lastUpdated` is rendered via `toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })` with a fixed locale so output is stable across machines.

---

## 5. Dependencies

Ordering and data dependencies between deliverables:

1. **JSON schema (Coder)** must be defined **before** `app.js` render logic is written and before the Designer finalizes the card template's `data-field` attributes. → Schema is the shared contract.
2. **CSS hooks / DOM contract (Designer)** must be defined **before** `app.js` render logic is written. → JS depends on class + `data-field` names being final.
3. **`app/index.html` template block (Designer)** must exist before `app/app.js` can be meaningfully tested end-to-end (though `app.js` can be drafted against the agreed contract in parallel).
4. **`app/styles.css` (Designer)** depends only on the DOM contract, not on `app.js` or the JSON file. Can proceed fully in parallel with Coder work once the contract is agreed.
5. **`.vscode/launch.json` (Coder)** is **independent** of all three other files. Can be built first, in parallel, or last — but it is required before end-to-end validation (§7).
6. **Sample `project-data.json` content (Coder)** depends only on the schema, so it can be produced immediately after the schema is agreed.

**Blocking chain (critical path):** Schema + DOM contract agreement → `app.js` → end-to-end validation.

---

## 6. Parallel Work — Phased Plan

### Phase 0 — Contract kickoff (short, sequential, Orchestrator-led)

- Orchestrator publishes:
  - The JSON schema (from §4.1) as the shared data contract.
  - The DOM/CSS hook list (from §3.2) as the shared markup contract.
- No file writes yet; this is a shared-agreement step so Phase 1 can fan out safely.

### Phase 1 — Fully parallel authoring (no file overlap)

Run all three tracks concurrently:

- **Track A (Designer):** Write `app/index.html` (skeleton + `<template>` with the agreed hooks) and `app/styles.css` (full responsive design + all four badge variants + progress bar + empty/error state styles). Uses hand-crafted mock cards in a scratch file if needed to validate visuals; the shipped `index.html` leaves `.dashboard` empty.
- **Track B (Coder — data):** Write `app/project-data.json` with the sample dataset from §4.1.
- **Track C (Coder — tooling):** Write `.vscode/launch.json` (and, if Option A, append the static-server task to `.vscode/tasks.json` without touching the existing Copilot CLI task).

No two tracks write to the same file. Safe to run in parallel.

### Phase 2 — Render logic (sequential after Phase 1)

- **Coder** writes `app/app.js`, consuming the finalized DOM contract from Track A and the schema from Track B.
- Requires Track A's `index.html` template block and Track B's `project-data.json` to exist so the Coder can smoke-test locally.

### Phase 3 — Integration & validation (sequential)

- Launch via `.vscode/launch.json`, execute the validation checklist in §7.
- Any fixes are scoped back to the original owning agent (no cross-file edits).

---

## 7. Validation Expectations

Manual QA checklist to run after Phase 3:

**Launch & load**

- [ ] `.vscode/launch.json` config launches a browser and loads `index.html` over `http://` (not `file://`).
- [ ] No errors or warnings in the browser DevTools console on initial load with the shipped `project-data.json`.
- [ ] Network tab shows a successful `200` for `project-data.json`.

**Rendering correctness**

- [ ] Number of `.project-card` elements in the DOM equals the number of valid entries in `project-data.json`.
- [ ] Card order matches JSON order (no implicit sort).
- [ ] Each field (`name`, `owner`, `lastUpdated`, `progress`) renders correct data per row.
- [ ] `lastUpdated` renders inside a `<time datetime="YYYY-MM-DD">` element.
- [ ] Progress bar fill width visually matches the `progress` value (spot-check 0, 50, 100).

**Status badges**

- [ ] Each of the four statuses appears at least once and shows a visually distinct badge.
- [ ] Badge text matches the status (color is not the only signal).
- [ ] Correct modifier class (`.status-badge--on-track` etc.) is applied on the DOM element.

**Responsive layout**

- [ ] At ~360px viewport: single-column grid, no horizontal scroll, no clipped text (long-name card wraps).
- [ ] At ~768px viewport: 2-column grid.
- [ ] At ~1200px viewport: 3+ column grid.
- [ ] Focus outlines remain visible at all breakpoints.

**Error & edge handling**

- [ ] Temporarily rename `project-data.json` → reload → `.dashboard-error` state is shown, no uncaught exception.
- [ ] Replace `project-data.json` with `[]` → `.dashboard-empty` state is shown.
- [ ] Add a malformed entry (missing `status`, or `progress: 150`) → that entry is skipped, others still render, single grouped `console.warn` describes the skip.
- [ ] Add an entry with an unknown `status` value → skipped with warning; no unstyled badge appears.

**Accessibility spot-check**

- [ ] Page has one `<h1>`; each card has an accessible name (`aria-labelledby` → project name).
- [ ] Tab order is logical (header → cards, if/when cards become interactive).
- [ ] Color contrast for badge text ≥ 4.5:1 (verify with DevTools).
- [ ] `prefers-reduced-motion` disables non-essential transitions.

**Determinism**

- [ ] Two consecutive loads produce identical DOM (compare via DevTools "Copy → Outer HTML" of `.dashboard`).

---

## 8. Open Questions & Assumptions

**Assumptions made:**

1. No backend, no database, no auth — v1 is fully static and read-only.
2. No build tooling, no package manager, no npm dependencies. Plain HTML/CSS/vanilla JS only.
3. The devcontainer includes either Python 3 (for `python3 -m http.server`) or the user is willing to install the "Live Server" VS Code extension for Option B. **Coder must confirm which is available before finalizing `launch.json`.**
4. Modern evergreen browser only (Chrome/Edge/Firefox current). No IE / legacy support.
5. English-only UI copy for v1; locale hardcoded to `en-US` for date formatting to keep output deterministic.
6. `.vscode/tasks.json`'s existing Copilot CLI task must remain untouched; if Option A is chosen, the new static-server task is *appended*, not merged into the existing task.
7. Introducing `app/app.js` as a separate file (rather than an inline `<script>`) is preferred to give the Coder a clean, non-overlapping file scope vs. the Designer's `index.html`.

**Open questions for the Orchestrator / learner to confirm before Phase 1:**

1. **Server strategy for `launch.json`:** Option A (Python http.server + tasks.json preLaunchTask) or Option B (Live Server extension)? This choice affects whether `.vscode/tasks.json` needs to be touched.
2. **Card interactivity:** Are cards clickable in v1 (e.g. linking to a project detail view), or purely informational? Plan currently assumes purely informational.
3. **Sorting/filtering controls:** Any UI controls to sort by status or filter by owner in v1? Plan currently assumes none — JSON order is display order.
4. **Branding:** Any required color palette, logo, or typography? Plan currently assumes the Designer chooses a neutral, accessible palette.
5. **`app/app.js` module type:** Ship as classic script (`<script defer>`) or `<script type="module">`? Both work over HTTP; classic is simpler, module gives cleaner scoping. Recommend classic + IIFE for v1 unless the Orchestrator prefers modules.
6. **Data volume expectations:** Plan assumes small datasets (<100 projects). No virtualization needed. Confirm.
