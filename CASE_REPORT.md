# Case report — PSE-A2-002

## 1. Summary and impact

- **Priority / rationale:** P2 (High / Partial Degradation). Fictional customer Cedar Demo Malaysia experienced an incomplete batch import where 12 records were accepted, the job was falsely reported as `completed`, but only 7 results were produced and visible. There is no evidence of a total platform outage, data corruption of completed records, cross-tenant data leakage, or security breach. Safe workaround is bounded to genuinely new single records; blind re-upload is prohibited to prevent duplicate processing.
- **Current status:** Baseline empirically verified in local support lab; causal defect isolated to mutating pagination in batch worker; awaiting regression test suite, permanent code patch, and lab case reconciliation.
- **Reported observation:** Customer stated: *"We uploaded 12 records. The import was accepted and now says complete, but we can see only seven results. We need to continue our work. Should we upload the file again?"* (Treated as an observation, not verified diagnosis).
- **Verified baseline:**
  - Lab health verified: `HTTP 200 OK`, `mode: "training"`, `batchSize: 5` ([evidence/baseline/01_health.json](evidence/baseline/01_health.json)).
  - Import summary: `total: 12`, `processed: 7`, `pending: 5`, `state: "completed"` ([evidence/baseline/02_cedar_summary.json](evidence/baseline/02_cedar_summary.json)).
  - Source records: Rows 1–5 (`DOC-001`–`DOC-005`) and rows 11–12 (`DOC-011`–`DOC-012`) marked `processed`. Rows 6–10 (`DOC-006`–`DOC-010`) remain orphaned in `pending` ([evidence/baseline/02_cedar_records.json](evidence/baseline/02_cedar_records.json)).
  - Results: Exactly 7 normalized results generated (`DOC-001`–`DOC-005`, `DOC-011`–`DOC-012`). Zero results for rows 6–10 ([evidence/baseline/02_cedar_results.json](evidence/baseline/02_cedar_results.json)).
  - Events: 2 batches executed (`[1,2,3,4,5]` count 5; `[11,12]` count 2), followed prematurely by `processing_completed` ([evidence/baseline/02_cedar_events.json](evidence/baseline/02_cedar_events.json)).
  - Control case: `maple-demo` / `control-002` remains intact with 3 records in `paused` state ([evidence/baseline/03_maple_control.json](evidence/baseline/03_maple_control.json)).
  - Workspace isolation: Querying Cedar's import under Maple workspace strictly returns `404 Not Found` ([evidence/baseline/04_cross_workspace_isolation.json](evidence/baseline/04_cross_workspace_isolation.json)).
- **Workaround / safety:** Customer instructed **not** to re-upload the file. Since `POST /imports` lacks deduplication, re-uploading would duplicate records 1–5 and 11–12 and distort ledger history. For urgent, genuinely new records, the single-record import flow may be used safely while existing records 6–10 are reconciled in place.

## 2. Draft customer communications

All messages below are drafts, not communications sent to a real customer. The simulated policy clock is separate from actual test timestamps and the real submission deadline.

### Acknowledgment — simulated 26 October 2026, 09:35 GST / 13:35 MYT

**Subject:** Update on your batch import inquiry [Case PSE-A2-002]

Hi Cedar Demo Malaysia Team,

Thank you for reaching out to Complyance Product Support.

We understand that your 12-record batch import shows as complete, but only seven results are currently visible in your workspace dashboard. We recognize this is holding up your processing workflow, and our team is actively investigating the discrepancy.

**Important immediate guidance:** Please **do not re-upload the file at this time**. Re-submitting the full file could create duplicate entries for the records that have already processed successfully. 

If you have urgent, **genuinely new single records** that were not part of this 12-record file, you may proceed with submitting them individually. 

I am personally taking ownership of this ticket. I will provide you with a concrete progress update within the next 25 minutes, by **10:00 GST / 14:00 MYT**, even if our investigation is still underway.

Best regards,  
Product Support Engineering, Complyance

---

### Checkpoint update — simulated 10:00 GST / 14:00 MYT

**Subject:** Progress Update: Investigating batch import [Case PSE-A2-002]

Hi Cedar Demo Malaysia Team,

Here is your scheduled progress update on case PSE-A2-002.

**What we have verified so far:**
- We inspected the backend state of your import (`case-002`). All 12 source records were successfully received and safely retained in our database.
- Exactly seven records (`DOC-001` through `DOC-005`, and `DOC-011` through `DOC-012`) have been fully processed with verified results. 
- The remaining five records (`DOC-006` through `DOC-010`) are currently in a pending state, though the overall job status prematurely indicated completion. None of your data has been lost.

**Current investigation & next steps:**
- We have identified the causal mechanism in our background batching logic that skipped these pending rows.
- We are currently verifying a fix in an isolated test environment and preparing a safe reconciliation procedure to process your five pending records without duplicating any previously generated results.
- Please continue to hold off on re-uploading the original file.

I will provide our next status update by **10:45 GST / 14:45 MYT**, or immediately upon completing our local test verification.

Best regards,  
Product Support Engineering, Complyance

---

### Final update / conditional closure — label your chosen simulated time

*(To be completed in Phase 5/6 following patch verification and lab reconciliation).*

## 3. Reproduction and root cause

- **Environment and steps:** *(To be completed in Phase 3/4)*
- **Expected versus observed:** *(To be completed in Phase 3/4)*
- **Causal explanation:** *(To be completed in Phase 3/4)*
- **Permanent change:** *(To be completed in Phase 3/4)*

## 4. Tests and existing-case reconciliation

| Check | Actual result and evidence |
| --- | --- |
| Regression: failure before, success after | *(Phase 3/4)* |
| Varied volumes, batch boundaries, different batch size | *(Phase 4)* |
| API validation and workspace isolation | Preserved: verified 404 on cross-workspace reads ([evidence/baseline/04_cross_workspace_isolation.json](evidence/baseline/04_cross_workspace_isolation.json)). |
| Original case before recovery | Verified: 12 total, 7 processed, 5 pending; results for 1–5, 11–12 ([evidence/baseline/02_cedar_summary.json](evidence/baseline/02_cedar_summary.json)). |
| Patched-worker restart and approved `resume-case` | *(Phase 5)* |
| Original case after recovery | *(Phase 5)* |
| Paused Maple control | Verified baseline: 3 records in paused state ([evidence/baseline/03_maple_control.json](evidence/baseline/03_maple_control.json)). |

## 5. Escalation and customer ownership

*(To be completed in Phase 5/6)*

## 6. Submission notes

- **Evidence index:**
  - [Baseline Health Check](evidence/baseline/01_health.json)
  - [Cedar Demo Case-002 Summary](evidence/baseline/02_cedar_summary.json)
  - [Cedar Demo Case-002 Source Records](evidence/baseline/02_cedar_records.json)
  - [Cedar Demo Case-002 Results](evidence/baseline/02_cedar_results.json)
  - [Cedar Demo Case-002 Event Log](evidence/baseline/02_cedar_events.json)
  - [Maple Demo Control-002 Summary](evidence/baseline/03_maple_control.json)
  - [Cross-Workspace Isolation 404 Check](evidence/baseline/04_cross_workspace_isolation.json)
- **AI assistance:** *(To be completed in Phase 6)*
- **Limitations:** *(To be completed in Phase 6)*
