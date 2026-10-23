---

description: "Task list for Incoming Goods Conformity Ledger"
---

# Tasks: Incoming Goods Conformity Ledger

**Input**: Design documents from `/specs/001-goods-conformity-ledger/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Included. Constitution Principle V makes test-first chaincode NON-NEGOTIABLE, and
the workflow gate requires integration tests on a multi-node network. Chaincode tests MUST
be written and seen failing before the matching implementation task.

**Organization**: Tasks are grouped by user story so each story can be built and tested on
its own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1–US5)

## Path Conventions

- Chaincode: `chaincode/conformity/`
- Gateway: `gateway/cmd/`, `gateway/internal/`, `gateway/migrations/`
- Web: `web/src/`, `web/tests/`
- Network: `network/compose/`, `network/config/`, `network/scripts/`
- Cross-component tests: `tests/integration/`; fixtures: `testdata/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Repository layout, toolchains, local 3-org network

- [ ] T001 Create the directory layout from plan.md (`chaincode/conformity/`, `gateway/cmd/gateway/`, `gateway/internal/{api,fabric,store,projector,resultsfile,notify,keys,config}/`, `gateway/migrations/`, `web/`, `network/{compose,config,scripts}/`, `tests/integration/`, `testdata/`, `tools/detcheck/`) and a root `Makefile` with targets `network-up`, `network-down`, `chaincode-deploy`, `stack-up`, `seed`, `test-chaincode`, `test-integration`, `test-web`
- [ ] T002 Initialize Go module in `chaincode/conformity/go.mod` (Go 1.23) with `github.com/hyperledger/fabric-contract-api-go/v2`, `github.com/hyperledger/fabric-chaincode-go/v2`, `github.com/shopspring/decimal`
- [ ] T003 [P] Initialize Go module in `gateway/go.mod` (Go 1.23) with `github.com/hyperledger/fabric-gateway`, `github.com/jackc/pgx/v5`, `github.com/coreos/go-oidc/v3`, `github.com/prometheus/client_golang`; logging uses stdlib `log/slog`
- [ ] T004 [P] Scaffold web app in `web/` with Vite + React 18 + TypeScript 5, add `react-i18next`, `i18next`, `oidc-client-ts`, `vitest`, `@testing-library/react`, `@playwright/test`
- [ ] T005 [P] Add Go lint config `.golangci.yml` (govet, staticcheck, errcheck) and web lint config `web/eslint.config.js`
- [ ] T006 [P] Implement determinism analyzer in `tools/detcheck/main.go` using `golang.org/x/tools/go/analysis` that fails on: any use of `float32`/`float64`, `time.Now`, `math/rand`, imports of `net`, `net/http`, `os`, and `range` over a map type; wire it into `make test-chaincode` against `chaincode/conformity/...`
- [ ] T007 Write Fabric channel config `network/config/configtx.yaml` with orgs `StdOrgMSP`, `Lab1MSP`, `Lab2MSP`, 3 Raft orderers owned by StdOrg, channel `conformity`, and Fabric CA server configs in `network/config/ca/{stdorg,lab1,lab2}/fabric-ca-server-config.yaml`
- [ ] T008 Write `network/compose/docker-compose.fabric.yaml` with 3 CAs, `orderer{0,1,2}.stdorg`, `peer0.stdorg`, `peer1.stdorg`, `peer0.lab1`, `peer0.lab2` (LevelDB state, Prometheus metrics enabled)
- [ ] T009 Write `network/scripts/network-up.sh` (enroll org admins, create channel `conformity`, join all peers) and `network/scripts/deploy-chaincode.sh` (package, install on all peers, approve by every org, commit with endorsement policy `AND('StdOrgMSP.peer', OutOf(1, 'Lab1MSP.peer', 'Lab2MSP.peer'))`)
- [ ] T010 Write `network/compose/docker-compose.stack.yaml` with PostgreSQL 16 for StdOrg and Lab1, Keycloak 26 importing `network/config/keycloak/stdorg-realm.json` and `network/config/keycloak/lab1-realm.json` (seed users `inspector1`, `assessor1`, `assessor2`, `techofficer1`, `techofficer2`, `auditor1`, `admin1`, `importer1` on StdOrg, `lab1user` on Lab1; password `Passw0rd!`; TOTP off in dev), StdOrg gateway on `:8080` and Lab1 gateway on `:8081`
- [ ] T011 [P] Add CI workflow `.github/workflows/ci.yml` running `make test-chaincode`, `make network-up chaincode-deploy stack-up test-integration`, `make test-web`; any failure blocks merge

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared chaincode core, identity/access, rule-set storage, gateway skeleton, web shell

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Chaincode core

