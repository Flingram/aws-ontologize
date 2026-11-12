# Implementation Plan: Ontology & Strands Agent Showcase (Glycurex T001)

## Overview

This plan implements the showcase-first demo in the exact phase order defined by the design's Phase / Milestone Plan. Phase 0 (real Neptune connectivity) is first and gating: no F01–F10 business logic is built until the openCypher write/read smoke test succeeds against real Neptune. Each later phase builds incrementally on the prior one and ends in its verifiable milestone (A01–A11). The Phase 1b backend (HTTP API, API_Lambda, Cognito, S3+CloudFront, Recruitment Hub shell) is reused and extended — never rebuilt — by Phases 2–5.

Languages are fixed by the design: AWS CDK in **TypeScript** (one project under `/infra`), the three Lambdas, the worker, and the seed path in **Python**, and the frontend in **React + TypeScript + Cloudscape**. Repository layout: `/infra`, `/backend/api`, `/backend/graph`, `/backend/worker`, `/frontend`, `/seed`, `/tests`, `/README.md`.

Testing is showcase-first: focused unit tests over the deterministic core, a couple of essential Neptune integration tests, and end-to-end acceptance checks mapping A01–A11. No property-based testing, no fabricated accuracy numbers. Test sub-tasks are marked optional with `*`.

## Tasks

- [ ] 1. Phase 0 — Neptune connectivity (standalone, gating)
  - [ ] 1.1 Scaffold the single CDK TypeScript project and repository layout
    - Create `/infra` CDK app (TypeScript), `cdk.json` with `modelId` and `region` (default `eu-central-1`) context values, and empty `/backend/graph`, `/backend/api`, `/backend/worker`, `/frontend`, `/seed`, `/tests` directories
    - Wire `modelId`/region as CDK context consumed by stack code
    - _Requirements: 18.2, 19.8, 21.1_

  - [ ] 1.2 Define the VPC and Neptune Serverless cluster construct
    - Create a VPC with 2 private **isolated** subnets, **no NAT Gateway**, no IGW
    - Provision one private Neptune Database Serverless cluster (min 1.0 NCU, max ~2.5–4.0 NCU), IAM database auth **enabled**, security group locked to the Graph_Lambda SG on port 8182
    - Set `RemovalPolicy.DESTROY` and skip final snapshot
    - _Requirements: 19.3, 19.4, 19.6, 20.2, 20.3, 21.3_

  - [ ] 1.3 Deploy Graph_Lambda into the VPC with the smoke-test operation
    - Create the Python Graph_Lambda in `/backend/graph`, attached to the VPC private isolated subnets, with `neptune-db:*` IAM permissions
    - Add a CDK log group with retention and `RemovalPolicy.DESTROY`
    - Add Graph_Lambda to the CDK stack as the only SG allowed to reach Neptune on 8182
    - _Requirements: 1.2, 19.4, 20.3, 21.3_

  - [ ] 1.4 Implement the openCypher Neptune connection and `smokeTest` operation
    - In `/backend/graph`, implement TLS + SigV4 IAM-authenticated openCypher HTTPS access to Neptune on 8182
    - Implement `smokeTest`: write a uniquely identified node via an openCypher write, read it back via an openCypher read, assert the read value equals the written value, then best-effort delete the node
    - Return a descriptive connectivity error if Neptune is unreachable or IAM auth fails
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ]* 1.5 Write the Phase 0 Neptune integration smoke test
    - Add a test in `/tests` that invokes Graph_Lambda `smokeTest` against real Neptune and asserts read value equals written value and best-effort delete ran
    - _Requirements: 1.1, 1.3, 1.5_

  - [ ] 1.6 Milestone R1 — run the smoke test against REAL Neptune and report
    - Run `cdk deploy` for the Phase 0 stack and invoke `smokeTest` against the real cluster
    - Confirm the read value is identical to the written value; record connectivity evidence, rough Neptune cost, and any blockers before any F01–F10 business logic
    - _Requirements: 1.3, 1.6_

