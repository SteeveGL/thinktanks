---
title: "Day-Phase Handoff Schema"
status: "draft"
date: 2026-10-03
decision: "Per-slice schema with a DAG-input `depends_on` field, a test-contract deliverable, and a blocked-slice contract."
---

# Day-Phase Handoff Schema — Dual-Phase Multi-Agent Coding System

**Artifact type:** Design (schema + consumption contract)
**Phase:** Day phase output → Night phase input
**Target repo:** `c:\Users\Steeve\Sources\github.com\SteeveGL\brainstorms`

---

## 0. Design rationale (why this schema exists)

This schema is the **contract between the two phases** of the system.

- The **day phase** produces, per slice, a *test contract* (failing tests / assertions) that becomes the spec.
- The **night phase** consumes that test contract in a **deterministic closed loop per slice**: the local LLM writes code to satisfy the slice's test contract, a Docker executor runs the tests, failing output is fed back to the LLM, retries are capped, and an exhausted slice is marked **blocked** so the run refuses to advance.

Two environmental constraints drive the schema's shape:

1. **Ollama is a local LLM — cheap but slow.** Every model call costs wall-clock time. The schema therefore encodes a `deterministic_execution_procedure` so the loop can run deterministic steps (Docker, test runner, assertions) **without** calling the model, and it keeps `retry` budgets explicit so we never burn the model on hopeless iterations.
2. **Backend-first build strategy.** Backend slices are the critical path. The `priority` field carries the backend-first signal so the DAG executor schedules backend slices first.

The schema is **design-level**: no full implementation code, only schema and illustrative snippets.

---

## 1. The per-slice schema (field-by-field)

The schema is a single record — one slice. The inventory is an ordered collection of these records. The schema is serialized as **YAML** throughout this document (see §2 for the canonical format choice).

### 1.1 Field table

| Field | Type | Required | Drives | Purpose | How the night loop uses it |
|---|---|---|---|---|---|
| `slice_id` | string | yes | DAG | Stable, unique identifier for the slice. | Key used to load the record, to record results, and to reference from other slices' `depends_on`. |
| `title` | string | yes | — | Human-readable name of the slice. | Display only (logging, morning-review UI). |
| `description` | string | no | — | Free-text scope of the slice. | Context prompt filler for the LLM; not consumed by the loop. |
| `priority` | enum `[backend, frontend, infra, doc]` | yes | DAG | **Backend-first signal.** `backend` slices are scheduled first. | Executor sorts by priority then topologically; backend slices get execution slots first. |
| `depends_on` | list[string] | yes | DAG | **DAG input.** List of `slice_id`s that must be green before this slice runs. | Executor builds the DAG, enforces topological order, and blocks this slice until every listed slice is `green`. |
| `test_contract` | string (YAML block / fenced code) | yes | Loop | The **failing tests / assertions** the code must satisfy. This is the spec. | The loop's acceptance oracle. Tests are run in Docker; pass = success, fail = feedback to LLM. |
| `acceptance_criteria` | list[string] | yes | Loop | Human-readable, non-test requirements (edge cases, constraints). | Fed to the LLM as additional context; loop checks them only when tests cannot express them. |
| `files_to_create_or_modify` | list[object] | yes | Loop | Files the slice owns. Each object: `{ path, action: create\|modify\|delete }`. | LLM told which files to touch; loop verifies changes land in these paths. |
| `deterministic_execution_procedure` | string (numbered steps) | yes | Loop | Deterministic, **non-LLM** steps (Docker build, test command, assertions). | Loop runs these **without** calling the model; only test failures are escalated to the LLM. |
| `retry_budget` | integer | no (default 3) | Loop | Max feedback iterations for this slice. | Loop decrements on each failed iteration; when it hits 0, slice → `blocked`. |
| `risk_score` | integer 1–5 | yes | DAG | Confidence/risk estimate (5 = most risky). | High-risk slices are scheduled earlier and given the full `retry_budget`. |
| `status` | enum `[pending, running, green, blocked]` | yes | DAG, Loop | Lifecycle state. | Updated by the loop (§4). `blocked` refuses run advancement. |
| `last_error` | string | no | Blocked | Last test/loop error output. | Populated on each failure; final value kept when slice becomes `blocked`. |
| `retry_count` | integer | no | Blocked | Number of feedback iterations consumed. | Incremented each failed iteration; compared to `retry_budget`. |
| `failure_classification` | enum `[none, genuine_failure, possible_test_gaming]` | no | Blocked | Distinguishes real failure from test-gaming. | Set when slice becomes `blocked` (§5). |
| `gaming_notes` | string | no | Blocked | Analyst note explaining the classification. | Read during the morning review to judge whether the model gamed the tests. |

> **DAG note:** `depends_on` is the **only** field that feeds the execution DAG. `priority` and `risk_score` influence *ordering within* a topological level, but `depends_on` defines the edges. A cycle in `depends_on` is a schema error and must fail fast before the night run starts.

---

## 2. Canonical file format (YAML) + realistic example

