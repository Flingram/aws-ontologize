# Requirements Document

## Introduction

This feature is a deliberately small but visually impressive AWS showcase for an **Ontology & Strands Agent platform**, built on the authoritative baseline `PROJECT_SPECIFICATION.md` v1.1. The first and only use case is **Glycurex T001 — Clinical Trial Recruitment** using synthetic data only.

The guiding principle is **SHOWCASE FIRST**: preserve all ten agreed features (F01–F10) but implement each with the minimum code and infrastructure needed to deliver five "wow moments" (PDF → ontology, patient → AI matching, live Neptune graph, chat with the ontology, agentic action). This document deliberately excludes production-grade scope (multi-tenancy, RBAC matrices, compliance audits, KMS architecture, WAF, event sourcing, workflow orchestrators).

**Amazon Neptune Database Serverless is the real, sole operational graph store.** No mock graph. S3 holds protocol PDFs only. Every requirement below is intentionally minimal; none introduces scope the baseline explicitly excludes.

Acceptance criteria are structured to map cleanly onto the baseline acceptance tests A01–A11. The mapping is noted per requirement.

## Glossary

- **Showcase**: The complete deployed demo application (frontend, backend, Neptune, S3, auth) for the Glycurex T001 use case.
- **Frontend**: The single-page React + TypeScript + AWS Cloudscape web application with five navigation items.
- **API_Lambda**: The single Python API Lambda behind the HTTP API that hosts the Strands Agent and PDF extraction logic.
- **Graph_Lambda**: The single Python Lambda inside the Neptune VPC that executes all Neptune openCypher access through the Ontology_Repository.
- **Matching_Worker**: The single Python Lambda invoked asynchronously to evaluate a patient against active trials.
- **Ontology_Repository**: The small reusable repository exposing `getObject`, `findObjects`, `getRelatedObjects`, `createObject`, and `createRelationship`, with config-driven allowed types.
- **Neptune**: The single private Amazon Neptune Database Serverless cluster (Property Graph, openCypher) that is the sole operational graph store.
- **Bedrock_Model**: The one configurable Amazon Bedrock model, referenced by an EU cross-region inference profile ID in `eu-central-1`, used for both PDF criteria extraction and eligibility evaluation.
- **Strands_Agent**: The single Strands Agent running inside the API_Lambda with three read-only tools and two approval-gated write actions.
- **Patient**: A synthetic patient graph node (`id`, `name`, eleven original clinical attributes, `synthetic=true`, `createdAt`).
- **ClinicalTrial**: A trial graph node (`id`, `name`, `status`, `criteriaSummary`, `criteriaJson`, `protocolS3Key`, `protocolVersion`, optional details).
- **Assessment**: An evaluation result graph node with deterministic id `patientId#trialId`.
- **Decision**: One of exactly three values — `Suitable`, `Not suitable`, `Manual review required`.
- **Processing_Status**: The lifecycle state of an Assessment — `Pending`, `Running`, `Completed`, or `Failed`.
- **Active_Trial**: A ClinicalTrial whose `status` is `Active`.
- **Seeded_Patients**: The 42 synthetic patients imported from `patients.csv`.
- **Demo_User**: The single authenticated user identity provided by the demo login.
- **CDK_Project**: The single AWS CDK TypeScript project that provisions and manages all Showcase infrastructure.
- **EU_Inference_Profile_Id**: The configurable Bedrock cross-region inference profile model ID (e.g., `eu.anthropic...`) required for model access in `eu-central-1`.

## Requirements

### Requirement 1: Phase 0 Neptune Connectivity Milestone

**User Story:** As a developer, I want a proven real Neptune connectivity milestone before business logic, so that the Showcase is built on a confirmed graph foundation rather than assumptions.

#### Acceptance Criteria

