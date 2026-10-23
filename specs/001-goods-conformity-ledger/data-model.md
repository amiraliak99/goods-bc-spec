# Data Model: Incoming Goods Conformity Ledger

Two layers:

- **On-chain** (Fabric world state + immutable history): IDs, hashes, verdicts, decisions.
  No personal data (Constitution III, FR-006).
- **Off-chain** (PostgreSQL): personal data, document bytes, user key material, read model
  for reports. Rebuildable read model; source of truth is the ledger.

Conventions: IDs are ULIDs as strings. Timestamps are the transaction timestamp
(`stub.GetTxTimestamp`, UTC, RFC 3339) — never wall-clock inside chaincode. Decimal values
are strings (e.g., `"0.75"`). Every on-chain write also emits an event named after the
function.

## On-chain entities

World-state keys use composite keys `<type>~<id>`. History of every key is available from
the ledger (`GetHistoryForKey`), which gives the full append-only trail.

### Participant

| Field | Type | Rules |
|-------|------|-------|
| participantId | string | Unique; matches cert attribute `participantId` |
| orgMsp | string | `StdOrgMSP` or a lab MSP |
| roles | []enum | `inspector`, `lab`, `assessor`, `techOfficer`, `importer`, `auditor`, `admin`, `service` (StdOrg gateway identity for auto-assessment and importer appeals) |
| status | enum | `active` / `revoked` |
| revokedAt | timestamp? | Set on revocation; actions after it are refused (FR-020) |

Importers are participants with no certificate; `participantId` links to the off-chain
importer record.

### Consignment

| Field | Type | Rules |
|-------|------|-------|
| consignmentId | string | Unique (FR-002) |
| parentId | string? | Set when this is a split lot of another consignment |
| importerId | string | Participant with role `importer` |
| productCategory | string | Code from the organization's category list |
| productDescription | string | Max 500 chars; no personal data |
| standardRef | object | `{number, edition}`; required |
| quantity | decimal string + unit | > 0 |
| origin | string | ISO 3166-1 alpha-2 |
| arrivalDate | date | ≤ registration time |
| registeredBy | participantId | Role `inspector` |
| documents | []DocumentRef | See below |
| status | enum | See state machine |
| currentVerdictId | string? | Latest verdict |
| currentDecisionId | string? | Latest decision |

### DocumentRef (embedded, on-chain)

| Field | Type | Rules |
|-------|------|-------|
| documentId | string | Matches off-chain record |
| kind | enum | `invoice`, `packingList`, `photo`, `testReport`, `dispositionEvidence`, `other` |
| sha256 | hex string | 64 chars (FR-005) |
| removed | bool | True after legal deletion; never set back to false |

### Correction (event-only)

Corrections do not overwrite history: the corrected field is updated in world state, and
the transaction stores `{targetId, field, oldValue, newValue, reason, by}`; both values stay
in key history (FR-004). `reason` is required, max 500 chars.

### Sample

| Field | Type | Rules |
|-------|------|-------|
| sampleId | string | Unique |
| consignmentId | string | Existing consignment |
| takenBy | participantId | Role `inspector` |
| takenAt | date | ≥ arrivalDate |
| assignedLabMsp | string | Lab that will test it |

### TestResult

| Field | Type | Rules |
|-------|------|-------|
| resultId | string | Unique |
| sampleId | string | Existing sample |
| testCode | string | Must exist in applicable rule set |
| value | decimal string | Required for numeric tests |
| passFail | enum? | For qualitative tests (e.g., labeling check): `pass`/`fail` |
| unit | string | Must be the rule unit or convertible by the rule set's factor table |
| method | string | Test method reference |
| reportDocumentId | string | DocumentRef of kind `testReport` |
| submittedBy | participantId | Role `lab`, from the sample's assigned lab |
| batchId | string | Groups results uploaded together from one file |

### RuleSet

| Field | Type | Rules |
|-------|------|-------|
| ruleSetId | string | Stable per standard edition |
| version | int | Starts at 1, +1 per published version |
| standardRef | object | `{number, edition}` |
| productCategories | []string | Categories this rule set applies to |
| tests | []RuleTest | At least one |
| unitFactors | map unit→{baseUnit, factor} | Decimal factors; stored as sorted list for determinism |
| effectiveFrom | date | Must be ≥ approval date |
| draftedBy | participantId | Role `techOfficer` |
| approvedBy | participantId? | Role `techOfficer`, ≠ draftedBy (FR-019) |
| status | enum | `draft` → `approved` (immutable after approval) |