**Format choice: YAML.** Rationale: the `test_contract` contains multi-line test code, and YAML's block scalars (`|` and `>`) express that more readably than JSON's escaped newlines. The example below is internally consistent: the failing tests in `test_contract` correspond to the slice's scope, and the `depends_on` correctly references a prior slice.

```yaml
# docs/day-phase-handoff-schema.md
# Canonical example: INVENTORY slice "persistence"
slice_id: inventory-persistence
title: Inventory persistence layer
description: >
  Durable storage for inventory items and quantity adjustments.
  Backend slice; must serialize item state and enforce non-negative
  quantity invariants before any presentation layer exists.
priority: backend
# --- DAG input ---
depends_on:
  - domain-models
# --- Test contract: these tests FAIL on a clean checkout and must PASS
#     after the night loop writes code. They ARE the spec. ---
test_contract: |
  # --- inventory_persistence_test.py ---
  def test_save_and_load_roundtrip():
      repo = InventoryRepository(sqlite_memory())
      item = Item(sku="ABC-1", quantity=10)
      repo.save(item)

      loaded = repo.get(sku="ABC-1")
      assert loaded is not None
      assert loaded.sku == "ABC-1"
      assert loaded.quantity == 10

  def test_decrement_below_zero_is_rejected():
      repo = InventoryRepository(sqlite_memory())
      repo.save(Item(sku="ABC-1", quantity=3))

      with pytest.raises(QuantityExceededError):
          repo.decrement(sku="ABC-1", amount=5)

  def test_missing_sku_returns_none():
      repo = InventoryRepository(sqlite_memory())
      assert repo.get(sku="DOES-NOT-EXIST") is None
acceptance_criteria:
  - "Quantity can never go negative; decrement past zero raises QuantityExceededError."
  - "Persistence survives a fresh repository instance (round-trip read after write)."
  - "Unknown SKU lookups return None, never raise."
  - "Schema/migration is idempotent across runs."
files_to_create_or_modify:
  - path: src/inventory/repository.py
    action: create
  - path: src/inventory/errors.py
    action: create
  - path: tests/test_inventory_persistence.py
    action: create
deterministic_execution_procedure: |
  1. Build the executor image: `docker build -f docker/Dockerfile.py . -t slice-runner`
  2. Run the slice tests: `docker run --rm -v $PWD:/work slice-runner pytest tests/test_inventory_persistence.py -q`
  3. Capture exit code: 0 = all contract tests pass; non-zero = failing assertions.
  4. On non-zero: capture stderr + failing assertion messages and feed to the LLM as feedback.
  5. Repeat 2–4 until pass or retry_budget exhausted.
retry_budget: 3
risk_score: 4
status: pending
# Blocked-slice fields (empty until a failure occurs):
retry_count: 0
last_error: ""
failure_classification: none
gaming_notes: ""
```

**Internal-consistency check:** The three failing tests assert (a) round-trip, (b) non-negative invariant, (c) missing-SKU handling. The `acceptance_criteria` enumerate the same invariants in prose, and `files_to_create_or_modify` points the LLM at exactly the module that must implement them. `depends_on: [domain-models]` is valid only if a `domain-models` slice exists and is `green`.

---

## 3. Fill-in template (what the day phase generates per slice)

The day phase emits one of these per slice. Blank fields are to be filled; required fields must never be left empty.

```yaml
# --- DAG identity ---
slice_id: "<snake_case_unique_id>"
title: "<human-readable title>"
description: |
  "<scope, edge cases, what is intentionally OUT of scope>"

# --- Ordering (DAG input) ---
priority: "<backend|frontend|infra|doc>"   # backend-first signal
depends_on:
  - "<slice_id that must be green first>"
  # - "another-dependency"

# --- Spec = failing tests (MUST fail on clean checkout) ---
test_contract: |
  "<actual failing test code / assertions the code must satisfy>"

acceptance_criteria:
  - "<non-test requirement / invariant>"
  - "<another invariant>"

# --- What the LLM is allowed to touch ---
files_to_create_or_modify:
  - path: "<relative/path/to/file.py>"
    action: "<create|modify|delete>"

# --- Deterministic, non-LLM steps (run without calling the model) ---
deterministic_execution_procedure: |
  1. "<docker / build step>"
  2. "<test-runner command>"
  3. "<how to capture pass/fail>"
  4. "<how to feed failures back>"

# --- Loop budgeting (Ollama is slow: keep this tight) ---
retry_budget: 3          # max feedback iterations
risk_score: 3            # 1 (safe) .. 5 (risky)

# --- Lifecycle (set by the night loop; day phase leaves these default) ---
status: pending          # pending -> running -> green | blocked
retry_count: 0
last_error: ""
failure_classification: none   # none | genuine_failure | possible_test_gaming
gaming_notes: ""
```

---

## 4. Night-loop consumption spec

The night loop reads the schema fields in three roles: **ordering**, **loop body**, and **lifecycle**.

### 4.1 Ordering (DAG)