- [ ] T012 Define all on-chain structs and enums from data-model.md in `chaincode/conformity/types.go` (Participant, Consignment, DocumentRef, Correction, Sample, TestResult, RuleSet, RuleTest, Verdict, Decision, Appeal, Disposition); decimal fields are `string`; roles enum `inspector`, `lab`, `assessor`, `techOfficer`, `importer`, `auditor`, `admin`, `service`
- [ ] T013 [P] Define error codes `ERR_FORBIDDEN`, `ERR_NOT_FOUND`, `ERR_INVALID`, `ERR_STATE`, `ERR_SOD`, `ERR_HASH_MISMATCH` and a constructor returning `CODE: message` in `chaincode/conformity/errors.go`
- [ ] T014 [P] Build in-memory fake stub for tests in `chaincode/conformity/fakestub_test.go`: world state, per-key history, fixed tx timestamp, tx ID, and configurable caller MSP + cert attributes `role`, `participantId`
- [ ] T015 Write state helpers in `chaincode/conformity/state.go`: composite keys `<type>~<id>`, `getJSON`/`putJSON`, `emit(eventName, payload)` named after the calling function, `txTime()` from `stub.GetTxTimestamp()` only, and `sortedKeys` helper (never range maps for output)
- [ ] T016 Write failing tests in `chaincode/conformity/access_test.go`: unknown participant → `ERR_FORBIDDEN`; revoked participant (action after `revokedAt`) → `ERR_FORBIDDEN`; wrong role → `ERR_FORBIDDEN`; correct role passes
- [ ] T017 Implement caller identity and `requireRole(roles...)` in `chaincode/conformity/access.go` reading `role` and `participantId` cert attributes via `cid`, checking Participant `status` is `active`
- [ ] T018 Write failing tests in `chaincode/conformity/participant_test.go`: `RegisterParticipant` only by `admin` of the same `orgMsp`; `RevokeParticipant` sets `status` `revoked` and `revokedAt`; past entries signed by the revoked participant stay readable (FR-020)
- [ ] T019 Implement `RegisterParticipant` and `RevokeParticipant` in `chaincode/conformity/participant.go`, plus a one-time `InitLedger` that registers the first StdOrg admin from the deploying identity
- [ ] T020 Write failing tests in `chaincode/conformity/ruleset_test.go`: `DraftRuleSet` by `techOfficer` creates `version` = previous + 1, `status` `draft`; every RuleTest requires `clause`; `ApproveRuleSet` by the drafter → `ERR_SOD`; approved rule set is immutable; `effectiveFrom` must be ≥ approval date; `GetRuleSet(id, date)` returns the highest approved version with `effectiveFrom` ≤ date; `unitFactors` stored as list sorted by unit
- [ ] T021 Implement `DraftRuleSet`, `ApproveRuleSet`, `GetRuleSet` (by version or in force on a date) in `chaincode/conformity/ruleset.go`
- [ ] T022 Register the contract and route all functions in `chaincode/conformity/contract.go` and `chaincode/conformity/main.go` (chaincode-as-a-service entry point)
- [ ] T023 Write `network/scripts/seed.sh` that enrolls seed users with Fabric CA attributes, registers participants, and drafts (`techofficer1`) + approves (`techofficer2`) `testdata/rulesets/iec60335-1-ed6-v1.json` via `peer chaincode invoke`; create that rule-set fixture with at least one numeric test with inclusive limit (`IR_MIN`, `MOhm`, min `"2"`), one with exclusive max and unit factor (`LEAK_MAX`, `mA`, `A -> mA = 1000`), and one qualitative test (`LABEL`)

