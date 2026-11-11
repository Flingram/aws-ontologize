# AWS Ontology & Agent Platform — Project Specification v1.1

**Date:** November 9, 2026  
**Status:** Product and technical requirements baseline for Kiro spec-driven development  
**First use case:** Glycurex T001 — Clinical Trial Recruitment (synthetic data only)  
**Objective:** A visually impressive, working AWS showcase — **not an enterprise product**.  
**UI language:** English; independent AWS/Cloudscape design, with user journeys inspired by the original Palantir Workshop.  
**Relationship to v1.0:** This v1.1 supersedes the technical implementation plan in v1.0. All ten agreed features and functional decisions remain in scope.

---

**Document audience:** Kiro. This file defines what to build, required technical constraints, and how success is verified. The developer workflow, phase sequence, and prompts live in `DEVELOPMENT_GUIDE.md`.

---

## 0. Product Principle: SHOWCASE FIRST

> Build a sexy, polished and demonstrably working AWS Ontology & Strands Agents SHOWCASE — not a production-grade enterprise platform. Preserve the agreed ten features, but implement each with the minimum code and infrastructure required. Favor a smooth visual demo over unnecessary abstractions, enterprise security, complex governance, multi-agent orchestration, extensive audit systems, and gold-plating. **Amazon Neptune must be the real source of graph objects and relationships.** No fake graph. Use synthetic patients only. Keep basic authentication, private Neptune access, server-side validation and explicit approval for all chat-agent write operations. Build incrementally and prove each milestone before moving on.

**The five wow moments:**
1. **PDF → Ontology:** Upload protocol → Bedrock extracts criteria → quick user review → activate the trial.
2. **Patient → AI Matching:** Add a patient through a form → automatically evaluate them against **all active trials**.
3. **Live Neptune Graph:** Show Patient → Assessments → Trials as a clickable graph backed by real Neptune data.
4. **Chat with the Ontology:** A Strands Agent answers questions grounded in the graph.
5. **Agentic Action:** The agent proposes a change or re-evaluation → user approves → visible update.

**Do not build:** A Palantir clone; Foundry Workshop/Pipeline Builder designers; a multi-agent swarm; GraphRAG; AgentCore; a custom MCP server; multi-tenancy; fine-grained RBAC; a no-code ontology designer; elaborate history tracking; full production compliance; clinical enrollment; or automatic retrospective matching when new trials are activated.

---

## 1. Scope — All Ten Features Remain Included

| ID | Feature | Smallest Visible Working Version |
|---|---|---|
| F01 | Glycurex T001 | Original protocol PDF, approved criteria, trial details view |
| F02 | Patient management | Import original CSV; search, filter, view details, and `Add Patient` |
| F03 | Automatic eligibility matching | **New** patient → every **active** trial → one assessment per trial |
| F04 | Three possible decisions | `Suitable`, `Not suitable`, `Manual review required` |
| F05 | Human review | Reviewer may correct a decision **with a required explanation** |
| F06 | Evaluation suite | Small, independently labeled test set; run and display pass/fail |
| F07 | Ontology Explorer | Interactive graph queried **live** from Neptune; click nodes for details |
| F08 | Interactive Strands Agent | Chat, three read-only tools, two approval-gated write actions |
| F09 | PDF criteria extraction | Upload → Bedrock → structured criteria → review/approve |
| F10 | Multiple trials | Draft/Active/Inactive; multiple trials and assessments per patient |

**Explicit scope boundary:** All ten features must work, but **without** advanced configuration UIs, complex roles, or general-purpose workflow designers.

---

## 2. Demo Journey (Mandatory Product Requirement)

1. Open **Recruitment Hub**. T001 and the 42 original synthetic patients are already present. A readable table and status chips immediately establish the use case.
2. Navigate to **Clinical Trials**. Upload a **second synthetic study protocol PDF**. AI displays extracted criteria. The user reviews the information, optionally adds a few basic trial details, and clicks `Activate Trial`.
3. Return to **Recruitment Hub**. Enter a new synthetic patient via `Add Patient`. **The agent must not create patients.**
4. The UI shows `Evaluating...`, then **one result per active trial**, including a one-sentence explanation. Update the status without requiring a manual refresh (simple polling is sufficient).
5. In **Ontology Explorer**, click the patient and explore edges to assessments and trials; show the corresponding details in a side panel.
6. In **AI Agent**, ask: `Which trials is this patient eligible for, and why?` The agent reads Neptune through approved tools and provides a concrete answer.
7. Ask the agent: `Re-evaluate patient Pxxx for trial T001.` Show an action preview with `Approve / Reject`. **No mutation before approval**; after approval, run the evaluation and update the UI.
8. Optionally open **Evaluation** to show expected vs. actual labels and the pass rate.