- [ ] 2. Phase 1 — Data model, OntologyRepository & seeding
  - [ ] 2.1 Implement the OntologyRepository with config-driven allowed types
    - In `/backend/graph`, implement `getObject`, `findObjects`, `getRelatedObjects`, `createObject`, `createRelationship`, and `upsertAssessment`
    - Make `ALLOWED_TYPES = {Patient, ClinicalTrial, Assessment}` and allowed edge types config-driven; reject any other type/edge
    - Keep domain eligibility rules OUT of the repository
    - _Requirements: 16.1, 16.2, 16.4, 16.5_

  - [ ] 2.2 Implement JSON-string property serialize/deserialize and the three node shapes
    - Model Patient (id, name, eleven clinical attributes, `synthetic=true`, `createdAt`), ClinicalTrial, and Assessment properties per the Glossary
    - Serialize complex data (`criteriaJson`, `criterionResultsJson`, `previousResultsJson`) as JSON-string properties on write and deserialize them on read
    - _Requirements: 16.1, 16.6_

  - [ ]* 2.3 Write unit test for JSON serialize/deserialize round-trip
    - Assert serialize → store shape → deserialize equals the original value for `criteriaJson`/`criterionResultsJson`/`previousResultsJson`
    - _Requirements: 16.6_

  - [ ] 2.4 Implement deterministic assessment id and create-or-update upsert
    - Implement `upsertAssessment` keyed on `patientId#trialId` using the design's openCypher MERGE (edges `HAS_ASSESSMENT`/`FOR_TRIAL`), accepting `previousResultsJson` computed in Python
    - Guarantee at most one current Assessment per patient/trial regardless of re-runs
    - _Requirements: 16.2, 16.3, 16.7, 5.5_

  - [ ] 2.5 Implement the dedicated direct seed path (create nodes, no matching)
    - In `/seed`, build a local script that invokes Graph_Lambda DIRECTLY using developer/local AWS IAM credentials (AWS SDK `lambda:InvokeFunction` on Graph_Lambda, or a direct Graph_Lambda `createObject` invocation) — it does NOT depend on API Gateway or API_Lambda (which do not exist until Phase 1b) and does NOT fan out to Matching_Worker
    - Load T001 + 42 patients from `files/patients.csv`, setting `synthetic=true` and ISO-8601 `createdAt`, idempotently (re-run creates no duplicates)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.7, 2.8_

  - [ ] 2.6 Implement the explicit "Evaluate seeded patients" batch operation
    - Add a distinct seed-script flag / action that fans out `reevaluate` per already-seeded patient via a direct Graph_Lambda/Matching_Worker invocation path (developer IAM creds, no API Gateway dependency in Phase 1), separate from automatic matching on `POST /patients`; it may be surfaced via the API later
    - _Requirements: 2.6_

  - [ ]* 2.7 Validate seed/neighborhood/assessment openCypher against real Neptune
    - Integration test: run the seed, assert exactly one T001 and exactly 42 patients, re-run and assert no duplicates; exercise the neighborhood query and an assessment upsert against real Neptune
    - _Requirements: 2.4, 2.7, 11.1, 16.3_

  - [ ] 2.8 Milestone A01 — T001 and exactly 42 patients exist, idempotent re-run
    - Run the seed against real Neptune; confirm one T001 and 42 patients and that re-running creates no duplicates
    - _Requirements: 2.1, 2.4, 2.7_

- [ ] 3. Phase 1b — Minimal Recruitment Hub UI on real data
  - [ ] 3.1 Deploy the HTTP API + API_Lambda serving GET /health and GET /patients
    - Add the API Gateway HTTP API and the Python API_Lambda in `/backend/api` (outside the VPC)
    - Implement `GET /health` (no auth) and `GET /patients` invoking Graph_Lambda synchronously; grant API_Lambda `lambda:InvokeFunction` on Graph_Lambda
    - Note: this backend is reused/extended in later phases, not rebuilt
    - _Requirements: 17.1, 17.2, 19.2, 19.4_

  - [ ] 3.2 Add Cognito user pool, Hosted UI, demo user and HTTP API JWT authorizer
    - CDK: Cognito user pool + Hosted UI + one Demo_User; attach a JWT authorizer to the HTTP API covering all routes except `GET /health`
    - _Requirements: 20.4, 20.5_

  - [ ] 3.3 Add S3 + CloudFront SPA hosting
    - CDK: private S3 site bucket with OAC CloudFront origin, `autoDeleteObjects`, `RemovalPolicy.DESTROY`
    - _Requirements: 19.1, 21.3_

  - [ ] 3.4 Build the minimal React + Cloudscape app shell with five nav items
    - In `/frontend`, scaffold React + TypeScript + Cloudscape (light theme) with nav items: Recruitment Hub, Clinical Trials, Ontology Explorer, AI Agent, Evaluation
    - Add the persistent "Synthetic Demo Data – Not for Clinical Use" label and loading states; attach the Cognito JWT as a bearer token on API calls (no AWS keys in browser)
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 20.1, 20.5_

  - [ ] 3.5 Implement the Recruitment Hub patient table reading real seeded Neptune data
    - Render the patient table from `GET /patients` (real seeded Neptune data) behind Cognito login
    - _Requirements: 4.1_

  - [ ] 3.6 Milestone — Recruitment Hub shows the 42 real seeded patients in the browser
    - Verify the patient table displays the 42 seeded patients from Neptune behind Cognito login with the synthetic-data label visible
    - _Requirements: 3.1, 3.4, 4.1_