### Gateway core

- [ ] T024 Implement env config loading in `gateway/internal/config/config.go`: org MSP, ordered peer endpoint list (own peer first, StdOrg peer as fallback for lab gateways), DB URL, OIDC issuer/client, SMTP relay, key-wrapping key file path, listen address
- [ ] T025 Write off-chain schema `gateway/migrations/001_init.sql` for tables `importer`, `app_user` (`calendar` in `jalali`/`gregorian`, `locale` in `fa`/`en`), `user_key`, `document` (CHECK `size <= 26214400`), `results_upload`, `notification`, `rm_consignment`, `rm_verdict`, `rm_decision`, `rm_disposition`, `projection_checkpoint`; apply embedded migrations in order at startup from `gateway/internal/store/migrate.go`
- [ ] T026 [P] Implement pgx pool and user/key queries in `gateway/internal/store/store.go` and `gateway/internal/store/users.go`
- [ ] T027 [P] Implement AES-256-GCM wrap/unwrap of user private keys with the wrapping key file in `gateway/internal/keys/keys.go` with a round-trip test `gateway/internal/keys/keys_test.go` (`// ponytail: file wrapping key, move to HSM for production`)
- [ ] T028 Implement Fabric client in `gateway/internal/fabric/client.go`: per-request signer built from the authenticated user's unwrapped key, `Submit`/`Evaluate`, failover across the configured peer list, parse `ERR_*` code from chaincode errors, return `txId`
- [ ] T029 [P] Implement user enrollment with Fabric CA (register + enroll with attributes `role`, `participantId`, store wrapped key) in `gateway/internal/fabric/enroll.go`
- [ ] T030 Implement OIDC bearer middleware in `gateway/internal/api/auth.go` (go-oidc verifier, map `sub` to `app_user`, roles from token, require a second-factor `acr` claim for every role except `importer`)
- [ ] T031 [P] Implement error mapping in `gateway/internal/api/errors.go` (`ERR_INVALID` 422, `ERR_FORBIDDEN`/`ERR_SOD` 403, `ERR_NOT_FOUND` 404, `ERR_STATE` 409, `ERR_HASH_MISMATCH` 409; body `{code, message, details}`) and fa/en message catalog in `gateway/internal/api/messages.go` selected by `Accept-Language`
- [ ] T032 Implement router and server in `gateway/internal/api/router.go` and `gateway/cmd/gateway/main.go` with `/healthz`, `/metrics`, slog JSON logger adding `txId` and `consignmentId` fields, serving the built SPA from `web/dist`
- [ ] T033 Implement block event projector skeleton in `gateway/internal/projector/projector.go`: resume from `projection_checkpoint`, dispatch events by name to registered handlers, update checkpoint in the same DB transaction
- [ ] T034 Implement participants endpoints `POST /api/v1/participants` and `POST /api/v1/participants/{id}/revoke` (admin) in `gateway/internal/api/participants.go`

### Web shell

- [ ] T035 [P] Set up i18n in `web/src/i18n/index.ts` with `web/src/i18n/fa.json` (default) and `web/src/i18n/en.json`; set `<html lang>` and `<html dir="rtl|ltr">` on language change; use CSS logical properties only in `web/src/styles.css`
- [ ] T036 [P] Implement `formatDate`/`formatDateTime` in `web/src/lib/date.ts` using `Intl.DateTimeFormat('fa-IR-u-ca-persian')` or `'en-GB'` per user calendar, input always UTC ISO strings; test in `web/tests/date.test.ts`
- [ ] T037 Implement OIDC login with `oidc-client-ts` in `web/src/auth.ts` and API wrapper in `web/src/lib/api.ts` (bearer token, `Accept-Language`, error code → translated message)
- [ ] T038 Implement app shell with role-based routes and user settings (language, calendar) in `web/src/App.tsx` and `web/src/pages/Settings.tsx`

### Integration harness

- [ ] T039 Implement integration helpers in `tests/integration/harness_test.go`: tokens via `network/scripts/token.sh <user>`, JSON HTTP helpers for `:8080`/`:8081`, `dockerStop`/`dockerStart`, direct DB access for tamper tests

