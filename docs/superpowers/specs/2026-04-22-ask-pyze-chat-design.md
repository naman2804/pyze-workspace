# Ask pyze — Conversational Analysis for the Workspace Demo

**Status:** Design approved, ready for implementation plan
**Date:** 2026-04-22
**Author:** Naman (via brainstorming session)

---

## Context

The pyze workspace demo (`index.html`, deployed at [naman2804.github.io/pyze-workspace](https://naman2804.github.io/pyze-workspace)) currently shows a static dashboard — KPI tiles, filter bar, process map, hand-authored opportunities queue, drill-downs. All content is baked in.

This spec adds the first user-interactive feature: a conversational analysis flow that lets product owners ask natural-language questions of their CDD/TM clickstream data, preview the answer, and save it to a personal workspace. The backend reuses the existing Gemini + BigQuery pipeline in `/Users/namanchetanrajdev/Documents/analysis-app/` (currently a Streamlit app).

**Why:** The demo needs to move from "here's what pyze could show" to "here's what pyze can do." A single live feature — real Gemini, real BQ — transforms the tone from mockup to prototype. It also sets up the architectural pattern (frontend + FastAPI + shared prompt context) that future features will extend.

**Scope posture:** "Make it work, then make it right." Local-only, no hosting, no auth, no multi-user, no scheduled refresh, no persistence beyond `localStorage`. These are explicitly deferred to a follow-up iteration.

---

## Summary of the Feature

1. The "New Opportunity" button in the Home opportunities queue is renamed to **"Ask pyze"**.
2. Clicking it opens a centered modal with a chat flow: question → plan (confirm/correct) → SQL + run (hidden) → result.
3. The user picks a **mode** (Live metric or Snapshot) and a **visualization type** (Number / Line / Bar / Table; Flow stubbed as "coming soon"), with Gemini pre-selecting a recommendation.
4. On save, the analysis lands in a new left-sidebar section called **My Analyses** — a freeform canvas where the user can arrange cards however they like. Optionally, the analysis can also be pinned to the Home dashboard.
5. Saved cards re-open the chat in follow-up mode for iteration.

---

## Architecture

### Runtime shape

- **Backend:** FastAPI app at `localhost:8000`, serving both the static frontend (`index.html`, assets) and four JSON endpoints.
- **Frontend:** Existing single-file `index.html` extended in-place. No build step, no framework.
- **Storage:** `localStorage` on the client. Backend is stateless.
- **Single-origin:** Frontend and backend on the same origin — no CORS.

### Repository layout

```
pyze-hero/
├── index.html                  # existing, extended
├── pyze_logo.png               # existing
├── server.py                   # new — FastAPI app
├── requirements.txt            # new — fastapi, uvicorn, + deps from analysis-app
├── static/                     # optional, if we split assets
└── docs/
    └── superpowers/specs/
        └── 2026-04-22-ask-pyze-chat-design.md
```

The analysis-app at `/Users/namanchetanrajdev/Documents/analysis-app/` stays in place. `server.py` imports its helper functions directly via a relative path import or by adding it to `PYTHONPATH`.

Rationale: keeping `analysis-app` separate lets the Streamlit version keep working for Naman's ad-hoc use, while the pyze-hero repo owns everything needed to serve the workspace demo. Duplication is avoided — the prompt context, SQL templates, and Gemini helpers all live in one place.

### Endpoints

All endpoints accept/return JSON. All maintain a `messages` array that represents the Gemini conversation turn-by-turn (matches the existing `app.py` pattern).

| Endpoint             | Purpose                                             | Request                                     | Response                                          |
|----------------------|-----------------------------------------------------|---------------------------------------------|---------------------------------------------------|
| `POST /plan`         | First pass on user's question → structured plan     | `{ question, dataset }`                     | `{ plan_text, messages }`                         |
| `POST /plan/refine`  | User-supplied correction → updated plan             | `{ messages, current_plan, correction }`    | `{ plan_text, messages }`                         |
| `POST /sql`          | Plan → validated SQL (with built-in retry)          | `{ messages, current_plan }`                | `{ sql, recommended_viz, validation_error }`      |
| `POST /run`          | Execute validated SQL on BigQuery                   | `{ sql, dataset }`                          | `{ columns, rows }`                               |

**Prompt addition:** the existing `SQL_INSTRUCTIONS` template (in `context.py`) is extended with one line: *"On the final line of your response, include a comment: `-- VIZ: <number|line|bar|table>` recommending the best visualization type for this query's output shape."* The backend parses this off before returning the SQL (so the comment never hits BigQuery), and returns it as `recommended_viz` in the `/sql` response. Frontend uses it to pre-select the viz picker when the result renders.

**Retry logic** for SQL validation is inherited from `app.py` (currently `MAX_SQL_RETRIES = 2`). Dry-run against BQ, feed the error back to Gemini, retry up to 2 times.

**Non-streaming** for v1. Gemini calls take 3–5s each, shown as an in-modal spinner. Streaming is a "make it right" upgrade.

### Run command

```
cd /Users/namanchetanrajdev/Documents/pyze-hero
uvicorn server:app --reload
```

Opens at `http://localhost:8000`.

---

## Chat Flow

### Entry point

- Button in the Home opportunities queue (currently "New Opportunity") is renamed to **"Ask pyze"**.
- A second entry point lives in the top-right of the My Analyses view: a `New Analysis` button that opens the same modal.

### Modal

- **Dimensions:** ~920px wide × ~600px tall, centered.
- **Scrim:** soft semi-transparent navy (e.g., `rgba(15, 29, 74, 0.45)`) — not heavy black. Matches the workspace's premium feel.
- **Entry animation:** scrim fade-in (120ms) + modal scale-in (200ms, transform 0.96 → 1.0, opacity 0 → 1, ease-out).
- **Dismiss:** top-right `×`, Esc key, or clicking outside the modal. Mid-flow state is discarded.

### Stages

The modal progresses through four stages. A compact breadcrumb at the top tracks position: `Question → Plan → Result`.

**Stage 1 — Question**
- Large textarea, placeholder: *"e.g. What's the average handling time per department over the last 3 months?"*
- Small controls below: dataset toggle (CDD / TM), mode picker (see below).
- Primary action: `Analyze` (disabled until text entered).
- Submit calls `POST /plan`.

**Mode picker** — segmented pill sitting beside the dataset toggle:
```
  [ Live metric ]  [ Snapshot ]
```
Live metric is default-selected. Help text below: *"Live metrics re-run on dashboard load. Snapshots freeze the answer at save time."*

**Stage 2 — Plan**
- Plan rendered as a styled definition list (Metric / Grouped by / Filters / Output) — not raw markdown.
- Two primary actions:
  - `Looks right` → advances to SQL+run.
  - `Not quite` → reveals inline correction textarea. Up to 3 corrections (`MAX_CORRECTIONS = 3` in existing code). After 3, force-advances with a soft notice.

**Stage 3 — SQL + Run (hidden from user)**
- Single combined stage. User sees a spinner labeled *"Writing and running your query…"*
- Backend: `POST /sql` (returns validated SQL, retries internally) → `POST /run` (executes against BQ, returns rows + recommended viz).
- If `/sql` exhausts retries or `/run` fails, modal shows the error + `Back to plan` button. No dead-ends.

**Stage 4 — Result**
- Rendered viz (type per picker — see next section).
- Viz picker above.
- Save panel below.

---

## Viz Picker

Segmented control above the result:

```
  Number | Line | Bar | Table | Flow ⚡
```

- **Recommended** pip appears beside whichever option Gemini suggested. That option is initially selected.
- **Flow** is greyed out with tooltip: *"Coming soon — flow diagrams launch in a later release."* Not selectable in v1.
- Switching selection re-renders in place (no backend round-trip — same `rows`, different viz).
- **Compatibility warning** (soft, non-blocking): if user picks Number for a 50-row result, an inline hint appears: *"This view works best for single-value results. Switch to Table?"*

### Viz expectations

| Type    | Expected shape                           | Rendering                                            |
|---------|------------------------------------------|------------------------------------------------------|
| Number  | 1 row × 1 value column                   | Large number + label (Instrument Serif for number)   |
| Line    | Time column + 1+ value columns           | Line chart, pyze palette accents                     |
| Bar     | Label column + 1 value column            | Horizontal or vertical bars, pyze palette            |
| Table   | Any                                      | Geist Mono headers, scrollable body                  |

**Chart library:** Chart.js (lightest viable option, well-documented, no framework needed).

---

## Save Flow

### Save panel

Below the result, inside the same modal — no second modal:

```
  Title: [ Gemini-generated short label, editable ]

  ☑ Save to My Analyses
  ☐ Also pin to Home

  [ Save ]   [ Cancel ]
```

- Title is auto-generated by Gemini from the question (short — e.g., "Avg handling time, last 3 months"). User can edit inline before saving.
- Primary button `Save` uses Rabobank orange (`#fe4811`).

### Saved-analysis record

Written to `localStorage` under key `pyze.analyses`:

```json
{
  "id": "uuid-v4",
  "title": "Avg handling time, last 3 months",
  "question": "What's the average handling time over the last 3 months?",
  "plan": "Metric: Avg handling time\n...",
  "sql": "SELECT ...",
  "messages": [/* Gemini conversation for follow-ups */],
  "mode": "live",
  "viz": "number",
  "dataset": "CDD",
  "createdAt": "2026-04-22T14:32:01Z",
  "updatedAt": "2026-04-22T14:32:01Z",
  "frozenResult": null,
  "lastResult": { "columns": [...], "rows": [...] },
  "lastRunAt": "2026-04-22T14:32:01Z",
  "position": { "x": 24, "y": 24 },
  "pinnedToHome": false
}
```

- For **snapshots**: `frozenResult` is populated, `lastResult` / `lastRunAt` stay null.
- For **live metrics**: `lastResult` / `lastRunAt` populated, `frozenResult` stays null.
- `position` defaults to a cascading top-left slot (see My Analyses section).

### Save animation

1. **Modal collapse** (~280ms) — result card scales to tile-size proportions; modal chrome (buttons, viz picker, breadcrumb) opacity-fades.
2. **Fly-to-destination** (~480ms, cubic-bezier arc):
   - Always flies toward the **My Analyses** sidebar nav item (pulses once on arrival).
   - If `Also pin to Home` is checked, a **second ghost** splits off mid-flight and arcs toward the appropriate Home zone (KPI row for live metrics, Pinned Snapshots strip for snapshots). Lands with a subtle pop-in (scale 0.94 → 1.0, 120ms).
3. **Scrim dismiss** (~120ms). DOM cleanup.

Total perceived save time: ~900ms.

**Implementation approach:** CSS transforms + transitions, no animation library. FLIP technique — record source bounding box, record destination bounding box, animate transform delta.

**Reduced-motion fallback:** `@media (prefers-reduced-motion: reduce)` → skip fly animation, fade-out modal and fade-in tile in place.

---

## My Analyses View

### Sidebar nav entry

```
  • Home
  • My Analyses         ← new
  • (other existing items)
```

Styled identically to existing nav items — simple grid/chart icon, Geist Mono label, navy active state.

### Canvas

- Full content area (same dimensions as Home canvas).
- Header strip:
  ```
  My Analyses                                   [ New Analysis ]
  Your saved questions, arranged however you like.
  ```
- Below the header: a **paper-white canvas** with a very subtle dotted background (4px dot spacing, low opacity) that hints at freeform positioning.

### Card positioning (Freeform, Fixed-size — option B)

- Cards absolutely-positioned on the canvas.
- User drags cards anywhere, any overlap allowed.
- No resize — each viz type has a fixed card size:
  - Number: 240 × 200
  - Line: 400 × 240
  - Bar: 400 × 240
  - Table: 400 × 280
- **New-card placement:** top-left with diagonal cascading offset (each new unmoved card offsets by 24px × 24px to avoid stacking on top).
- **Drag:** card body is grab-cursor. Drop persists `position` to `localStorage`. Most-recently-touched card gets highest z-index.

### Tile anatomy

```
┌───────────────────────────────────────────────┐
│  Avg handling time, last 3 months    [🖊 ⋯]   │  ← title + hover toolbar (translucent)
│  LIVE                                         │  ← mode chip (lime for LIVE, slate for SNAPSHOT)
│                                               │
│                  8:42                         │  ← viz body
│                                               │
│  Updated 12s ago                     CDD      │  ← meta row + dataset tag
└───────────────────────────────────────────────┘
```

- Title: Geist, 13px, 2-line max with ellipsis.
- Mode chip: styled with pyze status palette (LIVE = pyze lime `#7ea82e`; SNAPSHOT = neutral slate).
- Meta row: `Updated Xs ago` (live) or `Saved Apr 18` (snapshot) + dataset tag.
- Hover toolbar (top-right, translucent, PowerBI-style):
  - 🖊 pencil → re-open chat modal in follow-up mode.
  - ⋯ kebab → dropdown: `Duplicate` | `Pin to Home` / `Unpin from Home` | `Delete`.

### Live-metric refresh

On My Analyses page load, every card with `mode: "live"` fires `POST /run` with its saved SQL. Cards show a soft left-to-right shimmer across the viz area while loading. On response, viz updates and `lastRunAt` refreshes.

- Runs **in parallel** (all at once, not sequential).
- No refresh on a timer — only on page load.
- Failures: card shows a muted error badge with tooltip (*"Query failed: …"*). Data from previous `lastResult` stays visible.

### Follow-up mode

Clicking the pencil icon on a card re-opens the chat modal, pre-populated with the saved `messages` conversation. User sees:
- Original question (read-only, as a header at top of modal).
- Original plan rendered.
- Input field for a follow-up: *"Ask a follow-up or refine the plan…"*
- On save: choice of `Replace this analysis` (updates in place) or `Save as new` (creates a new record).

### Empty state

Centered, understated:

```
  No analyses saved yet.
  Click "New Analysis" to get started.
```

---

## Home Integration

### What changes on Home

1. **"New Opportunity" button → renamed "Ask pyze".** Same position in the opportunities queue top-right.
2. **KPI tile row** can receive user-pinned **live metrics**. They appear appended after the four hand-authored tiles, separated by a small dot divider.
3. **New strip: "Pinned snapshots"** — appears between the KPI row and the filter bar *only when at least one snapshot is pinned*. Hidden (zero height) when empty.
4. Process map, drill-downs, opportunities queue, footer — all untouched.

### Pinning / unpinning

- Pin from the tile's kebab menu on My Analyses → tile replicates to Home with the fly-to-destination animation.
- Unpin from the tile's kebab on Home → reverse fly-back toward the sidebar (signals "gone home"). Record's `pinnedToHome` flips to false; analysis still lives in My Analyses.

### Flow diagrams (future)

When v2 lands, flow-diagram cards pinned to Home replace or augment the process-map area. Not in v1 scope.

---

## Data & Privacy Notes

- All analyses live in the user's browser `localStorage`. Clearing browser data wipes them.
- BigQuery queries execute with Naman's gcloud credentials (already set up on his machine).
- Gemini API key is read from env (`GEMINI_API_KEY`) — same as existing `app.py`.
- No user data leaves the local machine beyond the Gemini + BQ API calls inherent to the feature.

---

## Out of Scope (v1)

Explicitly deferred to "make it right":

- **Flow-diagram visualization** (stubbed as "coming soon"). When Naman provides stage-level SQL templates, this becomes v2.
- **Hosting.** Feature runs locally only. No Cloud Run, no Vercel, no auth.
- **Multi-user / shared dashboards.** Single-user localStorage only.
- **Scheduled refresh** for live metrics. v1 refreshes only on page load.
- **Streaming Gemini responses.** v1 uses plain request/response with a spinner.
- **SQLite or DB-backed persistence.** localStorage is sufficient for prototype.
- **Export / share.** No way to export a tile as PNG or share a link.
- **Keyboard navigation** of canvas (tab-focus, arrow-key move). Mouse/trackpad only in v1.
- **Responsive / mobile layout.** Designed for laptop-class viewports only.
- **Analytics / telemetry.** No instrumentation.

---

## Success Criteria

The prototype is successful when:

1. Naman can run `uvicorn server:app --reload` and see the workspace at `localhost:8000`.
2. Clicking "Ask pyze" opens the modal. Typing a real CDD question returns a real plan from Gemini.
3. Confirming the plan returns real SQL, validated against BigQuery, with real results in <15s end-to-end.
4. Saving creates a card in My Analyses with the chosen viz type.
5. Pinning to Home adds the card to the KPI row (live) or Pinned Snapshots strip (snapshot), with the fly-to-destination animation.
6. Reloading the page restores all saved analyses from localStorage, with live metrics re-running automatically.
7. The existing workspace demo (KPIs, process map, drill-downs, opportunities queue) continues to work identically.

---

## Open Questions

None requiring resolution before implementation. The two surfaced during design are both settled:

- **Button naming** → "Ask pyze" (resolved).
- **Where snapshots land on Home** → new "Pinned snapshots" strip (resolved).

Any open questions that surface during implementation should be raised back to this doc before deviating.
