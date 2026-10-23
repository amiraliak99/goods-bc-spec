# Phase 0 Research: Incoming Goods Conformity Ledger

All Technical Context unknowns are resolved below. Format per item: Decision / Rationale /
Alternatives considered.

## R1. Ledger platform

- **Decision**: Hyperledger Fabric 2.5 LTS, one channel `conformity`, Raft ordering service.
- **Rationale**: Permissioned by design with organization-level membership (MSP) that maps
  directly to "standards organization + one org per accredited lab" (Clarification Q1).
  No cryptocurrency or gas. Endorsement policies enforce multi-party agreement in the
  network itself (Constitution IV). 2.5 is the current long-term-support line.
- **Alternatives considered**:
  - Hyperledger Besu / Quorum (permissioned EVM): workable, but account-level permissioning
    and Solidity's lack of decimals add risk for limit comparisons; org membership is weaker.
  - Corda: point-to-point sharing conflicts with "every lab holds a full copy" for
    tamper detection.
  - Public chain: rejected by spec assumption and personal-data rules.

## R2. Smart contract (chaincode) language

- **Decision**: Go, using `fabric-contract-api-go` v2.
- **Rationale**: Most mature Fabric chaincode runtime; static typing; deterministic when
  map iteration and floats are avoided (enforced by lint rule and tests). Same language as
  the gateway, so one toolchain.
- **Alternatives considered**: Node.js chaincode (dynamic typing, float numbers by default);
  Java (heavier runtime, no benefit here).

## R3. Deterministic numeric comparison

- **Decision**: All measured values and limits are carried as decimal strings and compared
  with `github.com/shopspring/decimal`. Floats are banned in chaincode. Unit conversion uses
  an explicit factor table stored inside each rule set (e.g., `A -> mA = 1000`).
- **Rationale**: Binary floats can give different rounding across nodes and misjudge exact
  boundary values (spec edge case "result exactly on a limit"). Factors inside the versioned
  rule set make conversion reproducible (Constitution II).
- **Alternatives considered**: Scaled integers (error-prone per unit); float64 (rejected).

## R4. Where assessment runs

- **Decision**: Assessment runs inside chaincode (`AssessConsignment`) using the rule-set
  version selected by the consignment's arrival date. The verdict, inputs hash, and
  rule-set version are written in the same transaction.
- **Rationale**: Every endorsing peer (organization and lab) computes the verdict
  independently; a dishonest peer's divergent result fails endorsement. Re-assessment by an
  auditor is a read-only evaluation of the same function (FR-011, FR-023).
- **Alternatives considered**: Off-chain rule engine with result anchored on chain —
  rejected, the verdict would not be independently computed by other node operators.

## R5. Endorsement policy

- **Decision**: Chaincode-level policy
  `AND('StdOrgMSP.peer', OutOf(1, 'Lab1MSP.peer', 'Lab2MSP.peer', ...))`, regenerated when
  a lab joins. Rule-set approval additionally requires a second technical officer's
  signature checked in chaincode.
- **Rationale**: The organization cannot write alone; any one lab offline does not block
  work (Clarification Q4).
- **Alternatives considered**: `MAJORITY Endorsement` — blocks writes if several labs are
  offline at once; organization-only endorsement defeats the purpose of lab nodes.

## R6. Identity, signing, and authentication

- **Decision**:
  - Each organization (StdOrg, each lab) runs its own Fabric CA and issues one X.509
    certificate per person, with attributes `role` and `participantId`.
  - Chaincode authorizes every function via `cid` attribute checks (Constitution IV:
    enforcement in contract logic).
  - People log in to their own organization's gateway with OIDC (Keycloak 26), with
    TOTP second factor for all roles except importers. The gateway holds each user's
    private key encrypted in its database (wrapping key from an HSM or, in dev, a file),
    unlocked only for that authenticated user's requests.
  - Importers have no Fabric identity; the StdOrg gateway submits their appeals with a
    service identity plus the importer's account ID recorded in the transaction.
- **Rationale**: Per-person signatures on every entry (FR-003) without asking staff to
  manage keys in the browser. Labs hold their own users' keys, so the organization cannot
  sign as a lab.
- **Alternatives considered**: In-browser key storage with WebCrypto (key loss and backup
  problems for non-technical staff); one service identity per organization (loses
  per-person signatures, rejected by FR-003).
- **Ceiling**: Custodial keys mean a compromised gateway can sign as its users. Upgrade
  path: HSM-resident user keys or smart-card signing.

## R7. Off-chain storage and personal data