**Checkpoint**: Network up, chaincode deployed with participants and pilot rule set, gateway authenticates users and signs per person, web shell logs in

---

## Phase 3: User Story 1 - Register an Incoming Consignment as a Tamper-Proof Record (Priority: P1) 🎯 MVP

**Goal**: Inspectors register consignments with documents; history is append-only; altered documents are detected

**Independent Test**: quickstart.md Scenario 1 — register, correct, tamper with invoice bytes, verify shows `mismatch`, missing `standardRef` → 422

### Tests for User Story 1 ⚠️ (write first, must fail)

- [ ] T040 [P] [US1] Tests in `chaincode/conformity/consignment_test.go`: `RegisterConsignment` only by `inspector`; required `importerId`, `productCategory`, `standardRef` `{number, edition}`; `productDescription` "Max 500 chars"; `quantity` "> 0"; `origin` "ISO 3166-1 alpha-2"; `arrivalDate` "≤ registration time"; status becomes `Registered`; duplicate ID → `ERR_INVALID`
- [ ] T041 [P] [US1] Tests in `chaincode/conformity/correction_test.go`: `CorrectConsignment` requires `reason` ("required, max 500 chars"); `consignmentId` and `registeredBy` not correctable; key history keeps old and new values; event carries `{targetId, field, oldValue, newValue, reason, by}`
- [ ] T042 [P] [US1] Tests in `chaincode/conformity/documents_test.go`: `AttachDocument` append-only by `inspector`/`lab`; `sha256` "64 chars" hex; `kind` in `invoice`, `packingList`, `photo`, `testReport`, `dispositionEvidence`, `other`; `RecordDocumentRemoval` admin-only, `removed` "never set back to false"; `VerifyDocument` returns `ok`/`mismatch`/`removed`
- [ ] T043 [P] [US1] Tests in `chaincode/conformity/split_test.go`: `SplitConsignment` creates child consignments with `parentId`, parent → `Split`; refused on final states with `ERR_STATE`
- [ ] T044 [P] [US1] Tests in `chaincode/conformity/history_test.go`: `GetHistory` returns entries ordered by timestamp with `txId`, `timestamp`, `function`, `actor`, `payload`; refused for `importer` and `lab`
- [ ] T045 [P] [US1] Integration test for quickstart Scenario 1 in `tests/integration/us1_registration_test.go`

### Implementation for User Story 1

- [ ] T046 [US1] Implement the consignment state machine table (all transitions in data-model.md "Consignment state machine"; anything else → `ERR_STATE`) in `chaincode/conformity/statemachine.go`
- [ ] T047 [US1] Implement `RegisterConsignment`, `CorrectConsignment`, `SplitConsignment`, `GetConsignment` in `chaincode/conformity/consignment.go`
- [ ] T048 [P] [US1] Implement `AttachDocument`, `RecordDocumentRemoval`, `VerifyDocument` in `chaincode/conformity/documents.go`
- [ ] T049 [P] [US1] Implement `GetHistory` (via `GetHistoryForKey` on the consignment and its child keys) in `chaincode/conformity/history.go`
- [ ] T050 [P] [US1] Implement importer personal-data store (never sent to chain) in `gateway/internal/store/importers.go`
- [ ] T051 [P] [US1] Implement document store in `gateway/internal/store/documents.go`: save bytes ≤ 25 MB, compute SHA-256, null bytes and set `removed_at` on legal deletion
- [ ] T052 [US1] Implement endpoints `POST /consignments`, `GET /consignments/{id}`, `GET /consignments/{id}/history`, `POST /consignments/{id}/corrections`, `POST /consignments/{id}/split`, `POST|GET|DELETE /consignments/{id}/documents[/{docId}]`, `GET /consignments/{id}/documents/{docId}/verify` in `gateway/internal/api/consignments.go`; importers see only own consignments (FR-021)
- [ ] T053 [US1] Add projector handlers for `RegisterConsignment`, `CorrectConsignment`, `SplitConsignment`, `AttachDocument`, `RecordDocumentRemoval` writing `rm_consignment` in `gateway/internal/projector/consignments.go`
- [ ] T054 [US1] Implement `GET /consignments` with filters `status`, `category`, `standard`, `importer`, `from`, `to` from the read model in `gateway/internal/api/consignments_list.go`
- [ ] T055 [P] [US1] Build inspector pages `web/src/pages/inspector/RegisterConsignment.tsx` (form + document upload, field errors in fa/en) and `web/src/pages/ConsignmentList.tsx`
- [ ] T056 [US1] Build `web/src/pages/ConsignmentDetail.tsx` with history timeline (actor, Jalali/Gregorian time), correction dialog, and per-document verify status
- [ ] T057 [P] [US1] Component test for registration validation messages in `web/tests/RegisterConsignment.test.tsx`