1. WHEN the Phase 0 smoke test runs, THE Graph_Lambda SHALL persist a uniquely identified smoke-test node via an openCypher write statement and then read that same node back via an openCypher read statement.
2. WHEN the Phase 0 smoke test runs, THE Graph_Lambda SHALL connect to Neptune using TLS and AWS IAM authentication.
3. WHEN the smoke-test read completes, THE Showcase SHALL confirm the read value is identical to the written value and report it as evidence of real Neptune read/write connectivity.
4. IF Neptune is unreachable or AWS IAM authentication fails during the Phase 0 smoke test, THEN THE Graph_Lambda SHALL return a descriptive connectivity error.
5. WHEN the Phase 0 smoke test completes, THE Graph_Lambda SHALL make a best-effort attempt to remove the smoke-test node.
6. THE Phase 0 milestone SHALL be considered complete only when a smoke test returns a read value identical to the written value, and SHALL be completed before any F01–F10 business logic depends on Neptune.

### Requirement 2: Seed T001 and 42 Synthetic Patients (F01, F02 — maps to A01)

**User Story:** As a demo presenter, I want T001 and the 42 original synthetic patients already present, so that the Recruitment Hub immediately establishes the use case.

#### Acceptance Criteria

1. WHEN the seed operation runs, THE Showcase SHALL create one ClinicalTrial node for T001 and 42 Patient nodes derived from `patients.csv`.
2. WHEN importing each Seeded_Patient, THE Showcase SHALL set the `id`, `name`, and the eleven original clinical attributes to values identical to the corresponding fields in `patients.csv`.
3. WHEN creating a Patient node during seeding, THE Showcase SHALL set `synthetic` to true and set `createdAt` to a valid ISO-8601 timestamp recorded at creation time.
4. WHEN the seed operation runs again, THE Showcase SHALL perform the seed idempotently so that no duplicate T001 or Seeded_Patient nodes are created, leaving exactly one T001 ClinicalTrial node and the 42 Seeded_Patient nodes.
5. WHEN Seeded_Patients are imported, THE Showcase SHALL NOT trigger automatic eligibility matching for any Seeded_Patient.
6. WHERE a separate seeded-patient evaluation is requested, THE Showcase SHALL provide an explicit "Evaluate seeded patients" operation that is distinct from automatic matching.
7. IF the seed operation is interrupted before completion, THEN THE Showcase SHALL be safely re-runnable so that a subsequent run completes the seed without creating duplicate T001 or Seeded_Patient nodes.
8. WHEN additional demo patients are added after seeding, THE Showcase SHALL allow them to be created and SHALL NOT treat the initial 42-patient count as a limit on adding further patients.

### Requirement 3: Single-Page Application With Five Navigation Items (F01–F10 UI shell)

**User Story:** As a demo presenter, I want one cohesive web application, so that the entire demo journey runs without switching to the AWS Console.

#### Acceptance Criteria

1. THE Frontend SHALL present five navigation items: Recruitment Hub, Clinical Trials, Ontology Explorer, AI Agent, and Evaluation.
2. THE Frontend SHALL use React, TypeScript, and AWS Cloudscape components with a light theme.
3. WHILE an operation is in progress, THE Frontend SHALL display a loading state for that operation.
4. THE Frontend SHALL display the label "Synthetic Demo Data – Not for Clinical Use" to identify patient data as synthetic.

### Requirement 4: Recruitment Hub Patient Management (F02)

**User Story:** As a demo presenter, I want to search, filter, and view synthetic patients and add new ones, so that I can drive the recruitment journey from one screen.

#### Acceptance Criteria

1. THE Recruitment Hub SHALL display a patient table with a trial dropdown, a search control, and a status filter.
2. WHEN the Demo_User selects a patient, THE Recruitment Hub SHALL open a Review Patient detail overlay containing patient data, trial criteria, the AI Decision, and the explanation.
3. WHILE the patient table is filtered to a selected trial, THE Review Patient overlay SHALL display all trial Assessments for the selected patient.
4. THE Recruitment Hub SHALL provide an `Add Patient` form that accepts a new synthetic patient's attributes.

