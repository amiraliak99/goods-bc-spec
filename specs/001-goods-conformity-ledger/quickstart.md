# Quickstart: Validate the Conformity Ledger End to End

Proves each user story on a local network. Endpoint details:
[contracts/gateway-api.md](contracts/gateway-api.md). Entities:
[data-model.md](data-model.md).

## Prerequisites

- Linux or macOS, Docker 24+ with Compose v2, Go 1.23, Node 20, `jq`, `curl`.
- 8 GB free RAM (3 orgs: StdOrg + Lab1 + Lab2).

## Setup

```bash
make network-up        # Fabric CAs, 3 orderers, StdOrg x2 peers, Lab1/Lab2 peers, channel
make chaincode-deploy  # package, approve by all orgs, commit `conformity`
make stack-up          # PostgreSQL, Keycloak (seed users), gateways (StdOrg :8080, Lab1 :8081)
make seed              # pilot rule set "IEC 60335-1 ed.6 v1", one importer, test users
```

Seed users (password `Passw0rd!`, TOTP disabled in dev): `inspector1`, `assessor1`,
`assessor2`, `techofficer1`, `techofficer2`, `auditor1`, `admin1` on StdOrg; `lab1user`
on Lab1; `importer1` in the importer portal. `scripts/token.sh <user>` prints a bearer
token.

## Scenario 1 — Immutable registration (Story 1)

1. As `inspector1`, `POST /consignments` with an invoice → expect `201` with
   `consignmentId` and `txId`.
2. `POST /consignments/{id}/corrections` changing quantity with a reason → `GET .../history`
   shows both the original and the correction entry.
3. Overwrite the invoice bytes directly in PostgreSQL, then
   `GET .../documents/{docId}/verify` → `status: "mismatch"`.
4. `POST /consignments` without `standardRef` → `422` naming `standardRef`.

## Scenario 2 — Automated assessment (Story 2)

1. `inspector1` records a sample assigned to Lab1.
2. As `lab1user` on `:8081`, upload `testdata/results-pass.csv` → verdict `Conforming`.
3. Repeat on a new consignment with `testdata/results-boundary.csv` (value equals an
   inclusive limit) → `Conforming`; with `results-fail.csv` → `NonConforming` showing value
   and limit; with `results-missing.csv` → `RequiresReview` listing missing tests.
4. Upload `testdata/results-bad-row.csv` → `422` with per-row errors and nothing recorded.
5. `POST /consignments/{id}/reassess` as `auditor1` → identical verdict and rule-set version.

## Scenario 3 — Decisions, overrides, appeals, dispositions (Story 3)

1. `assessor1` confirms the Conforming verdict with a `certificateRef` → `importer1` sees the
   decision in `/me/notifications`.
2. `assessor1` overrides a NonConforming verdict without justification → `422`; with
   justification → `201`; history shows both verdict and override.
3. `importer1` files an appeal; `assessor1` (original decider) tries to resolve → `403`;
   `assessor2` resolves → new linked decision.
4. On a NonConforming consignment, choose `rework`, complete with evidence → status
   `Sampling`; new results produce a new verdict linked to the same consignment.

## Scenario 4 — Independent audit (Story 4)

1. As `auditor1`, `GET /certificates/{ref}` then `/history` → chronological entries with
   actor and timestamp.
2. Any `POST` as `auditor1` → `403`.
3. Confirm the same history from a Lab1 peer directly:
   `scripts/peer-query.sh lab1 GetHistory <consignmentId>` → identical entries.

## Scenario 5 — Rule-set versions (Story 5)

1. `techofficer1` drafts v2 with a tighter limit and `effectiveFrom` tomorrow;
   `techofficer1` approves → `403 ERR_SOD`; `techofficer2` approves → `200`.
2. Assess a consignment with arrival date today → uses v1. Old verdicts still cite v1.

## Scenario 6 — Resilience (SC-010)

1. `docker stop peer0.lab1` → Lab1 submits results via StdOrg peer; writes still succeed
   (endorsed by StdOrg + Lab2).
2. `docker start peer0.lab1` → after catch-up, `scripts/peer-query.sh lab1 GetHistory`
   matches StdOrg.

## Automated checks

```bash
make test-chaincode     # unit tests + determinism static check
make test-integration   # gateway against local network, includes Scenario 6
make test-web           # Vitest + Playwright smoke per story
```

All must pass with zero failures.