**Checkpoint**: US1 works alone — consignments registered, corrected, tamper detected

---

## Phase 4: User Story 2 - Automated Conformity Assessment from Test Results (Priority: P2)

**Goal**: Labs submit results (form or CSV); chaincode computes deterministic verdicts citing rule-set version

**Independent Test**: quickstart.md Scenario 2 — pass, boundary, fail, missing, bad-row CSV, reassess identical

### Tests for User Story 2 ⚠️ (write first, must fail)

- [ ] T058 [P] [US2] Table-driven tests in `chaincode/conformity/assess_test.go`: all within limits → `Conforming`; value equal to inclusive limit → pass, equal to exclusive limit → fail; any fail → `NonConforming` with `measured` and `limit`; missing required test → `missing` + `RequiresReview`; two differing results same sample/test → `conflict` + `RequiresReview`; `A` converted to `mA` by factor; unknown unit → `unitError`; no approved rule set in force on `arrivalDate` → `RequiresReview`; qualitative `pass`/`fail`; same inputs + version → identical verdict and `inputsHash`; verdict cites version in force on `arrivalDate`, not latest
- [ ] T059 [P] [US2] Tests in `chaincode/conformity/results_test.go`: `RecordSample` by `inspector`, `takenAt` "≥ arrivalDate", status `Registered` → `Sampling`; `SubmitResults` by `lab` whose MSP equals sample `assignedLabMsp`; `testCode` "Must exist in applicable rule set"; `value` decimal string required for numeric, `passFail` for qualitative; any invalid result rejects the whole batch; status → `ResultsReceived`
- [ ] T060 [P] [US2] Tests in `chaincode/conformity/manual_verdict_test.go`: `RecordManualVerdict` by `assessor` requires justification, sets `manual=true`; `ReassessDryRun` writes nothing and returns the original verdict
- [ ] T061 [P] [US2] Tests in `gateway/internal/resultsfile/validate_test.go` per contracts/results-file-template.md: exact header, Persian/Arabic digits rejected, max 18 significant digits, `pass_fail` only `pass`/`fail`, `method` max 100 chars, max 5,000 rows and 5 MB, duplicate `(sample_id, test_code)` rejected, every error reported as `{row, column, code, message}`
- [ ] T062 [P] [US2] Create CSV fixtures `testdata/results-pass.csv`, `testdata/results-boundary.csv`, `testdata/results-fail.csv`, `testdata/results-missing.csv`, `testdata/results-bad-row.csv` matching `testdata/rulesets/iec60335-1-ed6-v1.json`
- [ ] T063 [US2] Integration test for quickstart Scenario 2 in `tests/integration/us2_assessment_test.go`

### Implementation for User Story 2