**Demo goal:** One cohesive journey in **one web application**. No need to switch to the AWS Console during the presentation.

---

## 3. UI — Beautiful, Simple, Palantir-Inspired

**Technology:** React + TypeScript + AWS Cloudscape. Professional light theme, clear typography, restrained colors, responsive components, status chips, and meaningful loading states. Do not invent a complicated design system.

**Five navigation items (one single-page application):**

- **Recruitment Hub** (home): Trial dropdown, search, status filter, patient table, and `Add Patient`. Clicking a patient opens a **Review Patient** detail overlay containing patient data, trial criteria, AI decision, explanation, and `Modify Decision`. The patient detail view shows **all** trial assessments, even if the main table is filtered to a selected trial.
- **Clinical Trials:** Trial cards or table, `Upload Protocol`, PDF preview next to extracted criteria, and `Activate Trial`. Statuses: `Draft / Active / Inactive`.
- **Ontology Explorer:** Graph (e.g., Cytoscape.js or React Flow) with `Patient`, `Assessment`, and `ClinicalTrial` nodes and real Neptune edges. Click a node → detail panel; one basic filter is sufficient. **No** graph editor.
- **AI Agent:** Chat, several predefined example prompts, visible tool activity, and an `Approve / Reject` card for write proposals.
- **Evaluation:** `Run Evaluation`, compact `Patient | Trial | Expected | Actual | Pass/Fail` table, and `Pass Rate` metric.

**Design rule:** The original Palantir Workshop is a reference for table/filter/review interactions. The uploaded Workshop variable lineage was modified later (it includes moVe-it and 55 patients); it is **not** a source of T001 clinical facts.

---

## 4. Simple Domain Model: Real Neptune Ontology

**Database:** **Amazon Neptune Database Serverless**, Property Graph with **openCypher**. Neptune Database is the real, sole persistent operational store for business objects, graph relationships, and assessments. S3 stores protocol PDFs only. **No DynamoDB, Aurora, RDS, or OpenSearch.**

**Node types (only three required):**

| Node | Minimum Properties |
|---|---|
| `Patient` | `id`, `name`, the eleven original patient attributes, `synthetic=true`, `createdAt` |
| `ClinicalTrial` | `id`, `name`, `status` (Draft/Active/Inactive), `criteriaSummary`, `criteriaJson`, `protocolS3Key`, `protocolVersion`, a few optional trial details |
| `Assessment` | `id`, `patientId`, `trialId`, `processingStatus`, `decision`, `reasoning`, `criterionResultsJson`, `reviewReason`, `reviewedDecision`, `updatedAt` |

**Edges:**

```cypher
(:Patient)-[:HAS_ASSESSMENT]->(:Assessment)-[:FOR_TRIAL]->(:ClinicalTrial)
```

**Reusable abstraction:** A small `OntologyRepository` with `getObject(type,id)`, `findObjects(type,filters)`, `getRelatedObjects(id)`, `createObject(...)`, and `createRelationship(...)`. Allowed types and operations are defined in compact configuration. **Do not build** a metamodel engine or visual ontology editor. Domain-specific rules (such as Glycurex eligibility) stay **outside** `OntologyRepository`. The API should be reusable without turning this showcase into a platform-building project.

**Storage:** Persist complex criteria/assessment JSON as JSON-string properties; deserialize when returning API responses. PDFs are stored in S3. No second persistence layer.

**Original CSV inputs:** `patients.csv` contains **42** patients and 15 columns; `patient_trial_link.csv` contains **42** links to T001. Import patients while preserving their original clinical values. The link CSV documents the original model; in the new model, relationships are represented through assessment nodes. No permanent parallel patient-to-trial link table is needed.

**Minimal assessment history:** The current assessment for each patient/trial shows the AI decision, an optional manually overridden decision, and the accompanying explanation. Re-evaluation retains at least the immediately previous result in a compact history (e.g., `previousResultsJson` or a short history stored on the assessment). Do not implement event sourcing or separate `AuditEvent` / `PendingAction` nodes.

---

## 5. Glycurex T001 — Domain Eligibility Rules

**Source:** Original `T001.pdf`, version 1.0, February 21, 2025 — **not** the later moVe-it criteria.

**Inclusion:** Age **30–70**, inclusive; confirmed Type 2 Diabetes Mellitus; **HbA1c 7.0–10.0%**, inclusive; stable metformin dose for **at least three months**; health status good/stable/normal; **eGFR ≥ 60 mL/min/1.73 m²**; ability and willingness to give informed consent and comply with study procedures; for women of childbearing potential, effective contraception **and** a negative pregnancy test.