**RuleTest**: `testCode`, `name`, `clause` (standard clause, required), `kind`
(`numeric` / `qualitative`), `unit`, `min?`, `max?`, `minInclusive`, `maxInclusive`,
`required` (bool), `samplesRequired` (int ≥ 1).

### Verdict

| Field | Type | Rules |
|-------|------|-------|
| verdictId | string | Unique |
| consignmentId | string | |
| ruleSetId + version | string + int | Version in force on arrivalDate (FR-009/010) |
| inputsHash | sha256 | Hash of sorted input result IDs and values |
| perTest | []{testCode, outcome, measured, limit, reason} | outcome: `pass`/`fail`/`missing`/`conflict`/`unitError` |
| overall | enum | `Conforming` / `NonConforming` / `RequiresReview` |
| manual | bool | True for the manual path (FR-017) |

Overall rule: any `fail` → `NonConforming`; else any `missing`/`conflict`/`unitError` or
no active rule set → `RequiresReview`; else `Conforming`. A `fail` only counts if no other
result for the same test and sample conflicts with it; conflicting results → `conflict`.

### Decision

| Field | Type | Rules |
|-------|------|-------|
| decisionId | string | |
| consignmentId | string | |
| verdictId | string | Verdict being confirmed/overridden |
| kind | enum | `confirm` / `override` / `appealOutcome` / `manual` |
| outcome | enum | `Conforming` / `NonConforming` |
| justification | string | Required for `override`, `appealOutcome`, `manual`; max 2000 chars |
| certificateRef | string? | Required when outcome `Conforming` |
| rejectionReasons | []string? | Required when outcome `NonConforming` |
| decidedBy | participantId | Role `assessor`; MUST NOT be the MSP/person who submitted the results (FR-019) |
| previousDecisionId | string? | Set for appeal outcomes |

### Appeal

`appealId`, `consignmentId`, `decisionId`, `filedBy` (importer participant), `grounds`
(max 2000 chars), `status` (`open` / `resolved`), `resolutionDecisionId?`.

### Disposition (Clarification Q5)

| Field | Type | Rules |
|-------|------|-------|
| dispositionId | string | |
| consignmentId | string | Final decision NonConforming |
| type | enum | `reExport` / `destruction` / `rework` |
| chosenBy | participantId | Role `inspector` or `assessor` |
| completedBy | participantId? | Role `inspector` |
| completedAt | date? | |
| evidenceDocumentIds | []string | ≥ 1 on completion |
| reassessmentConsignmentLink | string? | For rework: the new verdict cycle uses the same consignmentId |

## Consignment state machine

```text
Registered ──sample──▶ Sampling ──results──▶ ResultsReceived ──assess──▶ Assessed
Assessed ──confirm/override──▶ Conforming (final)
Assessed ──confirm/override──▶ NonConforming ──choose──▶ DispositionPending
DispositionPending ──complete(reExport|destruction)──▶ Closed
DispositionPending ──complete(rework)──▶ Sampling   (new samples, new verdict)
Conforming | NonConforming ──appeal──▶ UnderAppeal ──appealOutcome──▶ Conforming | NonConforming
Any non-final ──split──▶ Split (children get their own lifecycle)
```

Transitions not listed are refused by chaincode. `manual` verdicts enter at `Assessed`.

## Off-chain entities (PostgreSQL)

| Table | Key fields | Notes |
|-------|-----------|-------|
| importer | id, legal_name, national_id, contact_email, phone, locale | Personal data (never on chain) |
| app_user | id, participant_id, org, oidc_subject, locale, calendar (`jalali`/`gregorian`) | |
| user_key | user_id, encrypted_private_key, cert_pem | Wrapped with HSM/file key |
| document | id, consignment_id, kind, filename, mime, size (≤ 25 MB), sha256, bytes, removed_at | Bytes nulled on legal deletion |
| results_upload | id, lab_msp, filename, sha256, status, errors_json | Per-row errors on reject |
| notification | id, recipient_user_id, kind, payload_json, sent_at, read_at | |
| rm_consignment, rm_verdict, rm_decision, rm_disposition | mirror of chain fields + block number | Read model, rebuilt from ledger |
| projection_checkpoint | channel, last_block | Resume point for event listener |
