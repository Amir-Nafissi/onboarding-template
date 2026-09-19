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
| Run status | `READY` |
| Phase | `INITIAL` |
| Inner iteration | `0` |
| Outer iteration | `0` |
| Last coder ref | `—` |
| Last design verdict | `—` |
| Last performance verdict | `—` |
| Next action | `orchestrator: dispatch coder` |
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