**Exclusion:** Type 1 or secondary diabetes; insulin or antidiabetic medication other than metformin within the last three months; eGFR below 60; pregnancy; inability to consent or adhere to the protocol; age outside the permitted range.

**One simple decision algorithm:**

1. An LLM via Bedrock reads the **approved trial criteria** and the eleven patient attributes. It returns concise evidence for each criterion: `MET`, `NOT_MET`, `UNKNOWN`, or `NOT_APPLICABLE`, with a brief source/reference or explanation.
2. Check numeric limits (age, HbA1c, eGFR, metformin duration, when unambiguously extractable) **in Python code**. Missing or ambiguous values remain `UNKNOWN`.
3. At least **one definitively violated** criterion → `Not suitable`. Otherwise, if required information is missing → `Manual review required`. Only if **all applicable** criteria are demonstrably satisfied → `Suitable`.
4. If applicability is uncertain (such as contraception or pregnancy requirements), do not assume the criterion is satisfied. `NOT_APPLICABLE` is allowed only when supported by clear domain evidence.
5. A technical model/parsing error → `processingStatus=Failed`; **do not fabricate** a clinical eligibility decision.

**Output:** `decision`, `reasoning` (one clear sentence), `criterionResults`, `modelId`, and `evaluatedAt`. The approved trial protocol version must remain identifiable from the assessment. The original AIP Logic prompt is a **domain reference**, but normalize its inconsistent `enrollment_...` / `eligibility_...` field names to `decision` and `reasoning`.

**Critical:** The AI produces **preliminary screening assessments only**. No clinical clearance and no automatic patient enrollment.

---

## 6. PDF Upload and Trial Creation — No Pipeline Builder

**User journey:**

```text
Upload PDF → Extract criteria with Bedrock → Draft preview
           → User edits/completes details → Activate Trial
           → Only then include this trial when matching NEW patients
```

**Extraction output (minimal illustrative JSON):**

```json
{
  "trialId": "T001",
  "name": "Glycurex in Type 2 Diabetes Management",
  "summary": "Eligibility summary for human review",
  "criteria": [
    { "id": "age", "type": "inclusion", "field": "age", "operator": "between", "min": 30, "max": 70, "unit": "years", "sourcePage": 1 },
    { "id": "egfr", "type": "inclusion", "field": "egfr", "operator": "gte", "value": 60, "unit": "mL/min/1.73m2", "sourcePage": 2 }
  ]
}
```

**This is NOT the complete T001 criteria set.** The real extraction must cover **all** inclusion and exclusion criteria. Optional `sourceText` snippets may help users verify the extracted criteria.

**Implementation:** Upload the PDF to S3 (presigned URL or simple backend upload). One Python Lambda uses a compatible Bedrock model to extract structured criteria. If direct PDF input is unsupported by the approved model, first extract text from text-based PDFs locally. For scanned PDFs, return a clear MVP error instead of building a Textract pipeline. No flatten/join step and no Glue pipeline. Allow users to enter a few missing trial metadata fields directly in the review form.

**Rules:** A trial starts as `Draft`. Only an explicit click on `Activate` changes it to `Active`. **Do not retroactively match** existing patients upon activation. Only user-reviewed criteria may be used for subsequent evaluations. The original protocol PDF remains accessible.

---

## 7. Automatic Matching and Human Review

**Trigger:** Every patient newly added through Recruitment Hub automatically starts evaluation against **all currently active trials**. Importing the original 42 patients **must not trigger an unexpected bulk evaluation**. Provide a separate explicit `Evaluate seeded patients` operation if needed.

**Minimal workflow:**

```text
POST /patients
  → Create Patient node in Neptune
  → Invoke Matching Worker asynchronously (AWS Lambda invoke; EventBridge optional)
  → Read active trials
  → Create Pending/Running assessment for each trial
  → Call Bedrock eligibility evaluation
  → Save results in Neptune
  → UI polls until Completed or Failed
```

**Keep it simple:** An **asynchronous Lambda invocation** with payload `{patientId}` is sufficient for the MVP. EventBridge is **not required**; use it only if it clearly simplifies implementation. Duplicate invocations must not create multiple current assessments for the same patient/trial pair. A deterministic assessment ID (such as `patientId#trialId`) and simple create-or-update behavior are sufficient for this showcase. No Step Functions, SQS/DLQ infrastructure, or generic scheduling without a demonstrated need.

