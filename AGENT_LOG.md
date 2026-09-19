# AGENT_LOG.md — Shared Group Chat for the Review Loop

> **What this is.** The single shared, append-only log for the nested
> Coder / Design-Critic / Performance-Architect loop defined in
> [`AGENT_ORCHESTRATION.md`](./AGENT_ORCHESTRATION.md). It is the group chat **and**
> the audit trail: every task, finding, fix, verdict, and state change lands here.
>
> **Read this first if you are an agent.** Find the **Current loop state**, then the
> **Open findings** index, then the most recent entries addressed to `@you`.
> Append your entry at the **bottom**. Never edit or delete earlier entries.
>
> **Exit condition:** a `performance-architect` entry with `VERDICT: PASS`.

---

## 1. Ground rules

1. **Append-only.** Corrections are new entries that reference the old one. Never rewrite history.
2. **One writer at a time.** The loop is strictly sequential, so appends never race.
3. **Every finding lives here** with an ID, severity, location, rationale, fix, and acceptance check. Chat messages are pointers; this file is the record.
4. **Machine-readable tokens.** Reviewers end with `VERDICT: PASS` or `VERDICT: CHANGES_REQUESTED`. The coder ends with `STATUS: DONE` or `STATUS: BLOCKED`. The orchestrator parses these lines.
5. **Stable IDs.** `DC-###` for design findings, `PA-###` for performance findings. Never reuse an ID.
6. **Ownership of state.** The orchestrator maintains the **Current loop state** block. Reviewers add and later resolve rows in the **Open findings** index.

---

## 2. Entry format

### 2.1 Header

```markdown
### [NNNN] YYYY-MM-DD HH:MMZ — <handle> — <TYPE>
```

| Field | Values |
| --- | --- |
| `NNNN` | Zero-padded, monotonically increasing sequence number. Never reuse. |
| `handle` | `orchestrator`, `coder`, `design-critic`, `performance-architect` |
| `TYPE` | `STATE`, `TASK`, `HANDOFF`, `FIX`, `FINDINGS`, `VERDICT`, `DISPUTE`, `BLOCKED`, `ACCEPT`, `DONE` |

### 2.2 Finding block

```markdown
- **DC-003** `MAJOR` — `src/submission.hpp:42`
  - **Issue:** <what is wrong>
  - **Why:** violates `DESIGN.md` D2 / `REQUIREMENTS.md` S2 / UB
  - **Fix:** <exact change to make>
  - **Acceptance:** <how to verify the fix>
```

### 2.3 Severity

| Severity | Meaning | Blocks `PASS`? |
| --- | --- | --- |
| `BLOCKER` | Wrong results, UB, race, broken contract. | Yes |
| `MAJOR` | Violates a design decision or performance requirement. | Yes |
| `MINOR` | Quality/maintainability issue with a concrete fix. | Yes |
| `NIT` | Optional polish. | **No** |

`PASS` requires **zero open `BLOCKER`/`MAJOR`/`MINOR`**. `NIT`s may remain open at exit.

---

## 3. Entry templates (copy these)

<details>
<summary><b>orchestrator — STATE / TASK / DONE</b></summary>

```markdown
### [NNNN] YYYY-MM-DD HH:MMZ — orchestrator — STATE
**Phase:** INITIAL | INNER:design-review | OUTER:performance-review
**Iteration:** inner=<n> outer=<n>
**Action:** <spawn/dispatch/forward>
**Routing:** <finding IDs> → @coder
**Budget:** MAX_INNER=<n> MAX_OUTER=<n>
```
</details>

<details>
<summary><b>coder — FIX</b></summary>

```markdown
### [NNNN] YYYY-MM-DD HH:MMZ — coder — FIX
**Phase:** INITIAL | INNER:design-review | OUTER:performance-review
**Iteration:** <n>
**Trigger:** <finding IDs or TASK description>
**Changes:**
- `src/submission.hpp` — <what changed and why, referencing D#/S#/G#>
**Findings addressed:**
- `DC-001` — FIXED: <one line>
- `PA-004` — DISPUTED: <evidence-backed reason>
**Evidence:**
- build: ok
- ctest: 5/5 pass
- benchmark: `{"runtime_ms": ..., "score": ...}` (if run)
**To:** @design-critic, @performance-architect
STATUS: DONE
```
</details>

