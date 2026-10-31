# Complyance — Product Support Engineering Lab
## Assignment 2: Support Lifecycle Lab (Case PSE-A2-002)

**Candidate Submission** | **Priority:** P2 (High) | **Customer:** Cedar Demo Malaysia

---

## 1. Executive Summary & The Problem in Plain English

A customer, **Cedar Demo Malaysia**, uploaded a CSV containing **12 compliance records**. Shortly afterward, their dashboard displayed:
> **"Status: Complete"** — but only **7 results** were visible. 5 records had vanished.

Confused and blocked, the customer submitted an urgent support ticket asking:
> *"We uploaded 12 records. The import was accepted and now says complete, but we can see only seven results. We need to continue our work. Should we upload the file again?"*

### Why We Never Advise Blind Re-Upload:
The ingestion API (`POST /imports`) generates a new UUID for each upload and has no deduplication. If the customer re-uploaded the file, the 7 records that already processed successfully would be duplicated in their ledger, corrupting their tax/compliance reports. 

Instead, Product Support took ownership:
1. Advised the customer **not** to re-upload the file, offering a safe single-record workaround for genuinely new documents.
2. Verified the local database state and proved that the missing 5 records were safely retained in the database in a `pending` state.
3. Isolated the backend defect down to a single causal line of code.
4. Implemented a narrow, permanent patch and expanded test coverage across multiple volumes and batch configurations.
5. Safely reconciled the customer's existing data using `bun run resume-case`, recovering all 12 records with **zero duplicate data**.
6. Drafted customer communications and a Linear ticket for engineering release.

---

## 2. The Root Cause: Mutating SQL Pagination

This incident was caused by the classic distributed systems defect: **pagination over mutating state**.

### What the Code Did (`src/worker.ts`):
```typescript
let offset = 0;
while (true) {
  const batch = repository.pendingBatch(workspaceId, importId, batchSize, offset);
  if (batch.length === 0) break;
  repository.commitBatch(workspaceId, importId, batch);
  offset += batch.length; // <-- THE FATAL BUG
}
```

### What the Database Did (`src/repository.ts`):
1. `pendingBatch` queries:
   ```sql
   SELECT id, document_ref, label, state 
   FROM records 
   WHERE import_id = ? AND state = 'pending' 
   ORDER BY id 
   LIMIT ? OFFSET ?;
   ```
2. `commitBatch` immediately updates those rows:
   ```sql
   UPDATE records SET state = 'processed' WHERE id IN (...);
   ```

### Step-by-Step Breakdown of the Failure:
When Cedar Demo uploaded **12 records** with default `batchSize = 5`:

* **Batch 1 (`offset = 0, limit = 5`):**
  - Fetches records `[1, 2, 3, 4, 5]`.
  - Commits them: records 1–5 transition from `'pending'` to `'processed'`.
  - **The pending set in the database now SHRINKS from 12 rows to 7 rows:** `[6, 7, 8, 9, 10, 11, 12]`.
  - The worker increments: `offset = 0 + 5 = 5`.

* **Batch 2 (`offset = 5, limit = 5`):**
  - SQLite runs: `SELECT ... WHERE state = 'pending' LIMIT 5 OFFSET 5`.
  - Look at the remaining 7 pending rows: `[6, 7, 8, 9, 10, 11, 12]`.
  - `OFFSET 5` tells the database to **skip the first 5 pending rows** (`[6, 7, 8, 9, 10]`)!
  - It only returns the 6th and 7th items: records **`[11, 12]`**!
  - Commits records 11 and 12. Total processed so far: 5 + 2 = **7 records**.
  - The worker increments: `offset = 5 + 2 = 7`.

* **Batch 3 (`offset = 7, limit = 5`):**
  - Records `[6, 7, 8, 9, 10]` (5 records) are still pending in the table.
  - But SQLite runs with `OFFSET 7`. Since only 5 rows exist, `OFFSET 7` returns **0 rows**.
  - `batch.length === 0` evaluates to `true`. The loop exits!
  - The worker calls `finishImport()` and marks the job as **`completed`**!

**The Result:** The system processed 7 records, skipped 5 records, and falsely told the customer the import was complete.

---

## 3. The Permanent Causal Fix

When records transition out of the `'pending'` state, the queue naturally drains itself. **No offset is required.** The next batch of pending records is always sitting at index 0.