### Requirement 5: Automatic Eligibility Matching For New Patients (F03 — maps to A03)

**User Story:** As a demo presenter, I want a newly added patient evaluated against all active trials automatically, so that results appear without manual steps.

#### Acceptance Criteria

1. WHEN the Demo_User submits a new patient through the `Add Patient` form, THE Showcase SHALL create a Patient node in Neptune.
2. WHEN a new patient is created through the Recruitment Hub, THE Showcase SHALL invoke the Matching_Worker asynchronously with the new patient's id, and SHALL NOT invoke the Matching_Worker for Seeded_Patients imported from `patients.csv`.
3. WHEN the Matching_Worker runs for a new patient, THE Matching_Worker SHALL create exactly one Assessment for each Active_Trial.
4. WHILE a new patient's Assessment has not reached `Completed` or `Failed`, THE Frontend SHALL poll for its status without requiring a manual page refresh, and IF an Assessment has not reached `Completed` or `Failed` within 120 seconds, THEN THE Frontend SHALL stop polling and display a non-blocking status for that Assessment.
5. IF the Matching_Worker is invoked more than once for the same patient and trial, THEN THE Showcase SHALL update the existing Assessment identified by `patientId#trialId` rather than create an additional current Assessment.
6. WHEN a new patient is created and two trials are Active, THE Showcase SHALL produce exactly two Assessments for that patient, each with a Processing_Status and an explanation.

### Requirement 6: Three Eligibility Decisions (F04 — maps to A04)

**User Story:** As a demo presenter, I want every completed assessment to carry exactly one of three decisions, so that results are clear and bounded.

#### Acceptance Criteria

1. WHEN an Assessment reaches `Completed`, THE Showcase SHALL set its Decision to exactly one of `Suitable`, `Not suitable`, or `Manual review required`.
2. THE Showcase SHALL store a one-sentence `reasoning` value with each Completed Assessment.
3. THE Showcase SHALL demonstrate at least one example each of `Suitable`, `Not suitable`, and `Manual review required`.

### Requirement 7: Glycurex T001 Eligibility Evaluation (F04 — maps to A05)

**User Story:** As a demo presenter, I want T001 eligibility evaluated against the real protocol rules, so that decisions reflect the defined inclusion and exclusion criteria.

#### Acceptance Criteria

1. WHEN evaluating a patient for a trial, THE Showcase SHALL send the approved trial criteria and the eleven patient attributes to the Bedrock_Model and receive, for each criterion, a result of `MET`, `NOT_MET`, `UNKNOWN`, or `NOT_APPLICABLE` with brief evidence.
2. THE Showcase SHALL verify numeric limits — age, HbA1c, eGFR, and metformin duration — in Python code when the values are unambiguously extractable.
3. IF a required clinical attribute is missing or ambiguous, THEN THE Showcase SHALL record the corresponding criterion result as `UNKNOWN`.
4. IF at least one applicable criterion is definitively violated, THEN THE Showcase SHALL set the Decision to `Not suitable`.
5. IF no applicable criterion is definitively violated AND required information is missing, THEN THE Showcase SHALL set the Decision to `Manual review required`.
6. IF all applicable criteria are demonstrably satisfied, THEN THE Showcase SHALL set the Decision to `Suitable`.
7. WHEN a patient's HbA1c value is 6.9% for T001 with no other definite violation, THE Showcase SHALL set the Decision to `Not suitable`.
8. WHEN a patient's eGFR value is missing for T001 with no other definite violation, THE Showcase SHALL set the Decision to `Manual review required`.
9. THE Showcase SHALL treat the T001 inclusion rules as: age 30–70 inclusive, confirmed Type 2 Diabetes Mellitus, HbA1c 7.0–10.0% inclusive, stable metformin dose for at least three months, eGFR at least 60 mL/min/1.73m2, informed consent, and for women of childbearing potential effective contraception with a negative pregnancy test.
10. THE Showcase SHALL treat the T001 exclusion rules as: Type 1 or secondary diabetes, insulin or non-metformin antidiabetic medication within the last three months, eGFR below 60, pregnancy, inability to consent, and age outside 30–70.
11. WHERE applicability of a criterion is uncertain, THE Showcase SHALL record `UNKNOWN` rather than assume the criterion is satisfied.
12. WHEN an Assessment completes, THE Showcase SHALL store `decision`, `reasoning`, `criterionResultsJson`, the Bedrock_Model id, and an `evaluatedAt` value, and SHALL keep the evaluated trial `protocolVersion` identifiable from the Assessment.
13. THE Showcase SHALL present every AI-produced Decision as a preliminary screening result and SHALL NOT perform clinical clearance or automatic patient enrollment.
14. IF a technical, model, parsing, or schema-validation error occurs during evaluation, THEN THE Showcase SHALL set the Assessment Processing_Status to `Failed` and SHALL NOT assign a clinical Decision value.