- [ ] T064 [P] [US2] Implement `RecordSample` in `chaincode/conformity/samples.go`
- [ ] T065 [P] [US2] Implement `SubmitResults` in `chaincode/conformity/results.go`
- [ ] T066 [US2] Implement pure `evaluate(ruleSet, results) Verdict` using `shopspring/decimal` and sorted iteration, plus `AssessConsignment` (roles `assessor`, `service`), `RecordManualVerdict`, `ReassessDryRun` in `chaincode/conformity/assess.go`
- [ ] T067 [P] [US2] Implement CSV validation in `gateway/internal/resultsfile/validate.go`
- [ ] T068 [US2] Implement endpoints `POST /consignments/{id}/samples`, `POST /results`, `POST /results/upload`, `POST /consignments/{id}/assess`, `POST /consignments/{id}/manual-verdict`, `POST /consignments/{id}/reassess` in `gateway/internal/api/results.go`; no automatic assessment here (lab gateways have no `service` identity)
- [ ] T069 [US2] Add projector handlers for `RecordSample`, `SubmitResults`, `AssessConsignment`, `RecordManualVerdict` writing `rm_consignment`/`rm_verdict` in `gateway/internal/projector/verdicts.go`; on a `SubmitResults` event, the StdOrg gateway's projector calls `AssessConsignment` with the StdOrg `service` identity (lab gateways skip this step)
- [ ] T070 [P] [US2] Build lab pages `web/src/pages/lab/ResultsEntry.tsx` and `web/src/pages/lab/ResultsUpload.tsx` (per-row error table)
- [ ] T071 [US2] Add sample recording to `web/src/pages/ConsignmentDetail.tsx` and verdict panel `web/src/components/VerdictPanel.tsx` (per-test outcome, measured vs limit, rule-set version, manual flag)

**Checkpoint**: US1 + US2 work — results produce reproducible verdicts

---

## Phase 5: User Story 3 - Assessor Decision, Override, and Certificate Record (Priority: P3)

**Goal**: Assessors confirm/override with justification; certificates or rejections recorded; appeals and dispositions tracked; importers notified

**Independent Test**: quickstart.md Scenario 3

### Tests for User Story 3 ⚠️ (write first, must fail)

- [ ] T072 [P] [US3] Tests in `chaincode/conformity/decision_test.go`: `DecideConsignment` by `assessor`; `override` without `justification` → `ERR_INVALID` ("max 2000 chars"); `certificateRef` "Required when outcome `Conforming`"; `rejectionReasons` "Required when outcome `NonConforming`"; caller who submitted any result for the consignment, or from the submitting lab MSP → `ERR_SOD`; automated verdict and override both remain in history; a `documentChecks` hash that differs from the on-chain `sha256` → `ERR_HASH_MISMATCH`, and a missing check for any non-removed document → `ERR_INVALID`
- [ ] T073 [P] [US3] Tests in `chaincode/conformity/appeal_test.go`: `FileAppeal` only by `service` for the importer of record, `grounds` "max 2000 chars", status → `UnderAppeal`; `ResolveAppeal` by the original decider → `ERR_SOD`; resolution creates `appealOutcome` decision with `previousDecisionId`
- [ ] T074 [P] [US3] Tests in `chaincode/conformity/disposition_test.go`: `ChooseDisposition` requires final `NonConforming`, `type` in `reExport`/`destruction`/`rework`, status → `DispositionPending`; `CompleteDisposition` by `inspector` with `evidenceDocumentIds` "≥ 1 on completion"; `rework` → `Sampling`, others → `Closed`
- [ ] T075 [US3] Integration test for quickstart Scenario 3 in `tests/integration/us3_decisions_test.go`

### Implementation for User Story 3

- [ ] T076 [P] [US3] Implement `DecideConsignment` in `chaincode/conformity/decision.go`
- [ ] T077 [P] [US3] Implement `FileAppeal`, `ResolveAppeal` in `chaincode/conformity/appeal.go`
- [ ] T078 [P] [US3] Implement `ChooseDisposition`, `CompleteDisposition` in `chaincode/conformity/disposition.go`
- [ ] T079 [P] [US3] Implement notifications (SMTP via `net/smtp` + `notification` table rows, fa/en templates) in `gateway/internal/notify/notify.go`
- [ ] T080 [US3] Implement endpoints `POST /consignments/{id}/decisions`, `POST /consignments/{id}/appeals`, `POST /appeals/{id}/resolve`, `POST /consignments/{id}/dispositions`, `POST /dispositions/{id}/complete`, `GET /me/notifications` in `gateway/internal/api/decisions.go`; notify importer on final decision and appeal outcome; before calling `DecideConsignment`, recompute the SHA-256 of every non-removed document from `gateway/internal/store/documents.go` and pass them as `documentChecks`
- [ ] T081 [US3] Add projector handlers for decisions, appeals, dispositions writing `rm_decision`/`rm_disposition` (including certificate ref index) in `gateway/internal/projector/decisions.go`
- [ ] T082 [P] [US3] Build `web/src/pages/assessor/DecisionForm.tsx` (confirm/override, justification required for override) and `web/src/pages/inspector/Disposition.tsx`
- [ ] T083 [P] [US3] Build importer portal pages `web/src/pages/importer/MyConsignments.tsx`, `web/src/pages/importer/Appeal.tsx`, `web/src/pages/importer/Notifications.tsx`