### The Fix in `src/worker.ts`:
```typescript
// Removed the offset accumulator entirely:
while (true) {
  const batch = repository.pendingBatch(workspaceId, importId, batchSize);
  if (batch.length === 0) break;
  repository.commitBatch(workspaceId, importId, batch);
}
```

### The Fix in `src/repository.ts`:
```typescript
// Made offset optional with default 0:
pendingBatch(workspaceId: WorkspaceId, id: string, limit: number, offset = 0): SourceRecord[]
```

Now:
- Batch 1 processes `1..5`. (Pending remaining: `6..12`).
- Batch 2 processes `6..10`. (Pending remaining: `11..12`).
- Batch 3 processes `11..12`. (Pending remaining: `0`).
- Batch 4 sees 0 pending $\rightarrow$ Clean completion with all 12 records processed!

---

## 4. Repository Layout

```text
complyance-pse-assignment-2/
├── CASE_REPORT.md            # Complete 6-section formal incident report
├── README.md                 # System overview, technical explanation & execution guide
├── package.json              # Project scripts & runtime dependencies
├── tsconfig.json             # Strict TypeScript configuration
├── vitest.config.ts          # Vitest runner configuration
├── fixtures/                 # Provided synthetic fixtures
│   ├── case-002.csv          # 12 customer records (DOC-001 to DOC-012)
│   └── control-002.csv       # 3 control records for Maple Demo
├── scripts/
│   ├── reset.ts              # Resets local database to initial state
│   └── resume-case.ts        # Approved lab reconciliation tool for case-002
├── src/                      # Application backend source code
│   ├── config.ts             # Runtime environment configuration
│   ├── csv.ts                # Streaming CSV validation & parsing
│   ├── database.ts           # SQLite WAL schema, foreign keys & indexes
│   ├── domain.ts             # Domain models, enums & error definitions
│   ├── repository.ts         # Fixed DB queries (optional offset = 0)
│   ├── seed.ts               # Initial demo fixtures seeder
│   ├── server.ts             # HTTP loopback routes & request handling
│   ├── service.ts            # Business logic orchestration
│   └── worker.ts             # Fixed background queue worker (offset-free batching)
├── tests/                    # Automated integration & regression test suite
│   ├── harness.ts            # Test subprocess manager
│   └── http.test.ts          # 31 comprehensive integration tests
└── evidence/                 # Empirical timestamped proof
    ├── baseline/             # Pre-fix API responses showing 7/12 records and 5 pending
    ├── tests/                # Raw test outputs (failing before fix, 100% passing after)
    └── recovery/             # Post-fix reconciliation responses (12/12 records, 0 duplicates)
```

---

## 5. Quickstart & Verification Commands

### Prerequisites
- **Bun** $\ge 1.3$ (Installed: `1.4.4`)
- **Node.js** $\ge 22$ (Installed: `v26.11.1`)

### 1. Install Dependencies
```sh
bun install --frozen-lockfile
```

### 2. Verify TypeScript Compilation
```sh
bun run check-types
```
*(Result: 0 errors).*

### 3. Run the Automated Test Suite
```sh
bun run test
```
*(Result: 31/31 passed in ~870ms).*

### 4. Run the Support Lab Server
```sh
bun run start
```
Starts on `http://127.0.0.1:4328`.

### 5. Reconcile Existing Customer Case
To reconcile the stuck `case-002` in an existing lab database:
```sh
bun run resume-case
```

---

## 6. Test Suite & Verification Matrix

The test suite in `tests/http.test.ts` was expanded from the 25 starter tests to **31 comprehensive integration tests**:

| Test Group | Coverage & Boundary Scenarios | Result |
| :--- | :--- | :--- |
| **Regression PSE-A2-002** | 12 records across multiple batches matching Cedar Demo's exact input volume | ✅ **Passed** |
| **Volume Scaling** | Parameterized test for `1, 5, 6, 11, 15, 20` records across batch boundaries | ✅ **Passed** |
| **Configurable Batch Sizes** | Custom server instance with `batchSize = 3` processing 7 records (`[3, 3, 1]`) | ✅ **Passed** |
| **API Validation** | 14 client error tests (empty, missing headers, broken quotes, bad references) | ✅ **Passed** |
| **Tenant Isolation** | Strict cross-workspace isolation (`maple-demo` cannot see `cedar-demo` $\rightarrow$ 404) | ✅ **Passed** |
| **Control Integrity** | Paused Maple Demo control (`control-002`) remains paused and untouched | ✅ **Passed** |
| **Persistence on Restart** | State, IDs, and events cleanly survive server restarts | ✅ **Passed** |