### Requirement 8: Evaluation Error Handling (F04 — supports A04/A05 integrity)

**User Story:** As a demo presenter, I want technical failures kept separate from clinical decisions, so that the Showcase never fabricates an eligibility outcome.

#### Acceptance Criteria

1. IF a technical or parsing error occurs during evaluation, THEN THE Showcase SHALL set the Assessment Processing_Status to `Failed`.
2. IF an evaluation fails for technical reasons, THEN THE Showcase SHALL NOT assign a clinical Decision value to that Assessment.
3. IF the Bedrock_Model response does not conform to the expected structured schema, THEN THE Showcase SHALL treat the evaluation as `Failed`.

### Requirement 9: Human Review And Decision Override (F05 — maps to A06)

**User Story:** As a reviewer, I want to correct a decision with a mandatory explanation, so that human judgment is recorded without hiding the AI's original result.

#### Acceptance Criteria

1. WHEN the Demo_User chooses `Modify Decision`, THE Showcase SHALL require a non-empty explanation before saving the override.
2. WHEN a reviewed Decision is saved, THE Showcase SHALL store the overridden decision and its explanation on the Assessment while retaining the original AI preliminary Decision.
3. THE Review Patient overlay SHALL display the human override alongside the AI's original preliminary Decision.
4. WHEN a Decision is overridden, THE Showcase SHALL NOT silently replace the underlying AI finding.
5. WHEN human review is initiated from the Frontend, THE Showcase SHALL treat it as an explicit user action requiring no second confirmation dialog.

### Requirement 10: Evaluation Suite (F06 — maps to A10)

**User Story:** As a demo presenter, I want a small independently labeled test set with a pass rate, so that I can show evaluation quality without fabricated accuracy claims.

#### Acceptance Criteria

1. THE Showcase SHALL include at least six manually reviewed labeled test cases spanning all three Decision classes.
2. WHEN the Demo_User runs the evaluation, THE Evaluation view SHALL display a table of Patient, Trial, Expected, Actual, and Pass/Fail for each test case.
3. WHEN the evaluation completes, THE Evaluation view SHALL display a Pass Rate metric.
4. THE Showcase SHALL NOT treat the 42 Seeded_Patients as a validated gold-standard test set.

### Requirement 11: Ontology Explorer Live Graph (F07 — maps to A07)

**User Story:** As a demo presenter, I want an interactive graph queried live from Neptune, so that the audience sees real patient–assessment–trial relationships.

#### Acceptance Criteria

1. THE Ontology Explorer SHALL render Patient, Assessment, and ClinicalTrial nodes and their edges from live Neptune data.
2. WHEN the Demo_User clicks a node, THE Ontology Explorer SHALL display that node's details in a side panel.
3. WHEN a new patient and its Assessments are created, THE Ontology Explorer SHALL reflect the new nodes and edges on a subsequent graph load.
4. THE Ontology Explorer SHALL provide at least one basic filter control.