- [ ] 4. Phase 2 — Eligibility worker & automatic matching
  - [ ] 4.1 Create the Matching_Worker Lambda (outside VPC) and single-call Bedrock evaluation
    - In `/backend/worker`, create the Python Matching_Worker outside the VPC with `bedrock:InvokeModel` and `lambda:InvokeFunction` on Graph_Lambda (CDK)
    - Make exactly ONE Bedrock Converse call per Patient-Trial assessment that evaluates ALL applicable criteria from the approved `criteriaJson` at once and returns structured per-criterion results — an array of `{criterionId, result ∈ MET/NOT_MET/UNKNOWN/NOT_APPLICABLE, brief evidence}` — validated against a strict JSON schema before use
    - Do NOT make one model call per individual criterion
    - _Requirements: 7.1, 18.1, 18.3, 19.4_

  - [ ] 4.2 Implement deterministic numeric parsers and recheck
    - Parse age, HbA1c, eGFR, and metformin duration from `age`/`lab_results`/`treatment_history` only when unambiguous; return `UNKNOWN` otherwise
    - Apply T001 numeric limits: age 30–70 incl, HbA1c 7.0–10.0 incl, eGFR ≥ 60, metformin stable ≥ 3 months; deterministic recheck overrides the LLM on numeric limits
    - _Requirements: 7.2, 7.3, 7.9, 7.10, 7.11_

  - [ ]* 4.3 Write unit tests for numeric parsers
    - Test representative `lab_results`/`treatment_history` strings and assert `UNKNOWN` on missing/ambiguous values
    - _Requirements: 7.2, 7.3, 7.11_

  - [ ] 4.4 Implement decision aggregation and Failed handling
    - Aggregate: any applicable `NOT_MET` → `Not suitable`; else any applicable `UNKNOWN` → `Manual review required`; else `Suitable`; treat childbearing criteria as `NOT_APPLICABLE` for male patients
    - On technical/parse/schema error, set `processingStatus=Failed` with NO decision; store `reasoning` on completed assessments
    - _Requirements: 6.1, 6.2, 7.4, 7.5, 7.6, 7.7, 7.8, 7.14, 8.1, 8.2, 8.3_

  - [ ]* 4.5 Write unit tests for the decision aggregator including boundary cases
    - Assert HbA1c 6.9% → `Not suitable`; missing eGFR with no other violation → `Manual review required`; Failed carries no decision
    - _Requirements: 6.1, 7.7, 7.8, 7.14, 8.2_

  - [ ] 4.6 Persist assessments and store evaluation metadata via create-or-update
    - Worker reads active trials + patient via Graph_Lambda, upserts a `Running` assessment, then the `Completed`/`Failed` assessment keyed `patientId#trialId`, computing compact `previousResultsJson` in Python
    - Store `decision`, `reasoning`, `criterionResultsJson`, model id, `evaluatedAt`, and keep `protocolVersion` identifiable; present decisions as preliminary screening only
    - _Requirements: 5.3, 5.5, 7.12, 7.13, 16.7_

  - [ ] 4.7 Wire POST /patients to create patient + async-invoke the worker
    - Add `POST /patients` and `GET /patients/{id}` in API_Lambda: create the Patient node via Graph_Lambda, async-invoke Matching_Worker with the patient id, return 202; do NOT invoke the worker for seeded patients
    - _Requirements: 5.1, 5.2, 5.6, 17.2_

  - [ ] 4.8 Seed a SECOND synthetic Active trial directly into Neptune for A03
    - Extend the seed path to create a second simple synthetic `Active` trial with minimal `criteriaJson` via direct `createObject` (not via PDF upload)
    - _Requirements: 2.1, 13.6_

  - [ ]* 4.9 Write Neptune integration test for create-or-update determinism
    - Re-run matching for the same `patientId#trialId` and assert exactly one current Assessment remains
    - _Requirements: 5.5, 16.3_

  - [ ] 4.10 Milestone A03/A04/A05 — two active trials, three decisions, boundaries
    - Verify a new patient with two active trials yields exactly two assessments (A03), all three decisions are demonstrable (A04), and HbA1c 6.9% → Not suitable and missing eGFR (verified P086 fixture) → Manual review required (A05)
    - _Requirements: 5.6, 6.3, 7.7, 7.8_

  - [ ] 4.11 Create/verify the P086 synthetic evaluation fixture (42-seed unchanged)
    - Create or seed P086 (and the other five evaluation patients if not already present with the exact required values) as a SEPARATE synthetic evaluation fixture used only by the evaluation suite, OR verify an existing equivalent patient already has the exact required input values (for P086, missing eGFR → Manual review required)
    - Keep the original 42-patient seed UNCHANGED — do not mutate the 42-seed to fit labels
    - _Requirements: 10.1, 10.4, 2.8_