<details>
<summary><b>design-critic — FINDINGS / VERDICT</b></summary>

```markdown
### [NNNN] YYYY-MM-DD HH:MMZ — design-critic — FINDINGS
**Phase:** INNER (design review)
**Iteration:** <n>
**Reviewed:** `src/submission.hpp` @ <git short sha or diff summary>
**Verdict:** CHANGES_REQUESTED | PASS
**Findings:**
- **DC-003** `MAJOR` — `src/submission.hpp:42`
  - **Issue:** <what is wrong>
  - **Why:** violates `DESIGN.md` D2 / `REQUIREMENTS.md` S2 / UB
  - **Fix:** <exact change to make>
  - **Acceptance:** <how to verify it is fixed>
**To:** @coder
VERDICT: PASS | CHANGES_REQUESTED
```
</details>

<details>
<summary><b>performance-architect — FINDINGS / VERDICT</b></summary>

```markdown
### [NNNN] YYYY-MM-DD HH:MMZ — performance-architect — FINDINGS
**Phase:** OUTER (performance review)
**Iteration:** <n>
**Build:** Release, `-O3 -march=x86-64-v3`, OpenMP=<on/off>, threads=<n>
**Correctness:** N/5 pass
**Benchmark:** runs=[.., .., ..]; median=<ms>; score=<x>; target=<x>
**Roofline:** traffic≈3.36 GB; achieved≈<GB/s>; best streaming≈<GB/s>; efficiency≈<n>%
**Verdict:** CHANGES_REQUESTED | PASS
**Findings:**
- **PA-002** `MAJOR` — `src/submission.hpp:57`
  - **Issue:** <measured problem>
  - **Why:** violates `PERFORMANCE.md` §7 / D9
  - **Evidence:** <command + output>
  - **Fix:** <exact change>
  - **Acceptance:** <re-measurement that confirms it>
**To:** @coder
VERDICT: PASS | CHANGES_REQUESTED
```
</details>

---

## 4. Current loop state

> Maintained by the **orchestrator**. Update in place (this is the one exception to
> append-only: it is a status board, not history).

| Field | Value |
| --- | --- |
| Run status | `RUNNING` |
| Phase | `INITIAL` |
| Inner iteration | `0` |
| Outer iteration | `0` |
| Last coder ref | `—` |
| Last design verdict | `—` |
| Last performance verdict | `—` |
| Next action | `coder: implement src/submission.hpp` |
| `SCORE_TARGET` | `3.0` |
| `ROOFLINE_EFFICIENCY` | `70%` |
| `MAX_INNER` / `MAX_OUTER` | `5` / `6` |
| Blockers | none |

---

## 5. Open findings index

> Add a row when a finding is raised; move it to **Closed** when the coder's fix is
> accepted by the raising reviewer (or the finding is `DISPUTED` and accepted).

| ID | Severity | Raised by | Location | Summary | Status |
| --- | --- | --- | --- | --- | --- |
| — | — | — | — | _no findings yet_ | — |

<details>
<summary>Closed findings</summary>

| ID | Severity | Resolution | Closed by |
| --- | --- | --- | --- |
| — | — | — | — |

</details>

---

## 6. The group chat

<!-- Append new entries below this line. Do not edit or reorder existing entries. -->

### [0001] 2025-01-01 00:00Z — orchestrator — STATE

**Phase:** INITIAL
**Iteration:** inner=0 outer=0
**Action:** Log initialised. Awaiting kickoff (`/agentic-loop`). No run in progress.
**Routing:** —
**Budget:** MAX_INNER=5 MAX_OUTER=6

> **Instructions to the orchestrator:** on kickoff, update the **Current loop state**
> table, then dispatch `coder` with the task and append a `TASK` entry. Route each
> reviewer's findings to `coder` and append a `STATE` entry per transition. Exit only
> on `performance-architect` `VERDICT: PASS`, then append a `DONE` entry.

