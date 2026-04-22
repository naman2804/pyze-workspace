# Ask pyze Chat Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a local-prototype conversational analysis feature to the pyze workspace demo: a chat modal that takes a user's question, runs it through Gemini + BigQuery, previews the result, and saves it to either a personal "My Analyses" freeform canvas or the Home dashboard.

**Architecture:** FastAPI backend at `localhost:8000` serves the existing static `index.html` and four JSON endpoints that wrap the Gemini + BQ pipeline already in `/Users/namanchetanrajdev/Documents/analysis-app/`. Frontend is the existing single-file `index.html`, extended in place. Pure-logic JS is factored into ES modules (`js/*.js`) to enable a simple browser-based test harness. Storage is `localStorage`. No CORS, no database, no auth.

**Tech Stack:** Python 3.11+, FastAPI, Uvicorn, `google-genai`, `google-cloud-bigquery`, `pandas`, Pytest, vanilla JS + ES modules, Chart.js (via CDN), no build step.

**Reference spec:** `/Users/namanchetanrajdev/Documents/pyze-hero/docs/superpowers/specs/2026-04-22-ask-pyze-chat-design.md`

---

## Conventions used in this plan

- **TDD** for backend Python (pytest).
- **Manual verification** for DOM / animation / visual tasks — each such task includes an explicit checklist and the URL/steps to confirm.
- **Pure-logic JS** (parsers, storage shape, viz-type detection) gets unit tests via a browser-based test runner at `http://localhost:8000/tests.html`.
- **Commits after every task.** Small scope = easy rollback.
- **Secrets via env** — `GEMINI_API_KEY` must be set in the shell running `uvicorn`. Google Cloud auth is taken from the ambient `gcloud` application default.

---

## Task 1: Backend project setup

**Files:**
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/requirements.txt`
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/server.py`
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/tests/__init__.py` (empty)
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/tests/test_server.py`
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/.gitignore`

- [ ] **Step 1: Write `requirements.txt`**

```
fastapi==0.111.*
uvicorn[standard]==0.30.*
google-genai==0.7.*
google-cloud-bigquery==3.25.*
pandas==2.2.*
pytest==8.*
httpx==0.27.*
```

- [ ] **Step 2: Write `.gitignore`**

```
__pycache__/
*.pyc
.venv/
.env
.pytest_cache/
node_modules/
```

- [ ] **Step 3: Write the failing test for the health endpoint**

```python
# tests/test_server.py
from fastapi.testclient import TestClient
from server import app

client = TestClient(app)


def test_health_returns_ok():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- [ ] **Step 4: Run test to confirm it fails**

```bash
cd /Users/namanchetanrajdev/Documents/pyze-hero
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest tests/test_server.py::test_health_returns_ok -v
```

Expected: FAIL with "No module named 'server'".

- [ ] **Step 5: Write minimal `server.py`**

```python
"""FastAPI server for the Ask pyze feature. Serves the static frontend and exposes
Gemini + BigQuery endpoints that wrap helpers from ../analysis-app/."""
import os
import sys
from pathlib import Path
from fastapi import FastAPI
from fastapi.responses import FileResponse
from fastapi.staticfiles import StaticFiles

# Make ../analysis-app importable
ANALYSIS_APP = Path(__file__).parent.parent / "analysis-app"
sys.path.insert(0, str(ANALYSIS_APP))

app = FastAPI(title="pyze workspace backend")


@app.get("/health")
def health():
    return {"status": "ok"}


# Root serves index.html; everything else falls through to static.
@app.get("/")
def root():
    return FileResponse(Path(__file__).parent / "index.html")


# Static assets (pyze_logo.png, future js/ files)
app.mount("/js", StaticFiles(directory=Path(__file__).parent / "js"), name="js")


@app.get("/pyze_logo.png")
def logo():
    return FileResponse(Path(__file__).parent / "pyze_logo.png")
```