- [ ] 5. Phase 3 — API, Strands agent & approval
  - [ ] 5.1 Complete the full HTTP API surface
    - Add remaining routes in API_Lambda: `GET /trials`, `GET /trials/{id}`, `POST /trials/upload`, `POST /trials/extract`, `POST /trials/{id}/activate`, `GET /assessments`, `POST /assessments/{id}/review`, `GET /graph/neighborhood`, `POST /evaluation/run` (handlers may be stubbed where their phase is later), plus the agent routes added below
    - Deserialize JSON-string properties in responses
    - _Requirements: 17.3, 17.4, 17.5, 17.6, 17.7, 17.9, 16.6_

  - [ ] 5.2 Implement the one Strands Agent with three read-only tools
    - In API_Lambda, run exactly one Strands Agent with `get_object`, `find_objects`, `get_related_objects`, each calling Graph_Lambda; ground answers in retrieved objects; no `CreatePatient` tool, no raw-Cypher tool
    - _Requirements: 14.1, 14.2, 14.3, 14.6, 14.7_

  - [ ] 5.3 Implement the :_Proposal store operations in Graph_Lambda
    - Add `saveProposal`/`loadProposal`/`consumeProposal`/`discardProposal` persisting a short-lived `:_Proposal` record (`proposalId`, action, args, `createdAt`, `consumed=false`, short TTL)
    - Exclude `:_Proposal` from `ALLOWED_TYPES`, the ontology, the Explorer, and neighborhood results (distinct technical label)
    - _Requirements: 15.3, 16.4, 16.8_

  - [ ] 5.4 Implement the two approval-gated write actions and /agent/chat
    - Add `reevaluate_eligibility(patient_id, trial_id)` and `update_assessment(assessment_id, decision, reason)` that produce an unexecuted proposal (proposalId + concrete args) instead of mutating; `POST /agent/chat` returns reply + tool activity + optional proposal preview
    - _Requirements: 14.4, 14.5, 15.1, 15.2, 15.4, 17.8_

  - [ ] 5.5 Implement /agent/confirm with revalidation, consume, idempotent apply
    - On confirm: load proposal, revalidate action ∈ allowed set, args valid, caller = Demo_User, not expired/consumed/discarded; set `consumed=true`, then apply the idempotent action (reevaluate via Worker; update_assessment via targeted MERGE); on validation failure → no mutation + error; on apply failure after consume → error + re-issue a NEW proposal (no silent retry)
    - _Requirements: 15.6, 15.7, 15.8, 15.10, 20.6_

  - [ ] 5.6 Implement /agent/reject and not-actionable handling
    - `POST /agent/reject` discards the proposal with no mutation; approval of consumed/discarded/expired → no mutation + "no longer actionable" message
    - _Requirements: 15.5, 15.9_

  - [ ]* 5.7 Write unit test for the proposal lifecycle guard
    - Assert approve-once applies at most once; rejected/expired/consumed proposals cause no mutation
    - _Requirements: 15.5, 15.8, 15.9_

  - [ ] 5.8 Milestone A08/A09 — grounded chat and approval-gated writes
    - Verify grounded answers via read tools (A08) and that no change occurs before approval, reject = no change, approve = applied at most once (A09)
    - _Requirements: 14.3, 15.4, 15.5, 15.8_