**Checkpoint**: Full decision lifecycle works end to end

---

## Phase 6: User Story 4 - Independent Audit and Verification (Priority: P4)

**Goal**: Auditors verify history, documents, and verdicts read-only, matching lab peers

**Independent Test**: quickstart.md Scenario 4

- [ ] T084 [P] [US4] Test in `chaincode/conformity/auditor_readonly_test.go`: every write function called with role `auditor` → `ERR_FORBIDDEN`; `GetHistory`, `VerifyDocument`, `ReassessDryRun` allowed
- [ ] T085 [P] [US4] Write `network/scripts/peer-query.sh <org> <function> <args...>` running `peer chaincode query` against that org's peer
- [ ] T086 [US4] Integration test for quickstart Scenario 4 in `tests/integration/us4_audit_test.go` (history via gateway equals history from `peer0.lab1`)
- [ ] T087 [US4] Implement `GET /certificates/{ref}` in `gateway/internal/api/certificates.go`
- [ ] T088 [US4] Build `web/src/pages/auditor/AuditLookup.tsx` (by consignment ID or certificate number) and `web/src/pages/auditor/AuditConsignment.tsx` (history, verify all documents, reassess and show match/mismatch with recorded verdict)

**Checkpoint**: Auditors can verify any decision without operator help

---

## Phase 7: User Story 5 - Manage Conformity Rule Sets (Priority: P5)

**Goal**: Technical officers draft and approve rule-set versions through the UI with effective dates

**Independent Test**: quickstart.md Scenario 5

- [ ] T089 [US5] Integration test for quickstart Scenario 5 in `tests/integration/us5_rulesets_test.go` (self-approval `403 ERR_SOD`; future `effectiveFrom` keeps v1 in force; old verdicts still cite v1)
- [ ] T090 [US5] Implement endpoints `GET /rule-sets`, `GET /rule-sets/{id}/versions/{v}`, `POST /rule-sets`, `POST /rule-sets/{id}/versions/{v}/approve` in `gateway/internal/api/rulesets.go`
- [ ] T091 [P] [US5] Build `web/src/pages/techOfficer/RuleSetList.tsx` and `web/src/pages/techOfficer/RuleSetEditor.tsx` (tests with clause, unit, min/max, inclusivity, required, samplesRequired; unit factor table; effective date)