### Requirement 12: PDF Protocol Upload And Criteria Extraction (F09 — maps to A02)

**User Story:** As a demo presenter, I want to upload a protocol PDF and have criteria extracted for review, so that a trial can be created from a document.

#### Acceptance Criteria

1. WHEN the Demo_User uploads a protocol PDF, THE Showcase SHALL store the PDF in the private S3 bucket and record its `protocolS3Key`.
2. WHEN extraction runs, THE API_Lambda SHALL use the Bedrock_Model's native PDF document support (visual page processing combined with any extractable text) to produce structured criteria covering the protocol's inclusion and exclusion criteria, including image-based PDFs such as the T001 protocol.
3. WHEN extraction completes, THE Clinical Trials view SHALL display the extracted criteria for review next to the uploaded PDF.
4. IF the uploaded PDF is encrypted or password-protected, exceeds the Bedrock_Model's supported size or page limits, or yields no usable criteria, THEN THE Showcase SHALL return a clear error, and THE Showcase SHALL NOT add a separate OCR or Textract pipeline.
5. THE Clinical Trials view SHALL allow the Demo_User to enter a few missing trial metadata fields in the review form before activation.

### Requirement 13: Trial Activation Lifecycle (F09, F10 — maps to A02)

**User Story:** As a demo presenter, I want trials to start as Draft and become Active only on explicit approval, so that matching uses only reviewed criteria.

#### Acceptance Criteria

1. WHEN a trial is created from an uploaded PDF, THE Showcase SHALL set its `status` to `Draft`.
2. WHEN the Demo_User clicks `Activate Trial`, THE Showcase SHALL change the trial `status` to `Active`.
3. WHEN a trial is activated, THE Showcase SHALL NOT retroactively create Assessments for existing patients.
4. THE Showcase SHALL use only user-reviewed criteria when evaluating subsequent new patients.
5. THE Showcase SHALL keep the original protocol PDF accessible after activation.
6. THE Showcase SHALL support trial statuses `Draft`, `Active`, and `Inactive` and allow multiple trials and multiple Assessments per patient.

### Requirement 14: Strands Agent Grounded Chat (F08 — maps to A08)

**User Story:** As a demo presenter, I want to ask the agent questions and get answers grounded in the graph, so that the chat demonstrably reads real Neptune data.

#### Acceptance Criteria

1. THE Showcase SHALL run exactly one Strands_Agent inside the API_Lambda.
2. THE Strands_Agent SHALL expose exactly three read-only tools: `get_object`, `find_objects`, and `get_related_objects`.
3. WHEN the Demo_User asks a question in the AI Agent chat, THE Strands_Agent SHALL retrieve objects through its read-only tools and ground its answer in the retrieved objects.
4. WHILE the Strands_Agent is using tools, THE AI Agent view SHALL display the tool activity.
5. THE AI Agent view SHALL provide predefined example prompts.
6. THE Showcase SHALL NOT provide a `CreatePatient` agent tool.
7. THE Showcase SHALL NOT expose arbitrary Cypher execution as an agent tool.

### Requirement 15: Agent Approval-Gated Writes (F08 — maps to A09)

**User Story:** As a demo presenter, I want every agent-initiated write gated by explicit approval, so that no mutation happens before I approve and approval runs the action exactly once.

#### Acceptance Criteria

