# Implementation Plan: Incoming Goods Conformity Ledger

**Branch**: `001-goods-conformity-ledger` | **Date**: 2026-10-23 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-goods-conformity-ledger/spec.md`

## Summary

A permissioned ledger shared by the standards organization (StdOrg) and each accredited
lab records every step of an incoming consignment's life as signed, append-only
transactions. Chaincode holds versioned conformity rule sets and computes verdicts
deterministically on every endorsing peer, so neither the organization nor a lab can
fake an outcome alone. A Go gateway per organization gives staff, labs, importers, and
auditors a REST API and a Persian/English web app, keeps personal data and documents
off-chain (hash-anchored), and projects ledger events into a read model for reports.
Details: [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.23 (chaincode + gateway); TypeScript 5 (web)

**Primary Dependencies**: Hyperledger Fabric 2.5 LTS (peers, Raft orderers, Fabric CA),
`fabric-contract-api-go` v2, `fabric-gateway` Go SDK, `shopspring/decimal`, Keycloak 26
(OIDC), React 18 + Vite + `react-i18next`

**Storage**: Fabric ledger (on-chain, LevelDB state); PostgreSQL 16 off-chain (personal
data, documents ≤ 25 MB, encrypted user keys, read model)

**Testing**: `go test` (chaincode unit with fake stub, gateway), integration tests against
a local 3-org Fabric network in Docker Compose, Vitest, Playwright

**Target Platform**: Linux servers in containers (Docker Compose for dev/pilot); evergreen
desktop browsers

**Project Type**: Web service + chaincode + web SPA

**Performance Goals**: Verdict < 1 min after complete results for 95% of consignments
(SC-002); 500 consignments/day, 50 concurrent users (SC-008); registration form usable in
< 5 min (SC-007)

**Constraints**: 99.5% availability in working hours (SC-009); survive loss of any single
node without data loss (SC-010); deterministic chaincode (no floats/clock/randomness);
no personal data on chain; Persian RTL default + English; Jalali/Gregorian display, UTC
storage

**Scale/Scope**: Pilot with 1 StdOrg + 2–10 labs; ~180k consignments/year; ~10 rule sets
initially; ~25 screens

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | How the design complies | Status |
|-----------|------------------------|--------|
| I. Immutable Record of Truth | All lifecycle events are Fabric transactions; corrections are new transactions (`CorrectConsignment`); state is derivable from key history | Pass |
| II. Deterministic, Versioned Rules | Rule sets versioned on chain, cite standard + clause; verdict stores rule-set version + inputs hash; decimal math; determinism static check; overrides require justification | Pass |
| III. Minimal On-Chain Data | Only IDs + SHA-256 on chain; documents and importer data in PostgreSQL; `RecordDocumentRemoval` for legal deletion | Pass |
| IV. Permissioned Identity, Least Privilege | Fabric MSP per org, per-person certs with role attributes; access checked in chaincode; SoD for results/decision and draft/approve; revocation blocks new actions only. Importers are verified by the organization's Keycloak, which acts as its trust authority for importers; their appeals are signed by the StdOrg `service` identity and record the importer's account ID and a hash of their login token claims (importers act on the ledger only to appeal; certificates for thousands of importers add support load) | Pass |
| V. Smart Contract Safety | Test-first chaincode; table-driven boundary/malicious tests; static analysis; upgrades via Fabric lifecycle requiring approval by every org | Pass |
| VI. Traceability & Auditability | One consignmentId links everything; `GetHistory`, `VerifyDocument`, `ReassessDryRun` for auditors; logs carry `txId` | Pass |
| VII. Simplicity & Proven Components | Mature platforms only; one language for chaincode + gateway; one database; no object store, no message queue, no Kubernetes yet. Dependencies justified below | Pass |
| Domain constraints | Rule tests cite clause; manual-verdict path; UTC timestamps | Pass |
| Workflow gates | CI runs chaincode unit, integration on multi-node network, web tests | Pass |

**Dependency justification (Principle VII)**:

| Dependency | Needed for | Why not something simpler |
|-----------|-----------|---------------------------|
| Hyperledger Fabric | Shared tamper-evident ledger, org membership | Required by spec; only mature permissioned option with org-level endorsement |
| PostgreSQL | Personal data, documents, read model | One store covers all off-chain needs |
| Keycloak | OIDC login, TOTP, importer accounts | Building auth is a security risk; Fabric CA does not do user login |
| `shopspring/decimal` | Exact limit comparison | Stdlib has no decimal type; `big.Rat` lacks parsing/rounding conveniences needed |
| React + i18next | Interactive forms, RTL/LTR switching | Plain server templates make results entry and history views harder |

**Post-design re-check (after Phase 1)**: Data model, contracts, and quickstart keep all
items above at Pass. No violations; Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-goods-conformity-ledger/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   ├── chaincode.md
│   ├── gateway-api.md
│   └── results-file-template.md
└── tasks.md             # Phase 2 output (/speckit-tasks)
```

### Source Code (repository root)

```text
chaincode/conformity/          # Go chaincode
├── contract.go                # Function entry points + access checks
├── consignment.go             # Registration, corrections, split, samples, state machine
├── results.go                 # SubmitResults
├── assess.go                  # Deterministic rule evaluation
├── decision.go                # Decisions, appeals, dispositions
├── ruleset.go                 # Draft / approve / lookup by date
├── participant.go
└── *_test.go                  # Unit tests next to code

gateway/                       # Go service, one deployment per org
├── cmd/gateway/main.go
├── internal/api/              # REST handlers, OIDC middleware, error mapping
├── internal/fabric/           # Gateway SDK client, per-user signer
├── internal/store/            # PostgreSQL: importers, documents, keys, read model
├── internal/projector/        # Block event listener → read model
├── internal/resultsfile/      # CSV validation
├── internal/notify/           # SMTP + in-app notifications
└── migrations/

web/                           # React + TypeScript SPA
├── src/pages/                 # Per role: inspector, lab, assessor, techOfficer, auditor, importer, admin
├── src/components/
├── src/i18n/fa.json, en.json
└── tests/

network/                       # Local and pilot Fabric network
├── compose/                   # docker-compose files for 3 orgs, PostgreSQL, Keycloak
├── config/                    # configtx.yaml, crypto config, Keycloak realm exports
└── scripts/                   # network-up, deploy, peer-query, token

tests/integration/             # Gateway against running network (Go)
testdata/                      # Rule sets, CSV fixtures used by quickstart
Makefile
```

**Structure Decision**: Three deployables (chaincode, gateway, web) because Fabric
chaincode must be packaged separately from the client service, and the SPA ships as static
files served by the gateway. Unit tests live beside Go code; cross-component tests live in
`tests/integration/`.

## Complexity Tracking

No constitution violations to justify.