- [ ] 6. Phase 4 — Remaining frontend views + PDF extraction & trial lifecycle
  - [ ] 6.1 Implement PDF upload and native Bedrock extraction backend
    - `POST /trials/upload` returns an S3 presigned PUT and records `protocolS3Key` (private bucket); `POST /trials/extract` reads the PDF bytes and calls Bedrock Converse with a `document` block (`format: pdf`, source bytes, `citations: {enabled: true}`) to extract structured inclusion/exclusion criteria, handling image-based T001.pdf natively (no Textract/OCR)
    - Return a clear error only for encrypted/oversized/no-usable-criteria PDFs
    - Acceptance: the extraction path must produce usable inclusion AND exclusion criteria from the real image-based `files/T001.pdf` (verified by the mandatory integration check in Task 6.9)
    - _Requirements: 12.1, 12.2, 12.4, 18.1, 18.4_

  - [ ] 6.2 Implement trial activation lifecycle backend
    - `POST /trials/{id}/activate` changes Draft → Active with no retroactive matching; new trials from PDF start as Draft; keep the protocol PDF accessible; support Draft/Active/Inactive and multiple trials/assessments per patient
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5, 13.6_

  - [ ] 6.3 Build the Clinical Trials view (upload → extract → review → activate)
    - Trial cards/table, `Upload Protocol`, PDF preview next to extracted criteria, a review form to enter missing metadata, and `Activate Trial`
    - _Requirements: 12.3, 12.5, 13.1, 13.2_

  - [ ] 6.4 Complete the Recruitment Hub (Add Patient, Review overlay, override)
    - Add the `Add Patient` form; the Review Patient overlay showing patient data, trial criteria, AI Decision, and explanation, displaying all trial assessments for the selected patient even when the table is filtered; trial dropdown + search + status filter
    - Implement `Modify Decision` human override requiring a mandatory non-empty explanation, storing `reviewedDecision`+reason while retaining the original AI decision, shown alongside it; no second confirmation dialog
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 9.1, 9.2, 9.3, 9.4, 9.5_

  - [ ] 6.5 Build the Ontology Explorer live graph view
    - Render Patient/Assessment/ClinicalTrial nodes and edges from `GET /graph/neighborhood` (live Neptune), click node → side panel details, one basic filter; reflect new patient/assessments on subsequent load
    - _Requirements: 11.1, 11.2, 11.3, 11.4_

  - [ ] 6.6 Build the AI Agent chat view with proposal Approve/Reject card
    - Chat UI with predefined example prompts, visible tool activity, and an Approve/Reject card for write proposals calling `/agent/confirm` and `/agent/reject`
    - _Requirements: 14.4, 14.5, 15.2, 15.10_

  - [ ] 6.7 Add new-patient polling with 120s cutoff in the frontend
    - Automatically display eligibility results after creating a patient — this is a core demo requirement, so use simple polling (no WebSockets) to refresh assessment status without manual refresh; stop at 120s and show a non-blocking status
    - _Requirements: 5.4_

  - [ ] 6.9 Mandatory PDF integration check with the real T001.pdf
    - Upload/point to the actual `files/T001.pdf` (image-based/scanned) and call the real Bedrock Converse extraction path (`document` block, `citations: {enabled: true}`) via the Task 6.1 backend; this is a verify-before-claim check
    - CONFIRM that usable inclusion AND exclusion criteria are extracted from the image-based PDF with no Textract and no OCR pipeline; confirm a clear error path is returned only for encrypted/oversized/no-usable-criteria PDFs
    - _Requirements: 12.1, 12.2, 12.4, 18.1_

  - [ ] 6.8 Milestone A02/A06/A07 — PDF→criteria, override, live graph
    - Verify PDF → extracted criteria → Active only after approval (A02), human override with mandatory explanation while AI result stays visible (A06), and the live graph reflects a new patient/assessment (A07)
    - _Requirements: 12.2, 13.2, 9.2, 11.3_