1. THE Strands_Agent SHALL expose exactly two write actions: `reevaluate_eligibility` and `update_assessment`.
2. WHEN the Strands_Agent identifies a required write action, THE Showcase SHALL return an unexecuted action proposal carrying a unique proposal identifier, the action type, and its concrete arguments for the Frontend to display with `Approve` and `Reject`.
3. THE Showcase SHALL retain each pending proposal as minimal short-lived server-side state with a limited lifetime.
4. WHILE a write proposal is pending, THE Showcase SHALL NOT perform any graph mutation for that proposal.
5. WHEN the Demo_User rejects a proposal, THE Showcase SHALL make no graph mutation for that proposal and SHALL discard the proposal.
6. WHEN the Demo_User approves a proposal, THE Showcase SHALL validate server-side that the action type is one of the two allowed write actions, that the required arguments are present and valid, and that the request originates from the Demo_User, before any execution.
7. IF server-side validation of an approved proposal fails, THEN THE Showcase SHALL perform no graph mutation for that proposal and SHALL display an error message indicating the validation failure.
8. WHEN an approved proposal passes server-side validation, THE Showcase SHALL execute the approved action idempotently and SHALL mark the proposal consumed so that a repeated approval of the same proposal does not apply the change again.
9. IF an approval is received for a proposal that is already consumed, discarded, or expired, THEN THE Showcase SHALL perform no graph mutation and SHALL display a message indicating the proposal is no longer actionable.
10. WHEN an approved action completes, THE Showcase SHALL update the affected Assessment and graph in the Frontend and display a success or error message.

### Requirement 16: Ontology Data Model And Repository (F07 — supports A01/A07)

**User Story:** As a developer, I want a small reusable typed repository over three node types, so that all graph access is consistent without building a metamodel engine.

#### Acceptance Criteria

1. THE Showcase SHALL represent graph data using exactly three node types — Patient, ClinicalTrial, and Assessment — with the properties defined in the Glossary.
2. THE Showcase SHALL connect nodes using the edges `(:Patient)-[:HAS_ASSESSMENT]->(:Assessment)-[:FOR_TRIAL]->(:ClinicalTrial)`.
3. THE Showcase SHALL assign each Assessment a deterministic id of the form `patientId#trialId` with create-or-update behavior.
4. THE Ontology_Repository SHALL expose `getObject`, `findObjects`, `getRelatedObjects`, `createObject`, and `createRelationship` with config-driven allowed types.
5. THE Showcase SHALL keep domain eligibility rules outside the Ontology_Repository.
6. THE Showcase SHALL persist complex criteria and assessment data as JSON-string properties and deserialize them when returning API responses.
7. WHEN an Assessment is re-evaluated, THE Showcase SHALL retain at least the immediately previous result in a compact `previousResultsJson` history on the Assessment.
8. THE Showcase SHALL NOT implement a metamodel engine, a visual ontology editor, event sourcing, or separate audit/pending-action node types.

### Requirement 17: HTTP API Surface (supports A01–A10)

**User Story:** As a developer, I want one grouped HTTP API, so that the Frontend drives the entire journey through defined endpoints.

#### Acceptance Criteria

1. THE Showcase SHALL expose `GET /health`.
2. THE Showcase SHALL expose `GET /patients`, `POST /patients`, and `GET /patients/{id}`.
3. THE Showcase SHALL expose `GET /trials` and `GET /trials/{id}`.
4. THE Showcase SHALL expose `POST /trials/upload`, `POST /trials/extract`, and `POST /trials/{id}/activate`.
5. THE Showcase SHALL expose `GET /assessments` accepting `patientId` and `trialId` query parameters.
6. THE Showcase SHALL expose `POST /assessments/{id}/review`.
7. THE Showcase SHALL expose `GET /graph/neighborhood` accepting an `objectId` query parameter.
8. THE Showcase SHALL expose `POST /agent/chat`, `POST /agent/confirm`, and `POST /agent/reject`.
9. THE Showcase SHALL expose `POST /evaluation/run`.

### Requirement 18: Bedrock Model Configuration (supports A02/A04/A05)

**User Story:** As a developer, I want one configurable Bedrock model that works in the chosen region, so that model access works for both extraction and evaluation regardless of whether access is direct or via an inference profile.

#### Acceptance Criteria