### [0002] 2026-02-14 00:00Z — orchestrator — TASK

**Phase:** INITIAL
**Iteration:** inner=0 outer=0
**Action:** Run (re)started. All loop agents pinned to `deepseek/deepseek-flash`
(DeepSeek V4.1 Flash); previous attempt on the inherited Pro model was stopped.
Dispatching `coder` to implement `src/submission.hpp` end-to-end against
`REQUIREMENTS.md`, `DESIGN.md`, and `PERFORMANCE.md`.
**Routing:** —
**Budget:** MAX_INNER=5 MAX_OUTER=6
**Targets:** SCORE_TARGET=3.0, ROOFLINE_EFFICIENCY=70%

### [0003] 2026-09-19 17:18Z — coder — FIX

**Phase:** INITIAL
**Iteration:** inner=0 outer=0
**Trigger:** TASK [0002] — implement `src/submission.hpp` end-to-end.
**Changes:**
- `src/submission.hpp` — replaced the stub with the full implementation. One flat,
  row-major, 64-byte-aligned RAII buffer with `stride = round_up(cols, 8)` kept
  separate from `cols` (D1/D2/D3); rule-of-zero `Grid` preserving the declared
  constructor and both `operator()` overloads (D4); non-owning
  `ConstGridView`/`GridView` carrying pointer + shape (D5/D11); documented no-alias
  contract with `__restrict__` row pointers and debug-only identity plus
  pointer-range disjointness asserts (D6); verbatim boundary copy folded into one
  `#pragma omp parallel for schedule(static)` row loop, boundary rows via `memcpy`
  of exactly `cols` logical elements (D7); hoisted row bases and a SIMD-friendly
  inner loop with a canonical `j < last` bound required by `omp simd` (D8/D9);
  OpenMP over output rows with no hardcoded thread count (D10); `#pragma once`,
  `inline` helpers, no anonymous namespace (D12). Guards: early return for
  `rows == 0 || cols == 0`, `i + 1 == rows` boundary test (no unsigned underflow),
  overflow-checked padded stride and `rows * stride`.
**Findings addressed:**
- none raised yet — **Open findings** index is empty (INITIAL phase).
**Evidence:**
- build: ok (preset `-O3`; OpenMP 4.5 found and linked)
- ctest: 5/5 pass — `public/square-32`, `nonsquare-48x80`, `checker-64`,
  `one-step-50`, `nonsquare-boundary-80x50`, all `max_diff=0.000e+00`
- vectorization: `g++ -O3 -march=x86-64-v3 -fopenmp -fopt-info-vec-optimized` →
  `submission.hpp:263: optimized: loop vectorized using 32 byte vectors` (AVX2)
- alignment: base `% 64 == 0` for 1×1, 7×7, 31×31, 1000×1000, 1024×1024; stride 80
  for 48×80, 56 for 80×50, 8 for 1×1
- local edge matrix (outside the repo, 51 shape/step combinations incl. 1×1, 1×N,
  N×1, 2×2, 3×3, both non-square orientations, 1/3/10 steps): ALL PASS;
  `old_grid` unmodified (S3); uniform-field invariant holds
- ASan+UBSan: clean on harness `--check` and on the edge matrix
- TSan: **not usable for this toolchain** — GCC libgomp + TSan reports
  false-positive races even for a trivial correct ping-pong
  `#pragma omp parallel for` (minimal repro `/tmp/tsan_repro.cpp`: 3 races,
  exit 66), so the reports do not indicate a defect in this submission
- benchmark (preset `-O3`, default OMP, shared machine — other agent sessions
  running, load average 2–15): under contention score min 0.68 / median ≈1.7 /
  best 2.22. In an earlier idle window the same binary reached
  `runtime_ms≈52–55`, `score≈3.0–3.15` with 16 threads. Local figures are
  contention-limited; recommend the performance architect re-measure on idle
  hardware.
**To:** @design-critic, @performance-architect
STATUS: DONE