**Human review:** The reviewer sees the result and may use `Modify Decision` with a **mandatory explanation**. Display the human override clearly alongside the AI's preliminary decision. Never silently overwrite the underlying AI finding. No multi-stage approval workflow.

---

## 8. One Strands Agent, Real Neptune Tools, Simple Approval

Run **exactly one** Strands Agent in the Python backend. Make the Bedrock model configurable. No AgentCore and no separate MCP server.

**Read-only tools:**

```python
get_object(object_type, object_id)
find_objects(object_type, filters)
get_related_objects(object_id)
```

**Agent-initiated write actions (approval required):**

```python
reevaluate_eligibility(patient_id, trial_id)
update_assessment(assessment_id, decision, reason)
```

**Do not provide** a `CreatePatient` agent tool. Patients are created through the Recruitment Hub form only.

**Minimum approval flow:**

1. The agent identifies a required write action and its concrete arguments.
2. The backend returns an **unexecuted** action proposal; the UI shows `Approve` and `Reject`.
3. **Reject** → no changes. **Approve** → the backend validates and executes the approved action **once**.
4. Update the affected assessment/graph in the UI and show a clear success or error message.

**Technical choice:** Prefer the supported **Strands `HumanInTheLoop` interrupt/resume flow** when practical. A web application must safely retain the paused agent state on the server until approval. For this MVP, use minimal short-lived server-side state or a temporary signed action reference; do not introduce a large `PendingAction` domain and lifecycle. A signed token **alone** does not guarantee **single execution**. If the SDK approach is unnecessarily complex, use a minimal server-side proposal ID: validate the stored parameters, mark the proposal consumed after execution, and set a short expiry. **Choose one approach, not both.**

**Non-negotiable:** The model or its tools must never execute a graph mutation directly before approval. The backend revalidates allowed action types, parameters, and the demo user. Do not expose arbitrary Cypher execution as an agent tool. Human review initiated directly from the UI is already an explicit user action and does **not** need a second confirmation dialog.

---

## 9. Compact AWS Architecture

```text
React + Cloudscape (static hosting)
          |
    API Gateway HTTP API + Demo Login
          |
     API Lambda (Python, Strands + Bedrock)
          |               \
          |                --> S3 (PDFs)
          |
          +-- AWS Lambda Invoke --> Graph Lambda (in VPC) --> Neptune Serverless (openCypher)
          |
          +-- async Lambda Invoke --> Matching Worker Lambda (Python + Bedrock)
                                          |
                                          +--> Graph Lambda --> Neptune
```

| Component | Minimum Choice |
|---|---|
| Frontend | React + TypeScript + Cloudscape, static hosting: S3/CloudFront **or** Amplify Hosting (choose one) |
| API | API Gateway HTTP API + **one** Python API Lambda; include Strands Agent here |
| Graph | **One** private Neptune Database Serverless instance/cluster, accessed using openCypher |
| Graph connection | **One** Python Graph Lambda in the Neptune VPC; all access through typed repository methods |
| Matching | **One** Python worker Lambda, invoked asynchronously instead of using a workflow orchestrator |
| AI | Amazon Bedrock; configure **one** affordable, capable, region-available model; optionally use it for both extraction and matching |
| Files | **One** private S3 bucket for protocol PDFs |
| Login | One demo user via Cognito/Hosted Login or comparably simple managed authentication; no custom password system |
| IaC | One AWS CDK TypeScript project; Python backend and React frontend in a monorepo |
| Logs | Basic CloudWatch logs and understandable error messages |

**Key network simplification:** Place **only the Graph Lambda** inside the VPC. The API and worker Lambdas run outside the VPC. The Graph Lambda needs no calls to external APIs and communicates privately only with Neptune. Other Lambdas invoke it through AWS Lambda Invoke using IAM. **No NAT Gateway.** First prove connectivity using the correct authentication, TLS, and Neptune openCypher client; then add business logic.

**Verify current AWS details before implementation:** Neptune configuration in the target region, IAM/openCypher client behavior, Bedrock model access, and PDF/structured-output support. Use `eu-central-1` as the proposed default region **only if** all required resources are available there.

**Cost warning:** Neptune Serverless has ongoing minimum capacity costs even while idle. Kiro must document a rough cost expectation, implement `cdk destroy`/cleanup, and make it possible to tear down the demo after use. **Do not** claim that Neptune is charged only while actively queried.

**Do not add:** AgentCore, Glue, Textract, DynamoDB, Aurora, OpenSearch, Step Functions, Fargate, EKS, WAF, complicated networking, NAT, service mesh, Organizations/SCP setup, or a sophisticated monitoring stack.

---

## 10. Minimum Endpoints and Repository Structure