- **Edges** come from `depends_on`. The executor topologically sorts all slices; a slice is *eligible* only when every entry in its `depends_on` is `green`.
- **Within a level**, slices are ordered by `priority` (`backend` first) then by `risk_score` (higher first).
- **Cycle detection** runs before the run starts; a cycle in `depends_on` aborts the run with a schema error.
- A slice whose dependency is `blocked` becomes `blocked` by propagation (the run refuses to advance).

### 4.2 Loop body (per eligible slice)

1. Load `test_contract`, `files_to_create_or_modify`, `deterministic_execution_procedure`.
2. Set `status: running`.
3. Run the **deterministic** steps (`deterministic_execution_procedure`) in Docker — **no model call**.
4. If tests pass → set `status: green`, record `last_error: ""`, exit slice.
5. If tests fail → capture stderr/assertions, build a feedback prompt, call the **model** (Ollama) to rewrite `files_to_create_or_modify`.
6. Increment `retry_count`. If `retry_count >= retry_budget` → go to §5 (blocked contract). Otherwise loop to step 3.

> **Ollama cost control:** steps 1–4 (Docker build, test execution) are deterministic and never call the model. The model is called **only** on a genuine test failure and only up to `retry_budget` times.

### 4.3 `status` lifecycle

```
pending ──▶ running ──▶ green        (success)
   │            │
   │            └────▶ blocked        (retries exhausted OR dependency blocked)
   └────────────────────────────────▶ blocked   (dependency blocked → propagation)
```

- `pending` → slice not yet scheduled.
- `running` → slice is in an active feedback iteration.
- `green` → all `test_contract` tests pass; slice unblocks dependents.
- `blocked` → retries exhausted or a dependency is blocked; **the run refuses to advance** and the slice surfaces in the morning review.

---

## 5. Blocked-slice contract

When a slice exhausts `retry_budget` (or a dependency is blocked), the schema records enough structured evidence that the **morning review** can decide unambiguously: *did the model genuinely fail, or did it game the tests?*

### 5.1 Fields recorded on blocking

| Field | Value when blocked | Why it matters |
|---|---|---|
| `status` | `blocked` | Run refuses to advance; gates the DAG. |
| `retry_count` | integer == `retry_budget` | Proves the full budget was consumed (not an early abort). |
| `last_error` | full stderr + failing assertion text from the **last** iteration | The raw signal the model could not satisfy. |
| `failure_classification` | `genuine_failure` **or** `possible_test_gaming` | The core morning-review decision. |
| `gaming_notes` | free-text analyst note | Explains *why* the classification was chosen. |

### 5.2 Classification criteria

- **`genuine_failure`** — the model produced code that still violates the `test_contract`; `last_error` shows real assertion failures / tracebacks the model could not resolve within budget.
- **`possible_test_gaming`** — the tests pass but the code clearly violates the `acceptance_criteria`, or `files_to_create_or_modify` were not actually changed, or tests were modified/trivialized. Signals the model satisfied the letter of the contract while breaking its spirit.

### 5.3 Example: a blocked slice record

```yaml
slice_id: inventory-persistence
title: Inventory persistence layer
priority: backend
depends_on:
  - domain-models
status: blocked
retry_count: 3
retry_budget: 3
last_error: |
  FAILED tests/test_inventory_persistence.py::test_decrement_below_zero_is_rejected
  AssertionError: expected QuantityExceededError, but decrement returned None
  src/inventory/repository.py:87: in decrement
failure_classification: genuine_failure
gaming_notes: >
  Model kept returning None on over-decrement across all 3 retries; no
  assertion was softened and no test file was modified. Looks like a real
  implementation gap, not gaming. Recommend manual implementation in morning.
```

### 5.4 Example: a test-gaming detection

```yaml
slice_id: inventory-persistence
status: blocked
retry_count: 3
retry_budget: 3
last_error: ""
failure_classification: possible_test_gaming
gaming_notes: >
  All contract tests pass, but acceptance_criteria "quantity can never go
  negative" is violated: decrement returns clamped 0 instead of raising.
  No test file was modified, yet behavior contradicts the spec. Review needed.
```

---

## 6. Validation rules (run before the night run)

- Every required field is present and non-empty.
- `priority` ∈ `{backend, frontend, infra, doc}`.
- `status` ∈ `{pending, running, green, blocked}`.
- `failure_classification` ∈ `{none, genuine_failure, possible_test_gaming}`.
- Every entry in `depends_on` references an existing `slice_id`.
- The `depends_on` graph is acyclic.
- If `status == blocked`, then `retry_count == retry_budget` and `failure_classification != none`.

---

## Output summary

| Deliverable | Location in this document |
|---|---|
| Field-by-field schema (table) | §1 |
| Canonical YAML format + realistic example | §2 |
| Fill-in template | §3 |
| Night-loop consumption spec | §4 |
| Blocked-slice contract | §5 |
| DAG input (`depends_on`) explicitly called out | §1.1 table, §4.1, §6 |