1. THE Showcase SHALL use one configurable Bedrock_Model for both PDF criteria extraction and eligibility evaluation.
2. THE Showcase SHALL reference the Bedrock_Model by a configurable model identifier that MAY be either a supported direct model id or an EU cross-region inference profile id, chosen based on actual availability and compatibility in the selected region.
3. THE Showcase SHALL validate Bedrock_Model output against a structured schema before using it.
4. THE Showcase SHALL treat uploaded PDFs and prompt content as untrusted input and SHALL NOT execute external commands or model-generated Cypher derived from that content.

### Requirement 19: Compact AWS Architecture (supports A07/A11)

**User Story:** As a developer, I want a minimal architecture with Neptune kept private, so that the Showcase is small, secure, and cheap to run.

#### Acceptance Criteria

1. THE Showcase SHALL host the Frontend as static hosting using either S3/CloudFront or Amplify Hosting, choosing exactly one.
2. THE Showcase SHALL expose the backend through an API Gateway HTTP API in front of one API_Lambda.
3. THE Showcase SHALL use one private Neptune Database Serverless cluster as the sole operational graph store.
4. THE Showcase SHALL adopt the three-Lambda arrangement (API_Lambda, Graph_Lambda, Matching_Worker) as the preferred minimal design, placing only the Graph_Lambda inside the Neptune VPC while the API_Lambda and Matching_Worker run outside the VPC and reach Neptune only by invoking the Graph_Lambda via AWS Lambda Invoke using IAM.
5. THE Showcase SHALL store protocol PDFs in one private S3 bucket.
6. THE Showcase SHALL NOT provision a NAT Gateway, Step Functions, SQS, a DLQ, Fargate, EKS, WAF, DynamoDB, Aurora, RDS, or OpenSearch.
7. THE Showcase SHALL NOT add AWS services beyond this minimal architecture unless the addition is technically justified to deliver a required feature.
8. THE Showcase SHALL use eu-central-1 as the default region only when all required resources are available there.

### Requirement 20: Security Essentials (supports A09/A11)

**User Story:** As a developer, I want the small non-negotiable safeguards, so that the demo is secure without enterprise overhead.

#### Acceptance Criteria

1. THE Showcase SHALL store only synthetic patients and SHALL label patient data "Synthetic Demo Data – Not for Clinical Use".
2. THE Showcase SHALL keep Neptune private and SHALL NOT make Neptune reachable from the browser or the public internet.
3. THE Showcase SHALL require TLS and AWS IAM for Neptune access.
4. THE Showcase SHALL require one Demo_User login for the web application.
5. THE Showcase SHALL NOT place AWS keys in the browser and SHALL NOT expose an unauthenticated public write API.
6. WHEN an agent-initiated write is approved, THE Showcase SHALL validate allowed types, ids, and parameters server-side and SHALL prevent trivial duplicate execution.

### Requirement 21: CDK Deployment And Teardown (F10 operational baseline — maps to A11)

**User Story:** As a developer, I want the Showcase deployable and removable via CDK with a clear cleanup checklist, so that I can tear down the demo and understand any resources that remain.

#### Acceptance Criteria

1. THE Showcase SHALL provision and manage all AWS infrastructure through one AWS CDK TypeScript project.
2. WHEN `cdk deploy` runs, THE CDK_Project SHALL deploy the complete Showcase.
3. WHEN `cdk destroy` runs, THE CDK_Project SHALL remove the Showcase resources it manages, including Neptune, S3 objects, and CloudWatch logs, using removal policies appropriate for demo resources.
4. THE CDK_Project SHALL avoid relying on manually created AWS resources, and any unavoidable manually created, retained, or bootstrap resource SHALL be documented.
5. THE Showcase SHALL document a simple cleanup checklist covering any resources not automatically removed by `cdk destroy`, such as CDK bootstrap resources or retained logs.
6. THE Showcase SHALL document a rough Neptune Serverless cost expectation stating a minimum of 1.0 NCU and continuous idle cost that never scales to zero, together with teardown instructions, and SHALL NOT claim that all AWS charges cease automatically.