**HTTP API (group routes together; do not split into individual microservices):**

```text
GET  /health
GET  /patients                 POST /patients          GET /patients/{id}
GET  /trials                   GET  /trials/{id}
POST /trials/upload            POST /trials/extract    POST /trials/{id}/activate
GET  /assessments?patientId=...&trialId=...
POST /assessments/{id}/review
GET  /graph/neighborhood?objectId=...
POST /agent/chat               POST /agent/confirm    POST /agent/reject
POST /evaluation/run
```

**Small repository layout:**

```text
/infra/             AWS CDK TypeScript
/backend/api/       HTTP API, Strands, PDF extraction
/backend/graph/     Neptune repository, openCypher
/backend/worker/    Automatic eligibility evaluation
/frontend/          React + Cloudscape
/seed/              CSV import, T001 seed, test patients
/tests/             A few critical end-to-end tests
/README.md          Deploy, demo, cleanup
```

**Core simplicity rule:** Do not create more layers, packages, or frameworks than this product needs. Python and TypeScript may call straightforward functions directly. No overarching CQRS, DDD, or event-sourcing architecture.

---

## 11. Minimum Necessary Security and Quality Rules

Even a demo needs these **small, non-negotiable** safeguards:

- **Synthetic patients only**, clearly labeled `Synthetic Demo Data – Not for Clinical Use`.
- Keep **Neptune private** and never accessible directly from the browser or public internet. Use TLS and AWS IAM as required by the service and selected access mechanism.
- Use **one demo login** for the web application; no AWS keys in the browser and no unauthenticated public write API.
- Agent-initiated writes **always** require explicit approval. Validate allowed types/IDs/parameters server-side and prevent trivial duplicate execution.
- Treat PDFs and prompt content as untrusted input. Never execute external commands or model-generated arbitrary Cypher.
- Treat clearly missing clinical attributes as `UNKNOWN`; validate LLM output against a structured schema.
- A few clear error cases are sufficient: upload failed, AI evaluation failed, graph unavailable. No enterprise alerting platform.

**Not required:** RBAC matrix, tenant isolation, full compliance audits, separate KMS key-management architecture, WAF, elaborate secrets-vault abstractions, full business-event history, or complex organizational approval processes.

---

## 12. Acceptance Tests — Only What Protects the Wow Effect

| ID | Observable Test | Pass Condition |
|---|---|---|
| A01 | Seed | T001 and exactly 42 original CSV patients exist in the graph; rerunning import does not create duplicates |
| A02 | PDF | New synthetic study PDF → criteria preview → `Active` only after user approval |
| A03 | Matching | Add one new patient with two active trials → exactly **two** assessments with statuses and explanations |
| A04 | Three decisions | Demonstrate correct `Suitable`, `Not suitable`, and `Manual review required` examples |
| A05 | Rule boundaries | HbA1c 6.9% for T001 = `Not suitable`; missing eGFR, absent other definite violations = `Manual review required` |
| A06 | Human review | Decision can be changed **with explanation**; AI's original preliminary result remains visible |
| A07 | Graph | Explorer shows real nodes/edges from Neptune and updates after a new patient/assessment |
| A08 | Strands chat | Agent responds using objects actually retrieved via tools, with grounded explanations |
| A09 | Agent approval | **No** changes before approval; Reject makes **no** changes; Approve executes the authorized action only once |
| A10 | Evaluation | At least six **manually reviewed** labeled test cases spanning all three classes, showing Expected vs. Actual and Pass Rate |
| A11 | Deployment | Reproducible CDK deployment, short demo instructions, and working cleanup |

**No fabricated accuracy numbers:** The 42 original patients are **not** already a validated gold-standard suite. First label 6–9 clinically reviewed synthetic cases; optionally expand the set later.

---

## 13. Provided Materials and Useful References

**Available source materials:** `T001.pdf`, `patients.csv`, `patient_trial_link.csv`, original YouTube transcript for *Your First Agentic AIP Workflow*, Data Lineage diagram, Workshop Variable Lineage diagram (later version), original AIP Logic System/Task prompts, and the Pipeline Builder description.

**Implementation references (verify current SDK versions):**
- Neptune openCypher: https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-opencypher.html
- Neptune Serverless capacity: https://docs.aws.amazon.com/neptune/latest/userguide/neptune-serverless-capacity-scaling.html
- Bedrock Converse: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Converse.html
- Strands HITL: https://strandsagents.com/docs/user-guide/sdk/agents/interventions/human-in-the-loop/

**Decision rule:** If an additional AWS component, infrastructure abstraction, or feature is not necessary to visibly deliver one of the five wow moments, **do not build it**.