**Checkpoint**: All user stories independently functional

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T092 Implement `GET /reports/summary?groupBy=status|category|standard|importer&from=&to=` including override counts and pending dispositions in `gateway/internal/api/reports.go`, and `web/src/pages/Reports.tsx`
- [ ] T093 Resilience integration test for quickstart Scenario 6 in `tests/integration/resilience_test.go` (stop `peer0.lab1`, Lab1 submits via StdOrg peer, restart, histories match)
- [ ] T094 [P] Add read-model rebuild command `gateway/cmd/rebuild-readmodel/main.go` (truncate `rm_*`, reset checkpoint, replay from block 0)
- [ ] T095 [P] Add `network/scripts/add-lab.sh` (create lab MSP, channel config update, regenerate endorsement policy with the new lab, chaincode lifecycle approve/commit by all orgs) and document it in `README.md`
- [ ] T096 [P] Add Prometheus + blackbox probe config `network/config/prometheus/prometheus.yml` probing `/healthz` and an authenticated read every minute during working hours (SC-009)
- [ ] T097 [P] Playwright smoke test per story in `web/tests/e2e/us1.spec.ts` … `web/tests/e2e/us5.spec.ts`, each run in both `fa` and `en`
- [ ] T098 [P] Translation completeness test in `web/tests/i18n.test.ts` (every key in `fa.json` exists in `en.json` and vice versa)
- [ ] T099 Security hardening in `gateway/internal/api/router.go`: CSP and security headers, upload MIME sniffing against declared `kind`, request size limits (25 MB documents, 5 MB results files)
- [ ] T100 [P] Load test behind build tag `load` in `tests/load/load_test.go`: 500 consignments/day profile with 50 concurrent users; assert 95% of verdicts within 1 minute of results submission (SC-002, SC-008)
- [ ] T101 Run every quickstart.md scenario and `make test-chaincode test-integration test-web`; record results in `specs/001-goods-conformity-ledger/quickstart.md` notes
- [ ] T102 Write `network/scripts/upgrade-chaincode.sh` (package new version, approve by every org, commit with the same endorsement policy, record version and approving MSPs in the release entry) and the upgrade procedure section in `README.md`; add integration test `tests/integration/upgrade_test.go` showing consignments, verdicts, and history registered under v1 are unchanged and readable after upgrading to v2
- [ ] T103 [P] Add `CHANGELOG.md` with sections "Contract versions" and "Rule-set versions" (version, effective date, approvers) and a CI step in `.github/workflows/ci.yml` that fails when `chaincode/` or `testdata/rulesets/` changes without a `CHANGELOG.md` change

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none
- **Foundational (Phase 2)**: after Setup — BLOCKS all stories
- **User Stories (Phases 3–7)**: after Foundational
- **Polish (Phase 8)**: after the stories it touches

### User Story Dependencies

- **US1 (P1)**: after Foundational; no story dependencies
- **US2 (P2)**: needs consignments from US1 (T046, T047) and the rule set from Foundational (T021, T023)
- **US3 (P3)**: needs verdicts from US2 (T066)
- **US4 (P4)**: read-only over US1–US3 data; can start after US1 with partial scenarios
- **US5 (P5)**: chaincode already in Foundational; needs only gateway + web; can run in parallel with US1

### Within Each User Story

- Chaincode tests first and failing (Constitution V), then chaincode implementation
- Chaincode before gateway endpoints; endpoints before projector handlers and web pages
- Integration test passes before the story checkpoint

### Parallel Opportunities

- Setup: T003, T004, T005, T006, T011
- Foundational: T013, T014; T026, T027, T029, T031; T035, T036
- US1: all tests T040–T045; T048, T049, T050, T051; T055, T057
- US2: T058–T062; T064, T065, T067; T070
- US3: T072–T074; T076–T079; T082, T083
- US5 can be built by a second developer while US1 is in progress

---

## Parallel Example: User Story 1

```bash
# Write all failing chaincode tests together:
Task: "Tests in chaincode/conformity/consignment_test.go"
Task: "Tests in chaincode/conformity/correction_test.go"
Task: "Tests in chaincode/conformity/documents_test.go"
Task: "Tests in chaincode/conformity/split_test.go"
Task: "Tests in chaincode/conformity/history_test.go"

# Then independent implementation files:
Task: "Implement AttachDocument, RecordDocumentRemoval, VerifyDocument in chaincode/conformity/documents.go"
Task: "Implement GetHistory in chaincode/conformity/history.go"
Task: "Implement document store in gateway/internal/store/documents.go"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1 Setup → Phase 2 Foundational
2. Phase 3 US1
3. **STOP and VALIDATE**: quickstart Scenario 1 passes
4. Demo tamper-proof registration to stakeholders

### Incremental Delivery

1. US1 → tamper-proof registry (MVP)
2. US2 → automated verdicts
3. US3 → legally usable decisions (minimum for go-live)
4. US4 → accreditation-ready audit
5. US5 → self-service rule management (seed script covers it until then)

### Parallel Team Strategy

1. Whole team: Setup + Foundational
2. Then: Developer A on US1 → US2 → US3 chaincode; Developer B on gateway/web for the same
   stories; Developer C on US5 and US4

---

## Notes

- [P] = different files, no dependency on unfinished tasks
- Commit after each task or logical group
- Any change to `chaincode/conformity/` needs two reviewers (constitution workflow gate)