- [ ] 7. Phase 5 — Evaluation suite & polish
  - [ ] 7.1 Build the labeled evaluation set and POST /evaluation/run
    - Define ≥ 6 manually-reviewed labeled cases from real `patients.csv` rows spanning all three classes (P018/P011/P014/P033 → Not suitable, P001 → Suitable with male NOT_APPLICABLE assumption, P086 → Manual review required); label the set as not a validated gold set
    - All six evaluation cases MUST have VERIFIED input values and expected outcomes checked against `files/patients.csv` and the T001 protocol; P086 is created/seeded as a SEPARATE synthetic fixture (or a verified equivalent is used), and the original 42-patient seed is left UNCHANGED
    - Implement `POST /evaluation/run` returning Expected/Actual/Pass per case + Pass Rate; do NOT treat the 42 seeded patients as a gold set; no fabricated accuracy numbers
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 17.9_

  - [ ] 7.2 Build the Evaluation view
    - Render `Run Evaluation`, a Patient | Trial | Expected | Actual | Pass/Fail table, and a Pass Rate metric
    - _Requirements: 10.2, 10.3_

  - [ ] 7.3 Write the README (deploy, demo, cleanup, cost)
    - Document `cdk deploy`/`cdk destroy`, the demo walkthrough, a cleanup checklist for bootstrap/retained resources, and Neptune continuous idle cost (min 1.0 NCU, never scales to zero); do NOT claim all charges cease automatically
    - _Requirements: 21.4, 21.5, 21.6_

  - [ ] 7.4 Milestone A10/A11 — verify evaluation, deploy and teardown
    - Run the evaluation and confirm the table + Pass Rate (A10); verify `cdk deploy` and `cdk destroy` end to end with demo removal policies (A11)
    - _Requirements: 10.2, 10.3, 21.2, 21.3_

## Notes

- Tasks marked with `*` are optional test/verification sub-tasks and can be skipped for a faster MVP; core implementation tasks are never optional.
- Phase 0 is first and gating — no F01–F10 logic depends on Neptune until the smoke test passes (R1.6).
- The Phase 1b backend (HTTP API, API_Lambda, Cognito, S3+CloudFront, Recruitment Hub shell) is reused and extended by Phases 2–5, not rebuilt.
- All infrastructure is one AWS CDK TypeScript project under `/infra` with demo removal policies (`RemovalPolicy.DESTROY`, `autoDeleteObjects`, Neptune skip final snapshot).
- Each sub-task references the specific requirement clauses it satisfies for traceability.
- Testing is focused unit tests + essential Neptune integration tests + A01–A11 acceptance checks; no property-based testing and no fabricated accuracy numbers.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["1.3"] },
    { "id": 3, "tasks": ["1.4"] },
    { "id": 4, "tasks": ["1.5"] },
    { "id": 5, "tasks": ["1.6"] },
    { "id": 6, "tasks": ["2.1", "2.2"] },
    { "id": 7, "tasks": ["2.3", "2.4"] },
    { "id": 8, "tasks": ["2.5"] },
    { "id": 9, "tasks": ["2.6", "2.7"] },
    { "id": 10, "tasks": ["2.8"] },
    { "id": 11, "tasks": ["3.1", "3.2", "3.3"] },
    { "id": 12, "tasks": ["3.4"] },
    { "id": 13, "tasks": ["3.5"] },
    { "id": 14, "tasks": ["3.6"] },
    { "id": 15, "tasks": ["4.1", "4.2"] },
    { "id": 16, "tasks": ["4.3", "4.4", "4.8"] },
    { "id": 17, "tasks": ["4.5", "4.6"] },
    { "id": 18, "tasks": ["4.7", "4.9", "4.11"] },
    { "id": 19, "tasks": ["4.10"] },
    { "id": 20, "tasks": ["5.1", "5.3"] },
    { "id": 21, "tasks": ["5.2", "5.4"] },
    { "id": 22, "tasks": ["5.5", "5.6"] },
    { "id": 23, "tasks": ["5.7"] },
    { "id": 24, "tasks": ["5.8"] },
    { "id": 25, "tasks": ["6.1", "6.2", "6.5"] },
    { "id": 26, "tasks": ["6.3", "6.4", "6.6", "6.7", "6.9"] },
    { "id": 27, "tasks": ["6.8"] },
    { "id": 28, "tasks": ["7.1"] },
    { "id": 29, "tasks": ["7.2", "7.3"] },
    { "id": 30, "tasks": ["7.4"] }
  ]
}
```
