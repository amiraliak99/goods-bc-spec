# Gateway REST API (v1)

Base path `/api/v1`. JSON over HTTPS. Auth: OIDC bearer token from the organization's
Keycloak realm. `Accept-Language: fa|en` selects message language. Errors:
`{ "code": "ERR_...", "message": "...", "details": [...] }` with chaincode codes mapped to
HTTP: `ERR_INVALID` 422, `ERR_FORBIDDEN`/`ERR_SOD` 403, `ERR_NOT_FOUND` 404,
`ERR_STATE` 409, `ERR_HASH_MISMATCH` 409. Every write response includes `txId`.

The same API is served by the StdOrg gateway and each lab gateway; each gateway exposes
only the endpoints its users' roles can use.

## Consignments

| Method | Path | Role | Maps to |
|--------|------|------|---------|
| POST | `/consignments` | inspector | `RegisterConsignment` |
| GET | `/consignments?status=&category=&standard=&importer=&from=&to=` | staff, auditor | read model |
| GET | `/consignments/{id}` | staff, auditor, importer (own) | `GetConsignment` |
| GET | `/consignments/{id}/history` | staff, auditor | `GetHistory` |
| POST | `/consignments/{id}/corrections` | inspector | `CorrectConsignment` |
| POST | `/consignments/{id}/split` | inspector | `SplitConsignment` |
| POST | `/consignments/{id}/documents` (multipart) | inspector, lab | store + `AttachDocument` |
| GET | `/consignments/{id}/documents/{docId}` | staff, auditor | file download |
| GET | `/consignments/{id}/documents/{docId}/verify` | auditor, staff | recompute hash + `VerifyDocument` |
| DELETE | `/consignments/{id}/documents/{docId}` | admin | null bytes + `RecordDocumentRemoval` |
| POST | `/consignments/{id}/samples` | inspector | `RecordSample` |

## Results and assessment

| Method | Path | Role | Maps to |
|--------|------|------|---------|
| POST | `/results` (JSON array) | lab | `SubmitResults` |
| POST | `/results/upload` (CSV, see results-file-template.md) | lab | validate, then `SubmitResults` |
| POST | `/consignments/{id}/assess` | assessor | `AssessConsignment` (also triggered automatically after results) |
| POST | `/consignments/{id}/manual-verdict` | assessor | `RecordManualVerdict` |
| POST | `/consignments/{id}/reassess` | auditor, assessor | `ReassessDryRun` |

## Decisions, appeals, dispositions

| Method | Path | Role | Maps to |
|--------|------|------|---------|
| POST | `/consignments/{id}/decisions` | assessor | `DecideConsignment` (sends importer notification) |
| POST | `/consignments/{id}/appeals` | importer | `FileAppeal` |
| POST | `/appeals/{id}/resolve` | assessor | `ResolveAppeal` |
| POST | `/consignments/{id}/dispositions` | inspector, assessor | `ChooseDisposition` |
| POST | `/dispositions/{id}/complete` | inspector | `CompleteDisposition` |

## Rule sets

| Method | Path | Role | Maps to |
|--------|------|------|---------|
| GET | `/rule-sets`, `/rule-sets/{id}/versions/{v}` | all staff, auditor | `GetRuleSet` |
| POST | `/rule-sets` | techOfficer | `DraftRuleSet` |
| POST | `/rule-sets/{id}/versions/{v}/approve` | techOfficer | `ApproveRuleSet` |

## Participants, reports, notifications

| Method | Path | Role | Maps to |
|--------|------|------|---------|
| POST | `/participants`, `/participants/{id}/revoke` | admin | `RegisterParticipant`, `RevokeParticipant` |
| GET | `/reports/summary?groupBy=status|category|standard|importer&from=&to=` | staff, auditor | read model, includes override counts |
| GET | `/certificates/{ref}` | auditor, staff | read model → consignment id |
| GET | `/me/notifications` | any | notification table |
| GET | `/healthz`, `/metrics` | none / internal | probes, Prometheus |