Also create `/Users/namanchetanrajdev/Documents/pyze-hero/js/` as an empty directory (the StaticFiles mount above will fail if it doesn't exist):

```bash
mkdir -p /Users/namanchetanrajdev/Documents/pyze-hero/js
touch /Users/namanchetanrajdev/Documents/pyze-hero/js/.gitkeep
```

- [ ] **Step 6: Run the test**

```bash
pytest tests/test_server.py::test_health_returns_ok -v
```

Expected: PASS.

- [ ] **Step 7: Manually verify the server boots**

```bash
uvicorn server:app --reload
```

Open `http://localhost:8000/` — the existing workspace demo should render. Open `http://localhost:8000/health` — should return `{"status":"ok"}`. Kill the server with Ctrl+C.

- [ ] **Step 8: Commit**

```bash
git add requirements.txt .gitignore server.py tests/ js/
git commit -m "feat(server): scaffold FastAPI app serving the static demo + health endpoint"
```

---

## Task 2: `/plan` endpoint

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/server.py`
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/tests/test_server.py`

- [ ] **Step 1: Write the failing test**

```python
# Append to tests/test_server.py
from unittest.mock import patch


def test_plan_returns_plan_and_messages():
    with patch("server.call_gemini") as mock_gemini:
        mock_gemini.return_value = "**Metric:** Avg handling time\n**Grouped by:** department\n**Filters:** last 3 months\n**Output:** table"
        response = client.post("/plan", json={
            "question": "What's the avg handling time by department in the last 3 months?",
            "dataset": "CDD",
        })
    assert response.status_code == 200
    body = response.json()
    assert "Metric" in body["plan_text"]
    assert isinstance(body["messages"], list)
    assert len(body["messages"]) == 1
    assert body["messages"][0]["role"] == "user"


def test_plan_rejects_unknown_dataset():
    response = client.post("/plan", json={"question": "foo", "dataset": "XYZ"})
    assert response.status_code == 400
    assert "dataset" in response.json()["detail"].lower()
```

- [ ] **Step 2: Run to confirm failure**

```bash
pytest tests/test_server.py -v
```

Expected: the two new tests FAIL with 404 (no `/plan` route).

- [ ] **Step 3: Implement `/plan` in `server.py`**

Add to the top of `server.py` (after existing imports):

```python
from fastapi import HTTPException
from pydantic import BaseModel
from google import genai
from context import TABLES, build_system_prompt, PLAN_INSTRUCTIONS

MODEL = "gemini-2.5-pro"
_gemini = None


def gemini_client():
    global _gemini
    if _gemini is None:
        _gemini = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
    return _gemini


def call_gemini(system_prompt: str, messages: list[dict]) -> str:
    contents = []
    for msg in messages:
        role = "user" if msg["role"] == "user" else "model"
        contents.append({"role": role, "parts": [{"text": msg["content"]}]})
    response = gemini_client().models.generate_content(
        model=MODEL,
        contents=contents,
        config={"system_instruction": system_prompt},
    )
    return response.text


class PlanRequest(BaseModel):
    question: str
    dataset: str


class PlanResponse(BaseModel):
    plan_text: str
    messages: list[dict]


@app.post("/plan", response_model=PlanResponse)
def plan(req: PlanRequest):
    if req.dataset not in TABLES:
        raise HTTPException(status_code=400, detail=f"Unknown dataset: {req.dataset}")
    system_prompt = build_system_prompt(TABLES[req.dataset])
    messages = [{"role": "user", "content": f"{req.question}\n\n{PLAN_INSTRUCTIONS}"}]
    plan_text = call_gemini(system_prompt, messages)
    return PlanResponse(plan_text=plan_text, messages=messages)
```

- [ ] **Step 4: Run tests to verify pass**

```bash
pytest tests/test_server.py -v
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add server.py tests/test_server.py
git commit -m "feat(server): add /plan endpoint wrapping Gemini plan generation"
```

---

## Task 3: `/plan/refine` endpoint

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/server.py`
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/tests/test_server.py`

- [ ] **Step 1: Write the failing test**

```python
def test_plan_refine_updates_plan():
    with patch("server.call_gemini") as mock_gemini:
        mock_gemini.return_value = "**Metric:** Avg handling time (corrected)\n**Grouped by:** team\n**Filters:** last 3 months\n**Output:** table"
        response = client.post("/plan/refine", json={
            "messages": [{"role": "user", "content": "original question"}],
            "current_plan": "**Metric:** Avg handling time\n**Grouped by:** department",
            "correction": "Group by team instead of department",
            "dataset": "CDD",
        })
    assert response.status_code == 200
    body = response.json()
    assert "team" in body["plan_text"]
    # messages should contain original + assistant + correction user turn
    assert len(body["messages"]) == 3
    assert body["messages"][-1]["role"] == "user"
    assert "team" in body["messages"][-1]["content"] or "correction" in body["messages"][-1]["content"].lower()
```

- [ ] **Step 2: Run to confirm failure**

```bash
pytest tests/test_server.py::test_plan_refine_updates_plan -v
```

Expected: FAIL (404).

- [ ] **Step 3: Implement in `server.py`**

```python
class RefineRequest(BaseModel):
    messages: list[dict]
    current_plan: str
    correction: str
    dataset: str


@app.post("/plan/refine", response_model=PlanResponse)
def plan_refine(req: RefineRequest):
    if req.dataset not in TABLES:
        raise HTTPException(status_code=400, detail=f"Unknown dataset: {req.dataset}")
    system_prompt = build_system_prompt(TABLES[req.dataset])
    messages = req.messages + [
        {"role": "assistant", "content": req.current_plan},
        {"role": "user", "content": f"That's not quite right. Here's my correction: {req.correction}\n\nPlease update the plan in the same format."},
    ]
    plan_text = call_gemini(system_prompt, messages)
    return PlanResponse(plan_text=plan_text, messages=messages)
```

- [ ] **Step 4: Run tests**

```bash
pytest tests/test_server.py -v
```

Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add server.py tests/test_server.py
git commit -m "feat(server): add /plan/refine endpoint for plan corrections"
```

---

## Task 4: `/sql` endpoint with VIZ extraction and retry loop

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/server.py`
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/tests/test_server.py`

- [ ] **Step 1: Write failing tests for VIZ extraction (pure function)**

```python
# In tests/test_server.py
from server import extract_viz


def test_extract_viz_with_comment():
    sql, viz = extract_viz("SELECT 1\n-- VIZ: number")
    assert sql == "SELECT 1"
    assert viz == "number"


def test_extract_viz_case_insensitive():
    sql, viz = extract_viz("SELECT 1\n-- viz: LINE")
    assert viz == "line"


def test_extract_viz_missing():
    sql, viz = extract_viz("SELECT 1")
    assert sql == "SELECT 1"
    assert viz is None


def test_extract_viz_unknown_value_returns_none():
    # Invalid viz names should be ignored (fall back to frontend default)
    sql, viz = extract_viz("SELECT 1\n-- VIZ: rainbow")
    assert viz is None
    assert "VIZ" not in sql  # still stripped
```

- [ ] **Step 2: Write failing test for `/sql` endpoint**

```python
def test_sql_returns_sql_and_viz():
    fake_sql = "```sql\nSELECT COUNT(*) FROM t\n-- VIZ: number\n```"
    with patch("server.call_gemini") as mock_gemini, \
         patch("server.validate_sql") as mock_validate:
        mock_gemini.return_value = fake_sql
        mock_validate.return_value = None  # no error
        response = client.post("/sql", json={
            "messages": [{"role": "user", "content": "Q"}],
            "current_plan": "plan",
            "dataset": "CDD",
        })
    body = response.json()
    assert response.status_code == 200
    assert body["sql"] == "SELECT COUNT(*) FROM t"
    assert body["recommended_viz"] == "number"
    assert body["validation_error"] is None


def test_sql_retries_on_validation_error():
    with patch("server.call_gemini") as mock_gemini, \
         patch("server.validate_sql") as mock_validate:
        mock_gemini.side_effect = [
            "SELECT broken",
            "SELECT fixed\n-- VIZ: table",
        ]
        mock_validate.side_effect = ["syntax error near 'broken'", None]
        response = client.post("/sql", json={
            "messages": [{"role": "user", "content": "Q"}],
            "current_plan": "plan",
            "dataset": "CDD",
        })
    body = response.json()
    assert body["sql"] == "SELECT fixed"
    assert body["recommended_viz"] == "table"
    assert body["validation_error"] is None
    assert mock_gemini.call_count == 2
```

- [ ] **Step 3: Run to confirm failure**

```bash
pytest tests/test_server.py -v
```

Expected: new tests FAIL.

- [ ] **Step 4: Implement in `server.py`**

```python
from context import SQL_INSTRUCTIONS
from google.cloud import bigquery

MAX_SQL_RETRIES = 2
VALID_VIZ = {"number", "line", "bar", "table"}

VIZ_INSTRUCTION_ADDENDUM = (
    "\n\nAdditionally, on the very last line of your response, include a comment "
    "in this exact format: `-- VIZ: <number|line|bar|table>` indicating which "
    "visualization best fits the query's output shape. Pick 'number' for single-value "
    "results, 'line' for time series, 'bar' for categorical comparisons, 'table' for "
    "multi-column or multi-row results."
)

_bq = None


def bq_client():
    global _bq
    if _bq is None:
        _bq = bigquery.Client(project="pyze-automation-dev3")
    return _bq


def _strip_fences(text: str) -> str:
    text = text.strip()
    if text.startswith("```"):
        text = text.split("\n", 1)[1]
    if text.endswith("```"):
        text = text.rsplit("```", 1)[0]
    return text.strip()


def extract_viz(sql_text: str) -> tuple[str, str | None]:
    """Extract `-- VIZ: <type>` from the last line of SQL. Returns (clean_sql, viz_or_none)."""
    lines = _strip_fences(sql_text).split("\n")
    if lines and lines[-1].strip().lower().startswith("-- viz:"):
        viz = lines[-1].split(":", 1)[1].strip().lower()
        clean = "\n".join(lines[:-1]).strip()
        return clean, viz if viz in VALID_VIZ else None
    return "\n".join(lines).strip(), None


def validate_sql(sql: str) -> str | None:
    """BigQuery dry-run. Returns error string or None."""
    try:
        job_config = bigquery.QueryJobConfig(dry_run=True)
        bq_client().query(sql, job_config=job_config)
        return None
    except Exception as e:
        return str(e)


class SqlRequest(BaseModel):
    messages: list[dict]
    current_plan: str
    dataset: str


class SqlResponse(BaseModel):
    sql: str
    recommended_viz: str | None
    validation_error: str | None


@app.post("/sql", response_model=SqlResponse)
def sql_endpoint(req: SqlRequest):
    if req.dataset not in TABLES:
        raise HTTPException(status_code=400, detail=f"Unknown dataset: {req.dataset}")
    system_prompt = build_system_prompt(TABLES[req.dataset])
    messages = req.messages + [
        {"role": "assistant", "content": req.current_plan},
        {"role": "user", "content": SQL_INSTRUCTIONS + VIZ_INSTRUCTION_ADDENDUM},
    ]
    raw = call_gemini(system_prompt, messages)
    sql, viz = extract_viz(raw)

    fix_messages = messages
    for attempt in range(MAX_SQL_RETRIES):
        err = validate_sql(sql)
        if err is None:
            return SqlResponse(sql=sql, recommended_viz=viz, validation_error=None)
        fix_messages = fix_messages + [
            {"role": "assistant", "content": raw},
            {"role": "user", "content": f"This SQL failed BigQuery validation with this error:\n\n{err}\n\nFix the SQL. Return ONLY the corrected SQL query, no explanation. Remember the `-- VIZ: ...` comment on the final line."},
        ]
        raw = call_gemini(system_prompt, fix_messages)
        sql, viz = extract_viz(raw)

    err = validate_sql(sql)
    return SqlResponse(sql=sql, recommended_viz=viz, validation_error=err)
```

- [ ] **Step 5: Run all tests**

```bash
pytest tests/test_server.py -v
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add server.py tests/test_server.py
git commit -m "feat(server): add /sql endpoint with retry loop + VIZ recommendation parsing"
```

---

## Task 5: `/run` endpoint

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/server.py`
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/tests/test_server.py`

- [ ] **Step 1: Write failing test**

```python
def test_run_returns_rows_and_columns():
    import pandas as pd
    fake_df = pd.DataFrame({"dept": ["A", "B"], "avg_ms": [1000, 2000]})
    with patch("server.run_query") as mock_run:
        mock_run.return_value = fake_df
        response = client.post("/run", json={"sql": "SELECT 1", "dataset": "CDD"})
    body = response.json()
    assert response.status_code == 200
    assert body["columns"] == ["dept", "avg_ms"]
    assert body["rows"] == [["A", 1000], ["B", 2000]]


def test_run_returns_error_on_failure():
    with patch("server.run_query") as mock_run:
        mock_run.side_effect = Exception("permission denied")
        response = client.post("/run", json={"sql": "SELECT 1", "dataset": "CDD"})
    assert response.status_code == 500
    assert "permission denied" in response.json()["detail"]
```

- [ ] **Step 2: Run to confirm failure**

```bash
pytest tests/test_server.py -v
```

Expected: new tests FAIL.

- [ ] **Step 3: Implement in `server.py`**

```python
def run_query(sql: str):
    return bq_client().query(sql).to_dataframe()


class RunRequest(BaseModel):
    sql: str
    dataset: str


class RunResponse(BaseModel):
    columns: list[str]
    rows: list[list]


@app.post("/run", response_model=RunResponse)
def run(req: RunRequest):
    if req.dataset not in TABLES:
        raise HTTPException(status_code=400, detail=f"Unknown dataset: {req.dataset}")
    try:
        df = run_query(req.sql)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
    return RunResponse(
        columns=list(df.columns),
        # Convert non-JSON-serializable types (Timestamp, numpy types) to plain strings/floats
        rows=df.astype(object).where(df.notna(), None).values.tolist(),
    )
```

Note: the `df.astype(object).where(df.notna(), None).values.tolist()` dance converts NaN to None and makes everything JSON-friendly. For Timestamps specifically, add an additional iso-format conversion; if Timestamps aren't produced by current KPIs, skip for now and handle in Task 23 (live-metric refresh) when real data flows in.

- [ ] **Step 4: Run tests**

```bash
pytest tests/test_server.py -v
```

Expected: all pass.

- [ ] **Step 5: Manual smoke test — hit real BigQuery**

```bash
uvicorn server:app --reload
```

In another terminal:

```bash
curl -s -X POST http://localhost:8000/run -H "Content-Type: application/json" \
  -d '{"sql":"SELECT COUNT(*) AS n FROM `pyze-automation-dev3.rabo_123.Rabo_CDD_FECRelatedCases`","dataset":"CDD"}' | python -m json.tool
```

Expected: `{"columns":["n"],"rows":[[<some_integer>]]}`. Kill the server.

- [ ] **Step 6: Commit**

```bash
git add server.py tests/test_server.py
git commit -m "feat(server): add /run endpoint for BigQuery execution"
```

---

## Task 6: Frontend JS modules + browser test harness

**Files:**
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/js/api.js`
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/js/storage.js`
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/js/viz.js`
- Create: `/Users/namanchetanrajdev/Documents/pyze-hero/tests.html`

- [ ] **Step 1: Write `js/api.js`**

```javascript
// js/api.js — thin wrappers over the backend endpoints. All return parsed JSON.

async function post(path, body) {
  const res = await fetch(path, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  if (!res.ok) {
    const detail = await res.json().catch(() => ({ detail: res.statusText }));
    throw new Error(detail.detail || "Request failed");
  }
  return res.json();
}

export const api = {
  plan: (question, dataset) => post("/plan", { question, dataset }),
  refine: (messages, currentPlan, correction, dataset) =>
    post("/plan/refine", { messages, current_plan: currentPlan, correction, dataset }),
  sql: (messages, currentPlan, dataset) =>
    post("/sql", { messages, current_plan: currentPlan, dataset }),
  run: (sql, dataset) => post("/run", { sql, dataset }),
};
```

- [ ] **Step 2: Write `js/storage.js`**

```javascript
// js/storage.js — localStorage-backed store for saved analyses.

const KEY = "pyze.analyses";

export function loadAll() {
  try {
    return JSON.parse(localStorage.getItem(KEY) || "[]");
  } catch {
    return [];
  }
}

export function saveAll(items) {
  localStorage.setItem(KEY, JSON.stringify(items));
}

export function upsert(item) {
  const items = loadAll();
  const idx = items.findIndex((x) => x.id === item.id);
  if (idx === -1) items.unshift(item);
  else items[idx] = item;
  saveAll(items);
  return item;
}

export function remove(id) {
  saveAll(loadAll().filter((x) => x.id !== id));
}

export function get(id) {
  return loadAll().find((x) => x.id === id) || null;
}

export function duplicate(id) {
  const src = get(id);
  if (!src) return null;
  const copy = { ...src, id: crypto.randomUUID(), title: src.title + " (copy)", createdAt: new Date().toISOString(), pinnedToHome: false };
  upsert(copy);
  return copy;
}

export function newRecord({ question, plan, sql, messages, mode, viz, dataset, title, result }) {
  const now = new Date().toISOString();
  return {
    id: crypto.randomUUID(),
    title,
    question,
    plan,
    sql,
    messages,
    mode,
    viz,
    dataset,
    createdAt: now,
    updatedAt: now,
    frozenResult: mode === "snapshot" ? result : null,
    lastResult: mode === "live" ? result : null,
    lastRunAt: mode === "live" ? now : null,
    position: null,
    pinnedToHome: false,
  };
}
```

- [ ] **Step 3: Write `js/viz.js` (skeleton — renderers filled in later)**

```javascript
// js/viz.js — visualization type inference and rendering dispatch.

export const VIZ_TYPES = ["number", "line", "bar", "table"];

export function inferViz({ columns, rows }) {
  if (!rows || rows.length === 0) return "table";
  if (rows.length === 1 && columns.length === 1) return "number";
  // Detect time column (name contains date/time/day/month/week)
  const timeRegex = /(date|time|day|week|month|hour|year)/i;
  const hasTime = columns.some((c) => timeRegex.test(c));
  if (hasTime && columns.length >= 2 && rows.length > 1) return "line";
  if (columns.length === 2 && rows.length <= 20) return "bar";
  return "table";
}

// Renderers — each renders into a given container element. Implementations added in Tasks 13-14.
export const renderers = {
  number: (container, data) => { container.textContent = "TODO"; },
  line: (container, data) => { container.textContent = "TODO"; },
  bar: (container, data) => { container.textContent = "TODO"; },
  table: (container, data) => { container.textContent = "TODO"; },
};

export function render(container, vizType, data) {
  container.innerHTML = "";
  const fn = renderers[vizType] || renderers.table;
  fn(container, data);
}
```

- [ ] **Step 4: Write `tests.html` browser test harness**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>pyze frontend tests</title>
  <style>
    body { font-family: monospace; padding: 20px; background: #fff; }
    .pass { color: #2d7a4a; } .fail { color: #b8362a; } .summary { font-weight: bold; margin-top: 20px; }
  </style>
</head>
<body>
  <h1>Frontend tests</h1>
  <div id="results"></div>
  <script type="module">
    import { inferViz, VIZ_TYPES } from "/js/viz.js";
    import { newRecord, loadAll, saveAll } from "/js/storage.js";

    const results = document.getElementById("results");
    let pass = 0, fail = 0;

    function test(name, fn) {
      try { fn(); results.innerHTML += `<div class="pass">✓ ${name}</div>`; pass++; }
      catch (e) { results.innerHTML += `<div class="fail">✗ ${name} — ${e.message}</div>`; fail++; }
    }

    function assertEq(a, b, msg = "") {
      if (JSON.stringify(a) !== JSON.stringify(b)) throw new Error(`${msg} expected ${JSON.stringify(b)} got ${JSON.stringify(a)}`);
    }

    // --- inferViz ---
    test("inferViz: single value → number", () =>
      assertEq(inferViz({ columns: ["x"], rows: [[1]] }), "number"));
    test("inferViz: time series → line", () =>
      assertEq(inferViz({ columns: ["date", "value"], rows: [["2025-01", 1], ["2025-02", 2]] }), "line"));
    test("inferViz: small categorical → bar", () =>
      assertEq(inferViz({ columns: ["dept", "avg"], rows: [["A", 1], ["B", 2]] }), "bar"));
    test("inferViz: empty → table", () =>
      assertEq(inferViz({ columns: [], rows: [] }), "table"));

    // --- storage ---
    test("newRecord: snapshot has frozenResult", () => {
      const r = newRecord({ question: "q", plan: "p", sql: "s", messages: [], mode: "snapshot", viz: "number", dataset: "CDD", title: "t", result: { columns: ["x"], rows: [[1]] } });
      assertEq(r.frozenResult, { columns: ["x"], rows: [[1]] });
      assertEq(r.lastResult, null);
    });
    test("newRecord: live has lastResult", () => {
      const r = newRecord({ question: "q", plan: "p", sql: "s", messages: [], mode: "live", viz: "number", dataset: "CDD", title: "t", result: { columns: ["x"], rows: [[1]] } });
      assertEq(r.lastResult, { columns: ["x"], rows: [[1]] });
      assertEq(r.frozenResult, null);
    });

    // Clean up test storage state
    saveAll([]);

    results.innerHTML += `<div class="summary ${fail === 0 ? 'pass' : 'fail'}">${pass} passed, ${fail} failed</div>`;
  </script>
</body>
</html>
```

- [ ] **Step 5: Run server and verify**

```bash
uvicorn server:app --reload
```

Open `http://localhost:8000/tests.html`. Expect: "6 passed, 0 failed".

- [ ] **Step 6: Commit**

```bash
git add js/ tests.html
git commit -m "feat(ui): scaffold js modules (api, storage, viz) and browser test harness"
```

---

## Task 7: Rename "New opportunity" button to "Ask pyze" + open empty modal

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html` (around line 534)

- [ ] **Step 1: Rename the button text**

In `index.html`, locate line 534 (contains `New opportunity`). Change `New opportunity` → `Ask pyze`. Keep classes and icon intact.

- [ ] **Step 2: Add a unique id to the button**

Modify the same line to add `id="btn-ask-pyze"`:

```html
<button id="btn-ask-pyze" class="chip-btn primary">...Ask pyze</button>
```

- [ ] **Step 3: Add modal shell HTML at the end of `<body>` (just before the closing `</body>` tag)**

```html
<div id="askpyze-modal" class="askpyze-modal" hidden aria-hidden="true" role="dialog">
  <div class="askpyze-scrim"></div>
  <div class="askpyze-frame">
    <button class="askpyze-close" id="askpyze-close" aria-label="Close">×</button>
    <div class="askpyze-content" id="askpyze-content">
      <!-- populated by JS -->
    </div>
  </div>
</div>
```

- [ ] **Step 4: Add modal CSS inside the existing `<style>` block (near the end, before closing `</style>`)**

```css
.askpyze-modal { position: fixed; inset: 0; z-index: 1000; display: flex; align-items: center; justify-content: center; }
.askpyze-modal[hidden] { display: none; }
.askpyze-scrim { position: absolute; inset: 0; background: rgba(15, 29, 74, 0.45); backdrop-filter: blur(2px); }
.askpyze-frame { position: relative; width: 920px; max-width: calc(100vw - 40px); height: 600px; max-height: calc(100vh - 40px); background: #fff; border-radius: 12px; box-shadow: 0 20px 60px rgba(15, 29, 74, 0.25); display: flex; flex-direction: column; overflow: hidden; }
.askpyze-close { position: absolute; top: 14px; right: 16px; background: none; border: none; font-size: 24px; color: var(--ink-2); cursor: pointer; z-index: 2; }
.askpyze-close:hover { color: var(--ink); }
.askpyze-content { flex: 1 1 auto; padding: 32px 40px; overflow-y: auto; }
```

- [ ] **Step 5: Add open/close JS inside an existing or new `<script>` tag near the bottom of the body**

```html
<script type="module">
  const modal = document.getElementById("askpyze-modal");
  const openBtn = document.getElementById("btn-ask-pyze");
  const closeBtn = document.getElementById("askpyze-close");
  const scrim = modal.querySelector(".askpyze-scrim");

  function open() { modal.hidden = false; modal.setAttribute("aria-hidden", "false"); document.body.style.overflow = "hidden"; }
  function close() { modal.hidden = true; modal.setAttribute("aria-hidden", "true"); document.body.style.overflow = ""; }

  openBtn.addEventListener("click", open);
  closeBtn.addEventListener("click", close);
  scrim.addEventListener("click", close);
  document.addEventListener("keydown", (e) => { if (e.key === "Escape" && !modal.hidden) close(); });

  window.askPyzeModal = { open, close };  // expose for later tasks
</script>
```

- [ ] **Step 6: Manual verify**

Reload `http://localhost:8000/`. Locate the Ask pyze button in the opportunities header. Click it — modal should appear with scrim and an empty white card. Press Esc, click scrim, or click × — modal should close.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(ui): rename 'New opportunity' to 'Ask pyze' and add modal shell with open/close"
```

---

## Task 8: Chat modal stage machine — Question stage

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

This task introduces a state machine that controls which stage renders inside the modal. It starts with the Question stage only; subsequent tasks add Plan, Result, etc.

- [ ] **Step 1: Add chat-stage CSS inside the existing `<style>` block**

```css
/* Ask pyze — chat stages */
.askpyze-header { display: flex; align-items: center; gap: 10px; font-family: "Geist Mono", monospace; font-size: 11px; letter-spacing: 0.08em; text-transform: uppercase; color: var(--muted); margin-bottom: 20px; }
.askpyze-header .crumb.active { color: var(--ink); }
.askpyze-header .crumb.done { color: var(--ember); }
.askpyze-header .arrow { color: var(--dim); }

.askpyze-stage h2 { font-family: "Instrument Serif", serif; font-size: 28px; font-weight: 400; color: var(--ink); margin: 0 0 8px; line-height: 1.2; }
.askpyze-stage p.hint { font-family: "Geist", sans-serif; font-size: 13px; color: var(--muted); margin: 0 0 24px; }

.askpyze-q-textarea { width: 100%; min-height: 140px; padding: 16px; font-family: "Geist", sans-serif; font-size: 15px; line-height: 1.5; border: 1px solid var(--line-2); border-radius: 8px; resize: vertical; outline: none; color: var(--ink); }
.askpyze-q-textarea:focus { border-color: var(--ember); box-shadow: 0 0 0 3px var(--ember-wash); }

.askpyze-controls { display: flex; align-items: center; gap: 24px; margin-top: 16px; }
.askpyze-field-label { font-family: "Geist Mono", monospace; font-size: 10px; letter-spacing: 0.08em; text-transform: uppercase; color: var(--muted); margin-right: 8px; }
.askpyze-toggle { display: inline-flex; background: var(--paper-2); border-radius: 6px; padding: 3px; }
.askpyze-toggle button { background: none; border: none; padding: 6px 14px; font-family: "Geist Mono", monospace; font-size: 11px; color: var(--muted); cursor: pointer; border-radius: 4px; }
.askpyze-toggle button.on { background: #fff; color: var(--ink); box-shadow: 0 1px 2px rgba(0,0,0,.08); }

.askpyze-actions { display: flex; justify-content: flex-end; gap: 12px; margin-top: 28px; }
.askpyze-btn-primary { background: var(--ember); color: #fff; border: none; padding: 10px 20px; font-family: "Geist", sans-serif; font-size: 14px; font-weight: 500; border-radius: 6px; cursor: pointer; transition: background .15s; }
.askpyze-btn-primary:hover:not(:disabled) { background: var(--ember-hover); }
.askpyze-btn-primary:disabled { background: var(--dim); cursor: not-allowed; }
.askpyze-btn-secondary { background: transparent; color: var(--ink-2); border: 1px solid var(--line-2); padding: 10px 20px; font-family: "Geist", sans-serif; font-size: 14px; border-radius: 6px; cursor: pointer; }
.askpyze-btn-secondary:hover { border-color: var(--ink-2); color: var(--ink); }
```

- [ ] **Step 2: Replace the inline `<script type="module">` from Task 7 with the stage-machine version**

```html
<script type="module">
  import { api } from "/js/api.js";

  const modal = document.getElementById("askpyze-modal");
  const content = document.getElementById("askpyze-content");
  const openBtn = document.getElementById("btn-ask-pyze");
  const closeBtn = document.getElementById("askpyze-close");
  const scrim = modal.querySelector(".askpyze-scrim");

  const state = {
    stage: "question",  // question | plan | running | result
    question: "",
    dataset: "CDD",
    mode: "live",       // live | snapshot
    messages: [],
    currentPlan: "",
    correctionCount: 0,
    sql: "",
    recommendedViz: null,
    result: null,
    error: null,
  };

  function reset() {
    Object.assign(state, { stage: "question", question: "", dataset: "CDD", mode: "live", messages: [], currentPlan: "", correctionCount: 0, sql: "", recommendedViz: null, result: null, error: null });
  }

  function open() { reset(); modal.hidden = false; document.body.style.overflow = "hidden"; render(); }
  function close() { modal.hidden = true; document.body.style.overflow = ""; }

  openBtn.addEventListener("click", open);
  closeBtn.addEventListener("click", close);
  scrim.addEventListener("click", close);
  document.addEventListener("keydown", (e) => { if (e.key === "Escape" && !modal.hidden) close(); });

  function breadcrumb() {
    const s = state.stage;
    const cls = (name, order) => {
      if (s === name) return "crumb active";
      const idx = ["question", "plan", "result"].indexOf(s);
      return order < idx ? "crumb done" : "crumb";
    };
    return `
      <div class="askpyze-header">
        <span class="${cls("question", 0)}">Question</span>
        <span class="arrow">→</span>
        <span class="${cls("plan", 1)}">Plan</span>
        <span class="arrow">→</span>
        <span class="${cls("result", 2)}">Result</span>
      </div>`;
  }

  function renderQuestion() {
    content.innerHTML = `
      ${breadcrumb()}
      <div class="askpyze-stage">
        <h2>What do you want to know?</h2>
        <p class="hint">Ask in plain English — pyze will translate it into the right query.</p>
        <textarea class="askpyze-q-textarea" id="q-input" placeholder="e.g. What's the average handling time per department over the last 3 months?">${state.question}</textarea>
        <div class="askpyze-controls">
          <div><span class="askpyze-field-label">Dataset</span>
            <span class="askpyze-toggle" id="toggle-dataset">
              <button data-val="CDD" class="${state.dataset === "CDD" ? "on" : ""}">CDD</button>
              <button data-val="TM" class="${state.dataset === "TM" ? "on" : ""}">TM</button>
            </span>
          </div>
          <div><span class="askpyze-field-label">Mode</span>
            <span class="askpyze-toggle" id="toggle-mode">
              <button data-val="live" class="${state.mode === "live" ? "on" : ""}">Live metric</button>
              <button data-val="snapshot" class="${state.mode === "snapshot" ? "on" : ""}">Snapshot</button>
            </span>
          </div>
        </div>
        <div class="askpyze-actions">
          <button class="askpyze-btn-primary" id="btn-analyze" disabled>Analyze</button>
        </div>
      </div>`;

    const textarea = content.querySelector("#q-input");
    const btn = content.querySelector("#btn-analyze");
    textarea.addEventListener("input", (e) => { state.question = e.target.value; btn.disabled = !state.question.trim(); });
    btn.disabled = !state.question.trim();
    content.querySelector("#toggle-dataset").addEventListener("click", (e) => {
      const val = e.target.closest("button")?.dataset.val; if (!val) return; state.dataset = val; render();
    });
    content.querySelector("#toggle-mode").addEventListener("click", (e) => {
      const val = e.target.closest("button")?.dataset.val; if (!val) return; state.mode = val; render();
    });
    btn.addEventListener("click", onAnalyze);
  }

  async function onAnalyze() {
    state.stage = "running-plan";
    render();
    try {
      const { plan_text, messages } = await api.plan(state.question, state.dataset);
      state.currentPlan = plan_text;
      state.messages = messages;
      state.stage = "plan";
      render();
    } catch (e) {
      state.error = e.message;
      state.stage = "error";
      render();
    }
  }

  function renderRunning(label) {
    content.innerHTML = `
      ${breadcrumb()}
      <div class="askpyze-stage" style="display:flex;flex-direction:column;align-items:center;justify-content:center;min-height:400px;">
        <div style="width:40px;height:40px;border:3px solid var(--line);border-top-color:var(--ember);border-radius:50%;animation:spin 0.8s linear infinite;"></div>
        <p style="margin-top:20px;color:var(--muted);font-family:'Geist Mono',monospace;font-size:12px;">${label}</p>
      </div>
      <style>@keyframes spin { to { transform: rotate(360deg); } }</style>`;
  }

  function renderError() {
    content.innerHTML = `
      ${breadcrumb()}
      <div class="askpyze-stage">
        <h2>Something went wrong</h2>
        <p class="hint">${state.error}</p>
        <div class="askpyze-actions">
          <button class="askpyze-btn-secondary" id="btn-back">Start over</button>
        </div>
      </div>`;
    content.querySelector("#btn-back").addEventListener("click", () => { reset(); render(); });
  }

  function render() {
    if (state.stage === "question") return renderQuestion();
    if (state.stage === "running-plan") return renderRunning("Understanding your question…");
    if (state.stage === "error") return renderError();
    if (state.stage === "plan") { content.innerHTML = `<div>Plan stage stub — implemented in next task. Plan:<pre>${state.currentPlan}</pre></div>`; return; }
    content.innerHTML = "<div>Unknown stage — " + state.stage + "</div>";
  }
</script>
```

- [ ] **Step 3: Manual verify**

Reload. Click Ask pyze — modal opens with the Question stage. Type a question — Analyze button enables. Toggle dataset/mode — buttons switch visually. Click Analyze — spinner appears, then plan-stage stub shows the raw Gemini plan.

If Gemini errors (no API key etc.), error screen shows with Start over.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ui): add Ask pyze chat stage machine with Question stage"
```

---

## Task 9: Plan stage — render plan, confirm, correct

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add plan-rendering CSS inside `<style>`**

```css
.askpyze-plan { background: var(--paper-2); border: 1px solid var(--line); border-radius: 8px; padding: 20px 24px; margin-bottom: 20px; }
.askpyze-plan dt { font-family: "Geist Mono", monospace; font-size: 10px; letter-spacing: 0.08em; text-transform: uppercase; color: var(--muted); margin-top: 14px; }
.askpyze-plan dt:first-child { margin-top: 0; }
.askpyze-plan dd { font-family: "Geist", sans-serif; font-size: 14px; color: var(--ink); margin: 4px 0 0; }

.askpyze-correction { margin-top: 16px; display: flex; gap: 10px; }
.askpyze-correction input { flex: 1; padding: 10px 14px; font-family: "Geist", sans-serif; font-size: 14px; border: 1px solid var(--line-2); border-radius: 6px; outline: none; }
.askpyze-correction input:focus { border-color: var(--ember); }
```

- [ ] **Step 2: Add a plan-parser helper at the top of the `<script type="module">`**

```javascript
function parsePlan(markdown) {
  const entries = [];
  const regex = /\*\*(.+?):\*\*\s*(.+)/g;
  let m;
  while ((m = regex.exec(markdown)) !== null) {
    entries.push([m[1].trim(), m[2].trim()]);
  }
  return entries.length ? entries : [["Plan", markdown.trim()]];
}
```

- [ ] **Step 3: Replace the plan-stage stub in `render()` with a real renderer**

```javascript
function renderPlan() {
  const entries = parsePlan(state.currentPlan);
  const entriesHtml = entries.map(([k, v]) => `<dt>${k}</dt><dd>${v}</dd>`).join("");
  const canCorrect = state.correctionCount < 3;
  content.innerHTML = `
    ${breadcrumb()}
    <div class="askpyze-stage">
      <h2>Here's what I understand</h2>
      <p class="hint">Check that pyze interpreted your question correctly. If anything's off, correct it below.</p>
      <dl class="askpyze-plan">${entriesHtml}</dl>
      ${canCorrect ? `
        <div class="askpyze-correction">
          <input id="correction-input" type="text" placeholder="What needs to change? (e.g. group by team instead of department)" />
          <button class="askpyze-btn-secondary" id="btn-correct">Update</button>
        </div>
        <p class="hint" style="margin-top:8px;">Corrections used: ${state.correctionCount} / 3</p>
      ` : `<p class="hint">Max corrections reached — proceeding with current plan.</p>`}
      <div class="askpyze-actions">
        <button class="askpyze-btn-primary" id="btn-confirm">Looks right — run it</button>
      </div>
    </div>`;

  content.querySelector("#btn-confirm").addEventListener("click", onConfirmPlan);
  if (canCorrect) {
    content.querySelector("#btn-correct").addEventListener("click", onCorrectPlan);
  }
}

async function onCorrectPlan() {
  const correction = content.querySelector("#correction-input").value.trim();
  if (!correction) return;
  state.stage = "running-plan";
  render();
  try {
    const { plan_text, messages } = await api.refine(state.messages, state.currentPlan, correction, state.dataset);
    state.currentPlan = plan_text;
    state.messages = messages;
    state.correctionCount += 1;
    state.stage = "plan";
    render();
  } catch (e) { state.error = e.message; state.stage = "error"; render(); }
}

function onConfirmPlan() {
  state.stage = "running-sql";
  render();
  // Actual /sql + /run wiring added in Task 10.
  setTimeout(() => { state.error = "SQL stage not implemented yet."; state.stage = "error"; render(); }, 300);
}
```

Update `render()` to call `renderPlan()` for `state.stage === "plan"` and `renderRunning("Running your query…")` for `running-sql`.

- [ ] **Step 4: Manual verify**

Reload. Ask a question (e.g. "What's the count of cases in CDD?"). After the Analyze call returns, Plan stage should render the parsed definition list. Click "Update" with a correction — plan should refine. Clicking "Looks right" should surface the stub error "SQL stage not implemented yet."

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(ui): add Plan stage with confirm + up to 3 corrections"
```

---

## Task 10: SQL + Run stage (wire /sql and /run, show result as table)

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add temporary table-renderer CSS**

```css
.askpyze-result { margin-top: 8px; border: 1px solid var(--line); border-radius: 8px; overflow: hidden; max-height: 360px; overflow-y: auto; }
.askpyze-result table { width: 100%; border-collapse: collapse; font-family: "Geist Mono", monospace; font-size: 12px; }
.askpyze-result th { text-align: left; padding: 8px 12px; background: var(--paper-2); border-bottom: 1px solid var(--line); font-weight: 500; color: var(--muted); }
.askpyze-result td { padding: 8px 12px; border-bottom: 1px solid var(--line); color: var(--ink); }
.askpyze-result tr:last-child td { border-bottom: none; }
```

- [ ] **Step 2: Replace the `onConfirmPlan` stub and add `renderResult`**

```javascript
async function onConfirmPlan() {
  state.stage = "running-sql";
  render();
  try {
    const sqlRes = await api.sql(state.messages, state.currentPlan, state.dataset);
    if (sqlRes.validation_error) {
      state.error = "Couldn't build a valid query after retrying: " + sqlRes.validation_error;
      state.stage = "error"; render(); return;
    }
    state.sql = sqlRes.sql;
    state.recommendedViz = sqlRes.recommended_viz || "table";
    const runRes = await api.run(state.sql, state.dataset);
    state.result = runRes;  // { columns, rows }
    state.stage = "result";
    render();
  } catch (e) { state.error = e.message; state.stage = "error"; render(); }
}

function renderResult() {
  const { columns, rows } = state.result;
  const thead = columns.map((c) => `<th>${c}</th>`).join("");
  const tbody = rows.slice(0, 100).map((r) =>
    `<tr>${r.map((v) => `<td>${v === null ? "—" : v}</td>`).join("")}</tr>`
  ).join("");
  const truncated = rows.length > 100 ? `<p class="hint" style="margin-top:8px;">Showing first 100 of ${rows.length} rows.</p>` : "";
  content.innerHTML = `
    ${breadcrumb()}
    <div class="askpyze-stage">
      <h2>Here's what I found</h2>
      <div class="askpyze-result">
        <table><thead><tr>${thead}</tr></thead><tbody>${tbody}</tbody></table>
      </div>
      ${truncated}
      <div class="askpyze-actions">
        <button class="askpyze-btn-secondary" id="btn-back-plan">Back to plan</button>
        <button class="askpyze-btn-primary" id="btn-save">Save analysis</button>
      </div>
    </div>`;
  content.querySelector("#btn-back-plan").addEventListener("click", () => { state.stage = "plan"; render(); });
  content.querySelector("#btn-save").addEventListener("click", () => alert("Save flow — Task 15"));
}
```

Update `render()` to call `renderResult()` when `state.stage === "result"`.

- [ ] **Step 3: Manual verify (requires real Gemini + BQ)**

```bash
uvicorn server:app --reload
```

Ask a simple real question: "How many cases are in CDD?". Expected flow: Question → Plan → spinner → Result stage with a 1-column/1-row table showing the count. Try a harder question that should produce multiple rows.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ui): wire SQL and run stages, show results as table"
```

---

## Task 11: Viz picker UI + five viz types

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/js/viz.js`

- [ ] **Step 1: Add Chart.js via CDN in `<head>` of `index.html`**

```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
```

- [ ] **Step 2: Replace the stub renderers in `js/viz.js` with real implementations**

```javascript
// js/viz.js — replace the `renderers` object with:
const PYZE_COLORS = ["#7ea82e", "#2ba8b6", "#e88a2e", "#b83d70", "#fe4811", "#0f1d4a"];

function renderNumber(container, { columns, rows }) {
  const value = rows[0]?.[0];
  const label = columns[0] || "";
  const formatted = typeof value === "number" ? value.toLocaleString() : String(value);
  container.innerHTML = `
    <div style="display:flex;flex-direction:column;align-items:center;justify-content:center;height:100%;">
      <div style="font-family:'Instrument Serif',serif;font-size:72px;line-height:1;color:#0f1d4a;">${formatted}</div>
      <div style="margin-top:12px;font-family:'Geist Mono',monospace;font-size:11px;letter-spacing:0.08em;text-transform:uppercase;color:#6b7590;">${label}</div>
    </div>`;
}

function renderLine(container, { columns, rows }) {
  const canvas = document.createElement("canvas");
  container.appendChild(canvas);
  const labels = rows.map((r) => r[0]);
  const datasets = columns.slice(1).map((col, i) => ({
    label: col,
    data: rows.map((r) => r[i + 1]),
    borderColor: PYZE_COLORS[i % PYZE_COLORS.length],
    backgroundColor: PYZE_COLORS[i % PYZE_COLORS.length] + "20",
    tension: 0.3,
    pointRadius: 2,
  }));
  new Chart(canvas, {
    type: "line",
    data: { labels, datasets },
    options: { responsive: true, maintainAspectRatio: false, plugins: { legend: { display: datasets.length > 1 } }, scales: { x: { ticks: { font: { family: "Geist Mono" } } }, y: { ticks: { font: { family: "Geist Mono" } } } } },
  });
}

function renderBar(container, { columns, rows }) {
  const canvas = document.createElement("canvas");
  container.appendChild(canvas);
  const labels = rows.map((r) => String(r[0]));
  const data = rows.map((r) => r[1]);
  new Chart(canvas, {
    type: "bar",
    data: { labels, datasets: [{ label: columns[1] || "value", data, backgroundColor: PYZE_COLORS[0] }] },
    options: { responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false } } },
  });
}

function renderTable(container, { columns, rows }) {
  const thead = columns.map((c) => `<th>${c}</th>`).join("");
  const tbody = rows.slice(0, 100).map((r) =>
    `<tr>${r.map((v) => `<td>${v === null ? "—" : v}</td>`).join("")}</tr>`
  ).join("");
  container.innerHTML = `
    <div class="pz-table-wrap">
      <table><thead><tr>${thead}</tr></thead><tbody>${tbody}</tbody></table>
    </div>`;
}

export const renderers = { number: renderNumber, line: renderLine, bar: renderBar, table: renderTable };
```

- [ ] **Step 3: Add viz-picker CSS in `<style>`**

```css
.askpyze-vizpicker { display: inline-flex; gap: 4px; margin-bottom: 16px; }
.askpyze-vizpicker button { background: var(--paper-2); border: 1px solid transparent; padding: 6px 14px; font-family: "Geist Mono", monospace; font-size: 11px; color: var(--muted); cursor: pointer; border-radius: 6px; position: relative; }
.askpyze-vizpicker button.on { background: #fff; border-color: var(--ember); color: var(--ember); font-weight: 500; }
.askpyze-vizpicker button.disabled { opacity: 0.4; cursor: not-allowed; }
.askpyze-vizpicker .rec-pip { position: absolute; top: -4px; right: -4px; width: 8px; height: 8px; background: var(--ember); border-radius: 50%; }
.askpyze-viz-wrap { height: 320px; border: 1px solid var(--line); border-radius: 8px; padding: 16px; display: flex; align-items: center; justify-content: center; }
.pz-table-wrap { width: 100%; height: 100%; overflow: auto; }
.pz-table-wrap table { width: 100%; border-collapse: collapse; font-family: "Geist Mono", monospace; font-size: 11px; }
.pz-table-wrap th { text-align: left; padding: 8px 12px; background: var(--paper-2); border-bottom: 1px solid var(--line); font-weight: 500; color: var(--muted); position: sticky; top: 0; }
.pz-table-wrap td { padding: 6px 12px; border-bottom: 1px solid var(--line); color: var(--ink); }
```

- [ ] **Step 4: Update `renderResult` in `index.html` to use the viz picker**

```javascript
import { render as renderViz, VIZ_TYPES } from "/js/viz.js";

// Inside the module script, track current viz type in state:
// state.viz = state.recommendedViz;

function renderResult() {
  if (!state.viz) state.viz = state.recommendedViz || "table";
  const pickerHtml = VIZ_TYPES.map((v) => {
    const rec = v === state.recommendedViz ? '<span class="rec-pip" title="Recommended"></span>' : "";
    return `<button data-viz="${v}" class="${v === state.viz ? "on" : ""}">${v[0].toUpperCase() + v.slice(1)}${rec}</button>`;
  }).join("") + `<button data-viz="flow" class="disabled" title="Coming soon — flow diagrams launch in a later release.">Flow ⚡</button>`;

  content.innerHTML = `
    ${breadcrumb()}
    <div class="askpyze-stage">
      <h2>Here's what I found</h2>
      <div class="askpyze-vizpicker" id="viz-picker">${pickerHtml}</div>
      <div class="askpyze-viz-wrap" id="viz-wrap"></div>
      <div class="askpyze-actions">
        <button class="askpyze-btn-secondary" id="btn-back-plan">Back to plan</button>
        <button class="askpyze-btn-primary" id="btn-save">Save analysis</button>
      </div>
    </div>`;
  renderViz(content.querySelector("#viz-wrap"), state.viz, state.result);
  content.querySelector("#viz-picker").addEventListener("click", (e) => {
    const btn = e.target.closest("button"); if (!btn || btn.classList.contains("disabled")) return;
    state.viz = btn.dataset.viz; renderResult();
  });
  content.querySelector("#btn-back-plan").addEventListener("click", () => { state.stage = "plan"; render(); });
  content.querySelector("#btn-save").addEventListener("click", () => alert("Save flow — next task"));
}
```

- [ ] **Step 5: Add viz-inferViz tests to `tests.html`**

No change needed; existing tests cover it. Just verify at `http://localhost:8000/tests.html` that they still pass.

- [ ] **Step 6: Manual verify**

Ask a real question. Result stage shows viz picker with Recommended pip on the Gemini-suggested type. Click other types — each renders (Number shows big number, Line renders with Chart.js, Bar renders bars, Table shows rows). Flow pill is greyed out and unclickable.

- [ ] **Step 7: Commit**

```bash
git add index.html js/viz.js
git commit -m "feat(ui): add viz picker with Number/Line/Bar/Table renderers and Flow stub"
```

---

## Task 12: Save panel + localStorage write

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add save-panel CSS in `<style>`**

```css
.askpyze-savepanel { background: var(--paper-2); border: 1px solid var(--line); border-radius: 8px; padding: 20px; margin-top: 20px; }
.askpyze-savepanel label { display: flex; align-items: center; gap: 10px; font-family: "Geist", sans-serif; font-size: 13px; color: var(--ink-2); margin: 8px 0; cursor: pointer; }
.askpyze-savepanel input[type="text"] { width: 100%; padding: 10px 14px; font-family: "Geist", sans-serif; font-size: 14px; border: 1px solid var(--line-2); border-radius: 6px; outline: none; }
.askpyze-savepanel input[type="text"]:focus { border-color: var(--ember); }
.askpyze-savepanel .label-row { display: flex; align-items: baseline; gap: 8px; margin-bottom: 10px; }
.askpyze-savepanel .label-row span { font-family: "Geist Mono", monospace; font-size: 10px; letter-spacing: 0.08em; text-transform: uppercase; color: var(--muted); }
```

- [ ] **Step 2: Import `storage.js` helpers at the top of the script module**

```javascript
import { newRecord, upsert } from "/js/storage.js";
```

- [ ] **Step 3: Replace the `btn-save` alert with a real save panel**

Modify `renderResult`: add a save-panel section controlled by a `state.showSavePanel` flag. Replace the save button click with a function that toggles that flag.

```javascript
function deriveTitle(question) {
  // Simple heuristic: first 60 chars, stripped of trailing punctuation
  const t = question.trim().replace(/\s+/g, " ");
  return t.length > 60 ? t.slice(0, 57) + "…" : t;
}

// Add state fields: state.showSavePanel, state.titleDraft, state.pinToHome
// Initialize in reset(): state.showSavePanel = false; state.titleDraft = ""; state.pinToHome = false;

// Inside renderResult, add AFTER the .askpyze-viz-wrap div:
function renderSavePanel() {
  if (!state.titleDraft) state.titleDraft = deriveTitle(state.question);
  return `
    <div class="askpyze-savepanel">
      <div class="label-row"><span>Title</span></div>
      <input type="text" id="save-title" value="${state.titleDraft.replace(/"/g, "&quot;")}" />
      <label><input type="checkbox" checked disabled /> Save to My Analyses</label>
      <label><input type="checkbox" id="save-pinhome" ${state.pinToHome ? "checked" : ""} /> Also pin to Home</label>
    </div>`;
}
```

Update `renderResult` to:
- Always render the save panel (replace `alert` logic).
- Have two buttons: `Cancel` (closes) and `Save` (runs `onSave`).

```javascript
// Modify the .askpyze-actions block inside renderResult:
<div class="askpyze-actions">
  <button class="askpyze-btn-secondary" id="btn-back-plan">Back to plan</button>
  <button class="askpyze-btn-primary" id="btn-save">Save analysis</button>
</div>
```

```javascript
// Add AFTER the viz-picker click handler:
content.querySelector("#btn-save").addEventListener("click", () => {
  state.showSavePanel = true;
  renderResult();
});
```

Then render the save panel conditionally inside `renderResult` if `state.showSavePanel` is true, and replace the actions row accordingly:

```javascript
// Replace action buttons when save panel is showing:
if (state.showSavePanel) {
  content.querySelector("#viz-wrap").insertAdjacentHTML("afterend", renderSavePanel());
  const titleInput = content.querySelector("#save-title");
  titleInput.addEventListener("input", (e) => { state.titleDraft = e.target.value; });
  content.querySelector("#save-pinhome").addEventListener("change", (e) => { state.pinToHome = e.target.checked; });
  // Replace action row
  const actions = content.querySelector(".askpyze-actions");
  actions.innerHTML = `
    <button class="askpyze-btn-secondary" id="btn-cancel-save">Cancel</button>
    <button class="askpyze-btn-primary" id="btn-confirm-save">Save</button>`;
  content.querySelector("#btn-cancel-save").addEventListener("click", () => { state.showSavePanel = false; renderResult(); });
  content.querySelector("#btn-confirm-save").addEventListener("click", onSave);
}

function onSave() {
  const record = newRecord({
    question: state.question,
    plan: state.currentPlan,
    sql: state.sql,
    messages: state.messages,
    mode: state.mode,
    viz: state.viz,
    dataset: state.dataset,
    title: state.titleDraft.trim() || deriveTitle(state.question),
    result: state.result,
  });
  record.pinnedToHome = state.pinToHome;
  upsert(record);
  close();
}
```

- [ ] **Step 4: Manual verify**

Ask a question → see result → click "Save analysis" — save panel appears. Edit title. Toggle "Also pin to Home". Click Save — modal closes. Open DevTools → Application → Local Storage → `localhost:8000` → verify the `pyze.analyses` key contains your record with correct title/mode/viz/pinnedToHome.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(ui): add save panel and localStorage persistence"
```

---

## Task 13: Save animation — modal collapse + fly-to-sidebar (sidebar stub)

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

This task adds the basic fly-to-sidebar animation to wire up the "magic moment." The actual My Analyses page is built in later tasks, but we add a placeholder sidebar nav item now so the fly animation has a target.

- [ ] **Step 1: Add "My Analyses" sidebar nav item**

Find the sidebar nav section in `index.html` (search for the sidebar `<nav>` or the first nav-item icon). Add a new `<a>` or `<button>` with `id="nav-analyses"` styled like the other items, using a simple grid SVG icon.

Consult the existing nav item markup for styling. Minimum viable version:

```html
<a href="#analyses" class="nav-item" id="nav-analyses" data-view="analyses">
  <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6"><rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/></svg>
  <span>My Analyses</span>
</a>
```

- [ ] **Step 2: Add animation CSS**

```css
@keyframes pz-pulse { 0% { transform: scale(1); } 50% { transform: scale(1.15); background: var(--ember-wash); } 100% { transform: scale(1); } }
.nav-item.pulsing { animation: pz-pulse 600ms ease-out; }

.askpyze-fly-ghost { position: fixed; pointer-events: none; z-index: 2000; background: #fff; border: 1px solid var(--line); border-radius: 8px; box-shadow: 0 10px 30px rgba(15,29,74,.2); transform-origin: top left; transition: transform 480ms cubic-bezier(.5,0,.2,1), opacity 280ms ease; }
```

- [ ] **Step 3: Replace `onSave` with a version that animates before closing**

```javascript
async function onSave() {
  const record = newRecord({
    question: state.question, plan: state.currentPlan, sql: state.sql,
    messages: state.messages, mode: state.mode, viz: state.viz, dataset: state.dataset,
    title: state.titleDraft.trim() || deriveTitle(state.question), result: state.result,
  });
  record.pinnedToHome = state.pinToHome;
  upsert(record);
  await animateSave();
  close();
}

function animateSave() {
  return new Promise((resolve) => {
    if (matchMedia("(prefers-reduced-motion: reduce)").matches) { resolve(); return; }
    const vizWrap = content.querySelector("#viz-wrap");
    const target = document.getElementById("nav-analyses");
    if (!vizWrap || !target) { resolve(); return; }
    const sourceRect = vizWrap.getBoundingClientRect();
    const targetRect = target.getBoundingClientRect();
    const ghost = vizWrap.cloneNode(true);
    ghost.className = "askpyze-fly-ghost";
    ghost.style.left = sourceRect.left + "px";
    ghost.style.top = sourceRect.top + "px";
    ghost.style.width = sourceRect.width + "px";
    ghost.style.height = sourceRect.height + "px";
    document.body.appendChild(ghost);

    // Fade original modal
    document.querySelector(".askpyze-frame").style.transition = "opacity 280ms";
    document.querySelector(".askpyze-frame").style.opacity = "0";
    document.querySelector(".askpyze-scrim").style.transition = "opacity 400ms";
    document.querySelector(".askpyze-scrim").style.opacity = "0";

    requestAnimationFrame(() => {
      const sx = targetRect.width / sourceRect.width;
      const sy = targetRect.height / sourceRect.height;
      const s = Math.min(sx, sy, 0.08);
      const dx = (targetRect.left + targetRect.width / 2) - (sourceRect.left + sourceRect.width * s / 2);
      const dy = (targetRect.top + targetRect.height / 2) - (sourceRect.top + sourceRect.height * s / 2);
      ghost.style.transform = `translate(${dx}px, ${dy}px) scale(${s})`;
      ghost.style.opacity = "0.2";
    });
    setTimeout(() => {
      ghost.remove();
      target.classList.add("pulsing");
      setTimeout(() => target.classList.remove("pulsing"), 650);
      resolve();
    }, 500);
  });
}
```

- [ ] **Step 4: Manual verify**

Ask a question → save. The viz card should shrink and fly toward the My Analyses nav item. Nav item pulses once on arrival. Modal closes. The saved record is in localStorage.

Enable `prefers-reduced-motion` in DevTools → Rendering panel → verify the animation is skipped (modal just closes; nav still pulses? — acceptable).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(ui): add My Analyses sidebar nav + save fly-to-destination animation"
```

---

## Task 14: My Analyses page — canvas, empty state, page switching

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add CSS for the analyses view**

```css
#view-home, #view-analyses { display: none; }
#view-home.active, #view-analyses.active { display: block; }

.analyses-header { display: flex; align-items: baseline; justify-content: space-between; padding: 40px 48px 12px; }
.analyses-header h1 { font-family: "Instrument Serif", serif; font-size: 36px; font-weight: 400; margin: 0; color: var(--ink); }
.analyses-header .sub { font-family: "Geist", sans-serif; font-size: 14px; color: var(--muted); margin-top: 4px; }
.analyses-header .chip-btn { margin-top: 8px; }

.analyses-canvas { position: relative; min-height: calc(100vh - 140px); margin: 0 48px 48px; background: #fff; background-image: radial-gradient(circle, var(--line-2) 1px, transparent 1px); background-size: 24px 24px; border: 1px solid var(--line); border-radius: 8px; }
.analyses-empty { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; color: var(--muted); font-family: "Geist", sans-serif; }
.analyses-empty h3 { font-family: "Instrument Serif", serif; font-size: 22px; color: var(--ink-2); margin: 0 0 8px; font-weight: 400; }
```

- [ ] **Step 2: Wrap existing main content in `<div id="view-home" class="active">`**

Find the main container that holds KPI tiles, filter bar, process map, drill-downs, opportunities queue, and footer. Wrap it all in:

```html
<div id="view-home" class="active">
  <!-- existing home content -->
</div>
```

- [ ] **Step 3: Add the analyses view HTML AFTER `#view-home`**

```html
<div id="view-analyses">
  <div class="analyses-header">
    <div>
      <h1>My Analyses</h1>
      <div class="sub">Your saved questions, arranged however you like.</div>
    </div>
    <button class="chip-btn primary" id="btn-new-analysis"><svg viewBox="0 0 24 24"><path d="M12 5v14M5 12h14"/></svg>New analysis</button>
  </div>
  <div class="analyses-canvas" id="analyses-canvas">
    <div class="analyses-empty" id="analyses-empty">
      <h3>No analyses saved yet</h3>
      <p>Click "New analysis" above — or "Ask pyze" on Home — to get started.</p>
    </div>
  </div>
</div>
```

- [ ] **Step 4: Add page-switching JS (inside the existing module script)**

```javascript
function showView(name) {
  document.getElementById("view-home").classList.toggle("active", name === "home");
  document.getElementById("view-analyses").classList.toggle("active", name === "analyses");
  document.querySelectorAll(".nav-item").forEach((n) => n.classList.toggle("active", n.dataset.view === name));
  if (name === "analyses") renderAnalysesCanvas();
}

document.querySelectorAll(".nav-item").forEach((n) => {
  n.addEventListener("click", (e) => {
    e.preventDefault();
    showView(n.dataset.view || "home");
  });
});

// The existing home nav item also needs data-view="home". Find it and add.

document.getElementById("btn-new-analysis").addEventListener("click", open);

function renderAnalysesCanvas() {
  const canvas = document.getElementById("analyses-canvas");
  const empty = document.getElementById("analyses-empty");
  const records = JSON.parse(localStorage.getItem("pyze.analyses") || "[]");
  // Remove existing cards (but keep empty state element)
  canvas.querySelectorAll(".analysis-card").forEach((c) => c.remove());
  empty.style.display = records.length === 0 ? "flex" : "none";
  // Card rendering added in Task 15
}
```

- [ ] **Step 5: Update existing home nav item to include `data-view="home"`**

Find the home nav item in the sidebar and add `data-view="home"`. It should get `.active` class when shown (already handled by `showView`).

- [ ] **Step 6: Manual verify**

Reload. Click My Analyses in sidebar — existing home content disappears, My Analyses view appears with "No analyses saved yet". Click Home — home returns. Click "New analysis" — modal opens.

Save an analysis. Go to My Analyses — empty state still shows (cards render in next task) but localStorage has the record.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(ui): add My Analyses view scaffold with page switching and empty state"
```

---

## Task 15: Tile rendering on My Analyses canvas

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/js/viz.js`

- [ ] **Step 1: Add tile CSS in `<style>`**

```css
.analysis-card { position: absolute; background: #fff; border: 1px solid var(--line); border-radius: 8px; box-shadow: 0 2px 6px rgba(15,29,74,.06); overflow: hidden; display: flex; flex-direction: column; user-select: none; cursor: grab; }
.analysis-card.dragging { cursor: grabbing; box-shadow: 0 10px 24px rgba(15,29,74,.18); z-index: 100; }
.analysis-card .ac-head { padding: 12px 14px 6px; display: flex; align-items: flex-start; justify-content: space-between; gap: 8px; }
.analysis-card .ac-title { font-family: "Geist", sans-serif; font-size: 13px; font-weight: 500; color: var(--ink); line-height: 1.3; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
.analysis-card .ac-chip { font-family: "Geist Mono", monospace; font-size: 9px; letter-spacing: 0.08em; padding: 3px 7px; border-radius: 3px; flex: 0 0 auto; }
.analysis-card .ac-chip.live { background: var(--pyze-lime-wash); color: #4a6518; }
.analysis-card .ac-chip.snapshot { background: var(--paper-2); color: var(--muted); }
.analysis-card .ac-body { flex: 1 1 auto; padding: 8px 14px; min-height: 0; }
.analysis-card .ac-meta { display: flex; justify-content: space-between; padding: 6px 14px 10px; font-family: "Geist Mono", monospace; font-size: 10px; color: var(--muted); }

/* Card sizes per viz type */
.analysis-card.viz-number { width: 240px; height: 200px; }
.analysis-card.viz-line { width: 400px; height: 240px; }
.analysis-card.viz-bar { width: 400px; height: 240px; }
.analysis-card.viz-table { width: 400px; height: 280px; }
```

- [ ] **Step 2: Add tile rendering in the module script**

```javascript
import { loadAll } from "/js/storage.js";
import { render as renderViz } from "/js/viz.js";

function cascadePosition(existingCount) {
  return { x: 24 + existingCount * 24, y: 24 + existingCount * 24 };
}

function timeAgo(iso) {
  const s = Math.floor((Date.now() - new Date(iso).getTime()) / 1000);
  if (s < 60) return `${s}s ago`;
  if (s < 3600) return `${Math.floor(s/60)}m ago`;
  if (s < 86400) return `${Math.floor(s/3600)}h ago`;
  return new Date(iso).toLocaleDateString();
}

function buildCard(record) {
  const card = document.createElement("div");
  card.className = `analysis-card viz-${record.viz}`;
  card.dataset.id = record.id;
  const pos = record.position || cascadePosition(0);  // replaced per-record below
  card.style.left = pos.x + "px";
  card.style.top = pos.y + "px";
  const modeChip = `<span class="ac-chip ${record.mode === "live" ? "live" : "snapshot"}">${record.mode === "live" ? "LIVE" : "SNAPSHOT"}</span>`;
  const metaLeft = record.mode === "live"
    ? (record.lastRunAt ? `Updated ${timeAgo(record.lastRunAt)}` : "Not yet run")
    : `Saved ${new Date(record.createdAt).toLocaleDateString()}`;
  card.innerHTML = `
    <div class="ac-head">
      <div class="ac-title" title="${record.title.replace(/"/g,"&quot;")}">${record.title}</div>
      ${modeChip}
    </div>
    <div class="ac-body"><div class="ac-viz-wrap" style="height:100%;display:flex;align-items:center;justify-content:center;"></div></div>
    <div class="ac-meta"><span>${metaLeft}</span><span>${record.dataset}</span></div>`;
  const wrap = card.querySelector(".ac-viz-wrap");
  const data = record.mode === "snapshot" ? record.frozenResult : record.lastResult;
  if (data) renderViz(wrap, record.viz, data);
  else wrap.innerHTML = `<span style="color:var(--dim);font-size:11px;font-family:'Geist Mono',monospace;">—</span>`;
  return card;
}

function renderAnalysesCanvas() {
  const canvas = document.getElementById("analyses-canvas");
  const empty = document.getElementById("analyses-empty");
  canvas.querySelectorAll(".analysis-card").forEach((c) => c.remove());
  const records = loadAll();
  empty.style.display = records.length === 0 ? "flex" : "none";
  records.forEach((r, i) => {
    if (!r.position) r.position = cascadePosition(i);
    canvas.appendChild(buildCard(r));
  });
}
```

- [ ] **Step 3: Manual verify**

Save two analyses with different viz types (e.g., a Number and a Table). Navigate to My Analyses — both tiles render with correct sizes, titles, LIVE/SNAPSHOT chips, meta rows. Live metric without data shows "—".

- [ ] **Step 4: Commit**

```bash
git add index.html js/viz.js
git commit -m "feat(ui): render saved analyses as tiles on My Analyses canvas"
```

---

## Task 16: Hover toolbar + pencil (re-open follow-up mode)

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add hover-toolbar CSS**

```css
.analysis-card .ac-toolbar { position: absolute; top: 8px; right: 8px; display: flex; gap: 4px; opacity: 0; transition: opacity 180ms; z-index: 3; }
.analysis-card:hover .ac-toolbar { opacity: 1; }
.analysis-card .ac-toolbar button { background: rgba(255,255,255,0.85); backdrop-filter: blur(4px); border: 1px solid var(--line); border-radius: 4px; width: 26px; height: 26px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--ink-2); padding: 0; }
.analysis-card .ac-toolbar button:hover { background: #fff; border-color: var(--ember); color: var(--ember); }
.analysis-card .ac-toolbar svg { width: 14px; height: 14px; fill: none; stroke: currentColor; stroke-width: 1.6; stroke-linecap: round; }

.ac-kebab-menu { position: absolute; top: 40px; right: 8px; background: #fff; border: 1px solid var(--line); border-radius: 6px; box-shadow: 0 6px 16px rgba(15,29,74,.1); z-index: 5; display: none; min-width: 140px; font-family: "Geist", sans-serif; font-size: 13px; }
.ac-kebab-menu.open { display: block; }
.ac-kebab-menu button { display: block; width: 100%; text-align: left; padding: 8px 14px; border: none; background: none; font: inherit; color: var(--ink); cursor: pointer; }
.ac-kebab-menu button:hover { background: var(--paper-2); }
.ac-kebab-menu button.danger { color: var(--crimson); }
```

- [ ] **Step 2: Add toolbar inside `buildCard`** — append this inside the `card.innerHTML` template (just after the opening `<div class="ac-head">…</div>`):

```html
<div class="ac-toolbar">
  <button class="ac-edit" title="Re-open chat"><svg viewBox="0 0 24 24"><path d="M12 20h9"/><path d="M16.5 3.5a2.12 2.12 0 013 3L7 19l-4 1 1-4z"/></svg></button>
  <button class="ac-kebab" title="More"><svg viewBox="0 0 24 24"><circle cx="12" cy="5" r="1"/><circle cx="12" cy="12" r="1"/><circle cx="12" cy="19" r="1"/></svg></button>
</div>
<div class="ac-kebab-menu"></div>
```

- [ ] **Step 3: Wire pencil to re-open chat (placeholder — full follow-up mode in Task 22)**

Inside `buildCard`, after the template insertion:

```javascript
card.querySelector(".ac-edit").addEventListener("click", (e) => {
  e.stopPropagation();
  openFollowUp(record.id);
});
```

Define a placeholder:

```javascript
function openFollowUp(id) {
  // Placeholder: open modal with original state pre-loaded, full implementation in Task 22.
  const record = loadAll().find((r) => r.id === id);
  if (!record) return;
  reset();
  state.question = record.question;
  state.dataset = record.dataset;
  state.mode = record.mode;
  state.messages = record.messages;
  state.currentPlan = record.plan;
  state.sql = record.sql;
  state.result = record.mode === "snapshot" ? record.frozenResult : record.lastResult;
  state.viz = record.viz;
  state.recommendedViz = record.viz;
  state.stage = "result";
  modal.hidden = false;
  document.body.style.overflow = "hidden";
  render();
}
```

- [ ] **Step 4: Manual verify**

Hover a tile — pencil + kebab appear. Click pencil — modal re-opens at Result stage with the saved viz and data shown.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(ui): add hover toolbar with pencil to re-open saved analysis in chat"
```

---

## Task 17: Kebab menu actions (Duplicate, Delete, Pin/Unpin to Home)

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Populate the kebab menu and wire up actions inside `buildCard`**

```javascript
const menu = card.querySelector(".ac-kebab-menu");
function kebabItems(r) {
  const pinLabel = r.pinnedToHome ? "Unpin from Home" : "Pin to Home";
  return `
    <button data-action="duplicate">Duplicate</button>
    <button data-action="pin">${pinLabel}</button>
    <button data-action="delete" class="danger">Delete</button>`;
}
menu.innerHTML = kebabItems(record);

card.querySelector(".ac-kebab").addEventListener("click", (e) => {
  e.stopPropagation();
  document.querySelectorAll(".ac-kebab-menu.open").forEach((m) => { if (m !== menu) m.classList.remove("open"); });
  menu.classList.toggle("open");
});

menu.addEventListener("click", (e) => {
  e.stopPropagation();
  const action = e.target.dataset.action; if (!action) return;
  menu.classList.remove("open");
  if (action === "duplicate") {
    const copy = duplicate(record.id);
    if (copy) { copy.position = null; upsert(copy); renderAnalysesCanvas(); }
  } else if (action === "delete") {
    if (confirm("Delete this analysis?")) { remove(record.id); renderAnalysesCanvas(); syncHomePinned(); }
  } else if (action === "pin") {
    record.pinnedToHome = !record.pinnedToHome;
    upsert(record);
    renderAnalysesCanvas();
    syncHomePinned();
  }
});
```

Import `duplicate` and `remove` at the top of the module:

```javascript
import { loadAll, upsert, remove, duplicate, newRecord } from "/js/storage.js";
```

Add an empty `syncHomePinned()` stub (real implementation in Task 20):

```javascript
function syncHomePinned() { /* implemented in Task 20 */ }
```

- [ ] **Step 2: Close kebab when clicking outside**

```javascript
document.addEventListener("click", () => {
  document.querySelectorAll(".ac-kebab-menu.open").forEach((m) => m.classList.remove("open"));
});
```

- [ ] **Step 3: Manual verify**

Hover a tile → click kebab → menu appears with Duplicate / Pin to Home / Delete. Click Duplicate — a new tile appears with " (copy)" suffix. Click Pin to Home — localStorage record updates (`pinnedToHome: true`); menu label flips to Unpin. Click Delete → confirm → tile removed. Click outside an open menu — menu closes.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ui): add kebab menu with Duplicate/Pin/Delete actions"
```

---

## Task 18: Drag-to-move with position persistence

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add drag logic inside `buildCard`**

```javascript
let dragStart = null;
card.addEventListener("mousedown", (e) => {
  // Ignore clicks on toolbar / menu
  if (e.target.closest(".ac-toolbar, .ac-kebab-menu")) return;
  dragStart = { x: e.clientX, y: e.clientY, left: parseInt(card.style.left), top: parseInt(card.style.top) };
  card.classList.add("dragging");
  // Bring to front
  const maxZ = Math.max(0, ...[...document.querySelectorAll(".analysis-card")].map((c) => parseInt(getComputedStyle(c).zIndex) || 0));
  card.style.zIndex = maxZ + 1;
});

document.addEventListener("mousemove", (e) => {
  if (!dragStart || !card.classList.contains("dragging")) return;
  const dx = e.clientX - dragStart.x;
  const dy = e.clientY - dragStart.y;
  card.style.left = Math.max(0, dragStart.left + dx) + "px";
  card.style.top = Math.max(0, dragStart.top + dy) + "px";
});

document.addEventListener("mouseup", () => {
  if (!card.classList.contains("dragging")) return;
  card.classList.remove("dragging");
  dragStart = null;
  // Persist position
  record.position = { x: parseInt(card.style.left), y: parseInt(card.style.top) };
  upsert(record);
});
```

- [ ] **Step 2: Manual verify**

Drag a tile around the canvas — follows cursor smoothly. Release — tile stays. Reload page — tile in same position. Drag another tile on top of the first — most recent drag z-index wins.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(ui): add drag-to-move with position persistence on My Analyses canvas"
```

---

## Task 19: Live-metric refresh on page load

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add shimmer CSS**

```css
.ac-viz-wrap.refreshing { position: relative; overflow: hidden; }
.ac-viz-wrap.refreshing::after { content: ""; position: absolute; inset: 0; background: linear-gradient(90deg, transparent, rgba(254,72,17,0.08), transparent); animation: pz-shimmer 1.4s ease-in-out infinite; }
@keyframes pz-shimmer { 0% { transform: translateX(-100%); } 100% { transform: translateX(100%); } }
```

- [ ] **Step 2: Update `renderAnalysesCanvas` to fire refreshes after cards mount**

```javascript
function renderAnalysesCanvas() {
  const canvas = document.getElementById("analyses-canvas");
  const empty = document.getElementById("analyses-empty");
  canvas.querySelectorAll(".analysis-card").forEach((c) => c.remove());
  const records = loadAll();
  empty.style.display = records.length === 0 ? "flex" : "none";
  records.forEach((r, i) => {
    if (!r.position) { r.position = cascadePosition(i); upsert(r); }
    canvas.appendChild(buildCard(r));
  });
  // Fire parallel refreshes for live metrics
  records.filter((r) => r.mode === "live").forEach(refreshLive);
}

async function refreshLive(record) {
  const card = document.querySelector(`.analysis-card[data-id="${record.id}"]`);
  if (!card) return;
  const wrap = card.querySelector(".ac-viz-wrap");
  wrap.classList.add("refreshing");
  try {
    const res = await api.run(record.sql, record.dataset);
    record.lastResult = res;
    record.lastRunAt = new Date().toISOString();
    upsert(record);
    renderViz(wrap, record.viz, res);
    // Update meta line
    const meta = card.querySelector(".ac-meta span:first-child");
    if (meta) meta.textContent = `Updated ${timeAgo(record.lastRunAt)}`;
  } catch (e) {
    wrap.innerHTML = `<span style="color:var(--crimson);font-size:11px;font-family:'Geist Mono',monospace;" title="${e.message.replace(/"/g,"&quot;")}">Error</span>`;
  } finally {
    wrap.classList.remove("refreshing");
  }
}
```

- [ ] **Step 3: Manual verify**

Save a live metric. Reload the page. Go to My Analyses — tile briefly shimmers, then viz updates with fresh data. Meta line says "Updated 1s ago". Save a snapshot — tile does NOT refresh on reload (shows original saved value).

Simulate network failure: kill the FastAPI server while on My Analyses and force a refresh (reload tab). Live tiles show "Error" in red.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ui): auto-refresh live metrics on My Analyses page load with shimmer state"
```

---

## Task 20: Home integration — inject pinned live metrics into KPI row

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add pinned-tile CSS that visually matches existing KPI tiles**

First, find the existing KPI tiles markup in the home view (it's near the top of `#view-home`). Identify the container class (e.g., `.kpi-row` or similar). Add an id `id="kpi-row"` to it if it doesn't have one, so JS can reference it.

Add CSS for pinned wrappers:

```css
.kpi-row .kpi-divider { width: 1px; height: 60%; background: var(--line-2); margin: 0 8px; align-self: center; }
.kpi-row .pinned-tile { /* copy of existing .kpi-tile style — duplicate the existing selector's properties */ }
.kpi-row .pinned-tile .pt-title { font-family: "Geist Mono", monospace; font-size: 10px; letter-spacing: .08em; text-transform: uppercase; color: var(--muted); }
.kpi-row .pinned-tile .pt-value { font-family: "Instrument Serif", serif; font-size: 32px; color: var(--ink); line-height: 1; margin-top: 6px; }
.kpi-row .pinned-tile .pt-menu-btn { position: absolute; top: 8px; right: 8px; opacity: 0; transition: opacity 180ms; background: none; border: none; cursor: pointer; color: var(--muted); }
.kpi-row .pinned-tile:hover .pt-menu-btn { opacity: 1; }
```

(Exact properties to mirror the existing KPI tile are project-specific — copy from the existing `.kpi-tile` or equivalent selector in `index.html`.)

- [ ] **Step 2: Implement `syncHomePinned` to inject/update pinned tiles**

```javascript
function syncHomePinned() {
  const row = document.getElementById("kpi-row");
  if (!row) return;
  // Remove existing pinned tiles + divider
  row.querySelectorAll(".pinned-tile, .kpi-divider").forEach((n) => n.remove());
  const pinned = loadAll().filter((r) => r.pinnedToHome && r.mode === "live");
  if (pinned.length === 0) return;
  const divider = document.createElement("div");
  divider.className = "kpi-divider";
  row.appendChild(divider);
  pinned.forEach((r) => row.appendChild(buildPinnedTile(r)));
  // Fire refreshes
  pinned.forEach(refreshPinned);
}

function buildPinnedTile(record) {
  const tile = document.createElement("div");
  tile.className = "kpi-tile pinned-tile"; // reuse existing class for visual consistency
  tile.dataset.id = record.id;
  tile.style.position = "relative";
  const data = record.lastResult;
  const value = data?.rows?.[0]?.[0] ?? "—";
  tile.innerHTML = `
    <button class="pt-menu-btn" title="Unpin from Home"><svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.6"><circle cx="12" cy="5" r="1"/><circle cx="12" cy="12" r="1"/><circle cx="12" cy="19" r="1"/></svg></button>
    <div class="pt-title">${record.title}</div>
    <div class="pt-value">${typeof value === "number" ? value.toLocaleString() : value}</div>
    <div class="pt-meta" style="font-family:'Geist Mono',monospace;font-size:10px;color:var(--muted);margin-top:6px;">Updated ${record.lastRunAt ? timeAgo(record.lastRunAt) : "—"}</div>`;
  tile.querySelector(".pt-menu-btn").addEventListener("click", (e) => {
    e.stopPropagation();
    record.pinnedToHome = false;
    upsert(record);
    syncHomePinned();
    renderAnalysesCanvas();
  });
  return tile;
}

async function refreshPinned(record) {
  try {
    const res = await api.run(record.sql, record.dataset);
    record.lastResult = res;
    record.lastRunAt = new Date().toISOString();
    upsert(record);
    const tile = document.querySelector(`.pinned-tile[data-id="${record.id}"]`);
    if (!tile) return;
    const value = res.rows?.[0]?.[0] ?? "—";
    tile.querySelector(".pt-value").textContent = typeof value === "number" ? value.toLocaleString() : value;
    tile.querySelector(".pt-meta").textContent = `Updated ${timeAgo(record.lastRunAt)}`;
  } catch (e) { /* silent on Home */ }
}

// Call on page load:
syncHomePinned();
```

- [ ] **Step 3: Manual verify**

Save a LIVE metric with `Also pin to Home` checked. Go to Home — the new tile appears after a small divider at the end of the KPI row. Click the kebab on the tile — tile disappears from Home, My Analyses record's `pinnedToHome` flips to false. Re-pin from My Analyses kebab menu → tile reappears on Home. Reload — tile is still there, shows fresh `Updated Ns ago`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ui): inject pinned live metrics into Home KPI row"
```

---

## Task 21: Pinned Snapshots strip on Home

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add the Pinned Snapshots strip HTML below the KPI row**

Find the element right after the KPI row container (likely the filter bar). Insert this directly after the KPI row:

```html
<div id="pinned-snapshots-strip" hidden>
  <div class="pss-header">
    <span class="pss-label">Pinned snapshots</span>
  </div>
  <div class="pss-list" id="pinned-snapshots-list"></div>
</div>
```

- [ ] **Step 2: Add CSS**

```css
#pinned-snapshots-strip { margin: 0 48px 12px; padding: 12px 0; border-top: 1px solid var(--line); border-bottom: 1px solid var(--line); }
#pinned-snapshots-strip[hidden] { display: none; }
.pss-header { display: flex; align-items: center; margin-bottom: 8px; }
.pss-label { font-family: "Geist Mono", monospace; font-size: 10px; letter-spacing: .08em; text-transform: uppercase; color: var(--muted); }
.pss-list { display: flex; gap: 12px; flex-wrap: wrap; }
.pss-card { width: 240px; height: 160px; background: #fff; border: 1px solid var(--line); border-radius: 8px; padding: 12px; display: flex; flex-direction: column; position: relative; }
.pss-card .pss-title { font-family: "Geist", sans-serif; font-size: 12px; color: var(--ink); font-weight: 500; display: -webkit-box; -webkit-line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; }
.pss-card .pss-value { flex: 1; display: flex; align-items: center; justify-content: center; font-family: "Instrument Serif", serif; font-size: 28px; color: var(--ink); }
.pss-card .pss-meta { font-family: "Geist Mono", monospace; font-size: 10px; color: var(--muted); }
.pss-card .pss-unpin { position: absolute; top: 6px; right: 6px; background: none; border: none; opacity: 0; cursor: pointer; color: var(--muted); }
.pss-card:hover .pss-unpin { opacity: 1; }
```

- [ ] **Step 3: Replace `syncHomePinned` (from Task 20) with the merged version and add `buildSnapshotCard`**

Find the existing `syncHomePinned` function (added in Task 20) and replace it entirely with:

```javascript
function syncHomePinned() {
  // KPI row (live metrics)
  const row = document.getElementById("kpi-row");
  if (row) {
    row.querySelectorAll(".pinned-tile, .kpi-divider").forEach((n) => n.remove());
    const live = loadAll().filter((r) => r.pinnedToHome && r.mode === "live");
    if (live.length > 0) {
      const divider = document.createElement("div");
      divider.className = "kpi-divider";
      row.appendChild(divider);
      live.forEach((r) => row.appendChild(buildPinnedTile(r)));
      live.forEach(refreshPinned);
    }
  }
  // Snapshots strip
  const strip = document.getElementById("pinned-snapshots-strip");
  const list = document.getElementById("pinned-snapshots-list");
  if (strip && list) {
    list.innerHTML = "";
    const snaps = loadAll().filter((r) => r.pinnedToHome && r.mode === "snapshot");
    strip.hidden = snaps.length === 0;
    snaps.forEach((r) => list.appendChild(buildSnapshotCard(r)));
  }
}

function buildSnapshotCard(record) {
  const card = document.createElement("div");
  card.className = "pss-card";
  card.dataset.id = record.id;
  const data = record.frozenResult;
  const value = data?.rows?.[0]?.[0] ?? "—";
  card.innerHTML = `
    <button class="pss-unpin" title="Unpin from Home"><svg viewBox="0 0 24 24" width="12" height="12" fill="none" stroke="currentColor" stroke-width="1.6"><line x1="6" y1="6" x2="18" y2="18"/><line x1="6" y1="18" x2="18" y2="6"/></svg></button>
    <div class="pss-title">${record.title}</div>
    <div class="pss-value">${typeof value === "number" ? value.toLocaleString() : value}</div>
    <div class="pss-meta">Saved ${new Date(record.createdAt).toLocaleDateString()} · ${record.dataset}</div>`;
  card.querySelector(".pss-unpin").addEventListener("click", (e) => {
    e.stopPropagation();
    record.pinnedToHome = false;
    upsert(record);
    syncHomePinned();
    renderAnalysesCanvas();
  });
  return card;
}
```

- [ ] **Step 4: Confirm `syncHomePinned()` is called on page load** (added in Task 20).

- [ ] **Step 5: Manual verify**

Save a SNAPSHOT with `Also pin to Home` checked. Go to Home — Pinned snapshots strip appears between the KPI row and the filter bar. Strip shows the snapshot card with its frozen value. Click ✕ on the card — card disappears, strip collapses when empty. Save a second snapshot, pin it — both cards show.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(ui): add Pinned Snapshots strip on Home"
```

---

## Task 22: Follow-up mode

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add follow-up state fields in `state`**

```javascript
// In reset():
state.followUpOf = null;  // id of the record being followed-up, or null
```

- [ ] **Step 2: Replace `openFollowUp` with a fuller version**

```javascript
function openFollowUp(id) {
  const record = loadAll().find((r) => r.id === id);
  if (!record) return;
  reset();
  state.followUpOf = record.id;
  state.question = record.question;
  state.dataset = record.dataset;
  state.mode = record.mode;
  state.messages = record.messages.slice();
  state.currentPlan = record.plan;
  state.sql = record.sql;
  state.result = record.mode === "snapshot" ? record.frozenResult : record.lastResult;
  state.viz = record.viz;
  state.recommendedViz = record.viz;
  state.correctionCount = 0;  // reset — allow follow-up corrections
  state.stage = "result";
  modal.hidden = false;
  document.body.style.overflow = "hidden";
  render();
}
```

- [ ] **Step 3: Modify `renderResult` to show follow-up-specific controls when `state.followUpOf` is set**

Add at the top of `renderResult`, inside the `.askpyze-stage` div (before the viz picker):

```javascript
const followUpBanner = state.followUpOf ? `
  <div style="background:var(--ember-wash);border-left:3px solid var(--ember);padding:10px 14px;margin-bottom:16px;border-radius:4px;">
    <div style="font-family:'Geist Mono',monospace;font-size:10px;letter-spacing:.08em;text-transform:uppercase;color:var(--ember);">Following up on</div>
    <div style="font-family:'Geist',sans-serif;font-size:13px;color:var(--ink);margin-top:4px;">${state.question}</div>
  </div>` : "";
```

Inject `${followUpBanner}` into the content.

Modify the save panel — if `state.followUpOf` is set, add a radio:

```javascript
// Inside renderSavePanel, before the close of .askpyze-savepanel:
const followupRadio = state.followUpOf ? `
  <div class="label-row" style="margin-top:16px;"><span>Save mode</span></div>
  <label><input type="radio" name="save-mode" value="replace" checked /> Replace this analysis</label>
  <label><input type="radio" name="save-mode" value="new" /> Save as a new analysis</label>
` : "";
return `<div class="askpyze-savepanel">...${followupRadio}</div>`;
```

- [ ] **Step 4: Modify `onSave` to handle replace vs new**

```javascript
function onSave() {
  const saveAsNew = state.followUpOf ? (content.querySelector('input[name="save-mode"]:checked')?.value === "new") : true;
  if (state.followUpOf && !saveAsNew) {
    // Replace in place
    const existing = loadAll().find((r) => r.id === state.followUpOf);
    if (existing) {
      Object.assign(existing, {
        question: state.question, plan: state.currentPlan, sql: state.sql,
        messages: state.messages, mode: state.mode, viz: state.viz, dataset: state.dataset,
        title: state.titleDraft.trim() || deriveTitle(state.question),
        frozenResult: state.mode === "snapshot" ? state.result : null,
        lastResult: state.mode === "live" ? state.result : null,
        lastRunAt: state.mode === "live" ? new Date().toISOString() : existing.lastRunAt,
        updatedAt: new Date().toISOString(),
      });
      upsert(existing);
    }
  } else {
    const record = newRecord({
      question: state.question, plan: state.currentPlan, sql: state.sql,
      messages: state.messages, mode: state.mode, viz: state.viz, dataset: state.dataset,
      title: state.titleDraft.trim() || deriveTitle(state.question), result: state.result,
    });
    record.pinnedToHome = state.pinToHome;
    upsert(record);
  }
  animateSave().then(() => { close(); if (document.getElementById("view-analyses").classList.contains("active")) renderAnalysesCanvas(); syncHomePinned(); });
}
```

- [ ] **Step 5: Add an "Ask a follow-up" button in the Result stage**

In `renderResult`, add BEFORE the Back to plan button:

```html
<button class="askpyze-btn-secondary" id="btn-followup" style="display:${state.followUpOf ? "inline-flex" : "none"};">Ask a follow-up</button>
```

Actually simpler: whenever we are in follow-up mode, show a textarea inside the result stage to type a follow-up question. For v1 prototype, a minimal version: if `state.followUpOf` is set, show a text input above the save panel:

```html
<div style="margin-top:12px;">
  <input type="text" id="followup-input" placeholder="Ask a follow-up or refine the plan…" class="askpyze-q-textarea" style="min-height:auto;padding:10px 14px;font-size:14px;" />
  <button class="askpyze-btn-secondary" id="btn-followup-submit" style="margin-top:8px;">Refine</button>
</div>
```

Wire to call `onCorrectPlan` with the typed text; on success, re-run SQL and show updated result.

(Note: full follow-up conversation flow is a v1.5 polish. For v1 prototype, re-opening pre-populates state and the user can save a Replace or New. Inline text-refine is a stretch goal.)

- [ ] **Step 6: Manual verify**

Save an analysis. Click pencil on a tile — modal re-opens at Result stage with a "Following up on …" banner. Save with `Replace this analysis` — the existing record is updated (check in localStorage, `updatedAt` changes). Save with `Save as a new analysis` — a new record appears.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(ui): follow-up mode — re-open saved analyses with replace/save-as-new"
```

---

## Task 23: prefers-reduced-motion fallback + final polish

**Files:**
- Modify: `/Users/namanchetanrajdev/Documents/pyze-hero/index.html`

- [ ] **Step 1: Add global reduced-motion CSS**

```css
@media (prefers-reduced-motion: reduce) {
  .askpyze-fly-ghost { transition: opacity 120ms !important; transform: scale(0.1) !important; }
  .nav-item.pulsing { animation: none; }
  .ac-viz-wrap.refreshing::after { animation: none; }
}
```

- [ ] **Step 2: Verify `animateSave` already has the reduced-motion guard (Task 13)**

Confirm: `if (matchMedia("(prefers-reduced-motion: reduce)").matches) { resolve(); return; }` at the top of the function.

- [ ] **Step 3: Manual verify**

DevTools → Rendering → `prefers-reduced-motion: reduce`. Save an analysis — modal closes near-instantly, no fly animation. Reload My Analyses — live tile refreshes without shimmer. Functionality otherwise unchanged.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ui): add prefers-reduced-motion fallback"
```

---

## Task 24: End-to-end verification

**No file changes — this is a human sanity check.**

- [ ] **Step 1: Full flow — live metric**

```bash
uvicorn server:app --reload
```

Navigate to `http://localhost:8000/`. Click "Ask pyze" in the opportunities queue header. Type: "How many cases are in CDD?". Click Analyze → review plan → click "Looks right — run it" → see the result. Pick Number viz (should be recommended). Click Save. Title should auto-fill. Check "Also pin to Home". Click Save — watch the fly animation. Go to My Analyses — tile is there. Go to Home — KPI row has the new tile. Reload — live tile refreshes on both pages.

- [ ] **Step 2: Full flow — snapshot**

Click "Ask pyze" again. Ask "What's the count of cases by stage?" → plan → run. Pick Bar viz. Switch Mode to Snapshot. Save with "Also pin to Home" checked. Verify it lands in the Pinned Snapshots strip on Home.

- [ ] **Step 3: Follow-up**

Go to My Analyses. Hover a tile → click pencil. Modal re-opens in follow-up mode. Choose "Save as a new analysis" → verify a duplicate-with-original-timestamp tile appears.

- [ ] **Step 4: Edge cases**

- Type a nonsense question ("asdfghjkl"). Verify graceful error handling (plan stage may show a generic Gemini response; retry retries; final error screen reached if validation exhausts).
- Ask a question referencing a non-existent column. Verify the SQL retry loop surfaces a Back to plan button eventually.
- Fill localStorage to 100+ saved analyses (via DevTools console: `for (let i = 0; i < 120; i++) { /* build record */ }`). Verify the canvas still renders and drag performance stays acceptable.
- Delete all analyses. Verify empty state appears.
- On the Home page, confirm: no tile is added by default, KPI row still shows the original 4 hand-authored tiles, opportunities queue, process map, drill-downs all render identically to before the feature.

- [ ] **Step 5: Final commit — release notes stub**

Nothing to commit unless issues were found in earlier tasks. Otherwise:

```bash
git log --oneline -20
```

Review the commit history. All tasks committed cleanly. Done.

---

## Self-Review Checklist

This section exists so the plan author (you, reading this later) can verify coverage.

- [x] **/plan endpoint** — Task 2.
- [x] **/plan/refine endpoint** — Task 3.
- [x] **/sql endpoint with VIZ extraction + retry** — Task 4.
- [x] **/run endpoint** — Task 5.
- [x] **Frontend module scaffolding + test harness** — Task 6.
- [x] **"Ask pyze" rename + modal shell** — Task 7.
- [x] **Question stage + dataset + mode picker** — Task 8.
- [x] **Plan stage with correction loop** — Task 9.
- [x] **SQL + run wiring, initial table render** — Task 10.
- [x] **Viz picker with Number/Line/Bar/Table + Flow stub** — Task 11.
- [x] **Save panel + localStorage write** — Task 12.
- [x] **Save fly-to-destination animation** — Task 13.
- [x] **My Analyses view with page switching + empty state** — Task 14.
- [x] **Tile rendering (anatomy, mode chip, meta line)** — Task 15.
- [x] **Hover toolbar + pencil (re-open chat)** — Task 16.
- [x] **Kebab menu (Duplicate, Pin/Unpin, Delete)** — Task 17.
- [x] **Drag-to-move with position persistence** — Task 18.
- [x] **Live-metric refresh on page load with shimmer** — Task 19.
- [x] **Home KPI row injection of pinned live metrics** — Task 20.
- [x] **Pinned Snapshots strip on Home** — Task 21.
- [x] **Follow-up mode (replace vs new)** — Task 22.
- [x] **prefers-reduced-motion fallback** — Task 23.
- [x] **End-to-end verification** — Task 24.

Spec sections all covered. No placeholders. Type names consistent (`record`, `state`, `api`, `renderViz`, `renderers`, `loadAll/upsert/remove/duplicate/newRecord`).

Gaps deliberately deferred to v2 (per spec's "Out of Scope" section): flow-diagram viz, hosting, multi-user, scheduled refresh, streaming Gemini, DB persistence, export/share, keyboard navigation, mobile layout, analytics.