---

## 7. Evidence Index

All evidence files are version-controlled and directly inspectable:

- **Baseline Inspection (Before Fix):**
  - [01_health.json](evidence/baseline/01_health.json) — System health & active batch size 5
  - [02_cedar_summary.json](evidence/baseline/02_cedar_summary.json) — Cedar Demo showing `total: 12, processed: 7, pending: 5, state: "completed"`
  - [02_cedar_records.json](evidence/baseline/02_cedar_records.json) — Records 6–10 stuck in `pending`
  - [02_cedar_results.json](evidence/baseline/02_cedar_results.json) — Only 7 results generated
  - [02_cedar_events.json](evidence/baseline/02_cedar_events.json) — Audit trail proving Batch 2 jumped to rows 11–12
  - [03_maple_control.json](evidence/baseline/03_maple_control.json) — Maple Demo untouched in `paused` state
  - [04_cross_workspace_isolation.json](evidence/baseline/04_cross_workspace_isolation.json) — Cross-workspace read returning `HTTP 404`

- **Test Execution Evidence:**
  - [01_failing_regression_before_fix.txt](evidence/tests/01_failing_regression_before_fix.txt) — Failing test assertion (`expected 12 processed, received 7`)
  - [02_passing_regression_after_fix.txt](evidence/tests/02_passing_regression_after_fix.txt) — Passing test after removing offset
  - [03_full_suite_edge_cases_passing.txt](evidence/tests/03_full_suite_edge_cases_passing.txt) — All 31 tests passing cleanly

- **Post-Recovery Evidence (After Reconciliation):**
  - [01_resume_command_ack.json](evidence/recovery/01_resume_command_ack.json) — Requeuing acknowledgment
  - [02_cedar_summary_recovered.json](evidence/recovery/02_cedar_summary_recovered.json) — Cedar Demo showing `total: 12, processed: 12, pending: 0`
  - [02_cedar_records_recovered.json](evidence/recovery/02_cedar_records_recovered.json) — All 12 records marked `processed`
  - [02_cedar_results_recovered.json](evidence/recovery/02_cedar_results_recovered.json) — All 12 results present with **0 duplicates**
  - [02_cedar_events_recovered.json](evidence/recovery/02_cedar_events_recovered.json) — Second batch event committing records `[6, 7, 8, 9, 10]`
  - [03_maple_control_recovered.json](evidence/recovery/03_maple_control_recovered.json) — Maple control remaining safely `paused`

---

## 8. AI Assistance & Transparency

- **AI Tools Used:** Antigravity AI Assistant (Claude Opus 4.6 & Gemini 3.8 Flash).
- **Usage & Interaction:**
  - Acted as a senior engineering mentor and pair programmer.
  - Assisted with repository scaffolding, codebase walkthroughs, and framing customer communications according to the SIM-1 exercise clock.
  - Helped mathematically explain the mutating offset mechanism.
- **Candidate Oversight & Independence:**
  - The candidate actively controlled every phase and refused to "vibe code".
  - The candidate paused the workflow to demand a line-by-line breakdown of why records 6–10 were dropped to ensure deep technical mastery of the bug.
  - Every file edit, test assertion, and git commit was reviewed and verified locally before pushing.

---

## 9. Submission Checklist

- [x] Public GitHub repository: [valor0506/complyance-pse-assignment-2](https://github.com/valor0506/complyance-pse-assignment-2)
- [x] Clean default branch (`main`) with intentional, step-by-step git history
- [x] Starter application runnable from root (`bun install`, `bun run start`, `bun run test`)
- [x] Narrow, causal code fix in `src/worker.ts` & `src/repository.ts`
- [x] Automated Vitest regression test failing before fix, passing after fix
- [x] Boundary tests across multiple volumes (`1, 5, 6, 11, 15, 20`) and custom batch size 3
- [x] All 31 integration tests passing
- [x] Verified existing customer case reconciliation with zero duplicate data
- [x] Completed `CASE_REPORT.md` with customer comms drafts and Linear ticket
- [x] Small evidence files linked with relative paths
- [x] Full AI disclosure and technical limitations documented