- **Decision**: PostgreSQL 16 at the StdOrg site holds: importer personal/contact data,
  user key material, evidence documents (`bytea`, max 25 MB per file), and the read model
  for reports. Each lab gateway has its own PostgreSQL for its users' key material and the
  results files it uploads. The ledger stores only IDs and SHA-256 hashes (Constitution III).
- **Rationale**: One storage product covers all needs at pilot volume (500 consignments
  per day ≈ a few GB per month); streaming replication covers availability.
- **Alternatives considered**: S3-compatible object store (MinIO) — extra service not yet
  needed. **Ceiling**: move documents to an object store if storage passes ~500 GB.
- **Legal deletion**: deleting a document removes the bytes and submits a
  `RecordDocumentRemoval` transaction; verification then reports "removed", not "tampered".

## R8. Reporting and search (FR-024)

- **Decision**: The gateway listens to block events and projects them into PostgreSQL
  read tables (consignments, verdicts, decisions, dispositions). Reports query those tables.
  Audit verification always reads the ledger directly, never the projection.
- **Rationale**: Ledger queries by period/category are slow and awkward; the projection is
  rebuildable from the ledger at any time, so it carries no trust.
- **Alternatives considered**: CouchDB rich queries on world state (no history, no joins).

## R9. Web application, RTL, and Jalali calendar

- **Decision**: React 18 + TypeScript + Vite single-page app, served by the gateway.
  Translations via `react-i18next` with `fa` (default) and `en` bundles. Direction set by
  `<html dir>` and CSS logical properties. Dates rendered with the native
  `Intl.DateTimeFormat('fa-IR-u-ca-persian')` or `'en-GB'`; no calendar library.
- **Rationale**: Native `Intl` supports the Persian calendar in all current browsers;
  logical CSS properties handle RTL without a separate stylesheet (FR-025, FR-026).
- **Alternatives considered**: Server-rendered pages (harder interactive forms for results
  entry); date libraries such as moment-jalaali (unneeded).

## R10. Results file template (Clarification Q2)

- **Decision**: UTF-8 CSV with a fixed header, one row per sample-test result, validated in
  full before any submission; any invalid row rejects the whole file with per-row errors.
  Format in [contracts/results-file-template.md](contracts/results-file-template.md).
- **Rationale**: Every LIMS and spreadsheet can export CSV; strict all-or-nothing matches
  the spec.
- **Alternatives considered**: XLSX (parsing ambiguity, locale number formats); JSON
  (unfriendly for lab staff).

## R11. Notifications

- **Decision**: Email through the organization's SMTP relay plus an in-app notification
  list in the importer portal.
- **Rationale**: Spec assumes existing channels; SMS gateway can be added later.

## R12. Testing strategy

- **Decision**:
  - Chaincode: Go unit tests with a mocked stub (`shimtest`-style fakes) written first for
    every rule and access check (Constitution V); boundary, unit-mismatch, missing, and
    conflicting-result cases are table-driven.
  - Static analysis: `go vet`, `staticcheck`, and a custom check forbidding `float32/64`,
    `time.Now`, `math/rand`, and map range in chaincode.
  - Integration: gateway tests against a local 3-org Fabric network (StdOrg + 2 labs) in
    Docker Compose, including stopping one lab peer.
  - Web: Vitest for components; one Playwright smoke flow per user story.
- **Rationale**: Matches the constitution's CI gate (multi-node local network).

## R13. Observability

- **Decision**: Structured JSON logs from gateway and chaincode with `txId` and
  `consignmentId` fields; Prometheus metrics from Fabric peers/orderers and the gateway
  (`/metrics`); availability measured by a synthetic login + read probe every minute during
  working hours (SC-009).
- **Rationale**: Constitution VI requires logs correlated to transaction IDs; Fabric
  exposes Prometheus metrics natively.

## R14. Deployment topology

- **Decision**:
  - StdOrg: 2 peers, 3 Raft orderers, 1 gateway (2 replicas behind a load balancer),
    PostgreSQL primary + standby, Keycloak, Fabric CA.
  - Each lab: 1 peer, 1 gateway, PostgreSQL, Keycloak realm (or StdOrg-hosted realm for
    small labs), Fabric CA.
  - Containers on Linux hosts; Docker Compose for dev and pilot; Kubernetes left for later.
- **Rationale**: Survives loss of any single node (SC-010). **Ceiling**: all orderers at
  StdOrg means the organization controls ordering; labs still detect tampering because
  blocks are hash-chained and they hold full copies. Upgrade path: give labs orderer seats.
