# Chaincode Contract: `conformity`

Channel: `conformity`. All functions read the caller's `participantId` and `role` from the
certificate attributes (`cid`), refuse revoked participants, and refuse calls whose role is
not listed. Errors use stable codes: `ERR_FORBIDDEN`, `ERR_NOT_FOUND`, `ERR_INVALID`,
`ERR_STATE` (illegal transition), `ERR_SOD` (segregation of duties), `ERR_HASH_MISMATCH`.

Inputs and outputs are JSON matching [data-model.md](../data-model.md).

## Write functions

| Function | Allowed role(s) | Input | Effect / key checks |
|----------|----------------|-------|---------------------|
| `RegisterParticipant` | admin | Participant | Create; admin of own MSP only |
| `RevokeParticipant` | admin | participantId, reason | status → revoked |
| `RegisterConsignment` | inspector | Consignment (no status) + DocumentRefs | status → Registered; all required fields present |
| `AttachDocument` | inspector, lab | consignmentId, DocumentRef | Append only |
| `RecordDocumentRemoval` | admin | consignmentId, documentId, legalBasis | `removed=true` |
| `CorrectConsignment` | inspector | consignmentId, field, newValue, reason | Correction event; identity fields (`consignmentId`, `registeredBy`) not correctable |
| `SplitConsignment` | inspector | consignmentId, lots[] | Child consignments with `parentId`; parent → Split |
| `RecordSample` | inspector | Sample | status Registered → Sampling |
| `SubmitResults` | lab | batchId, TestResult[] | All-or-nothing; caller MSP = sample's assignedLabMsp; status → ResultsReceived |
| `AssessConsignment` | assessor, service | consignmentId | Selects rule set by arrivalDate, writes Verdict; status → Assessed |
| `RecordManualVerdict` | assessor | consignmentId, perTest[], justification | Verdict with `manual=true` |
| `DecideConsignment` | assessor | Decision + `documentChecks[] {documentId, sha256}` | `ERR_SOD` if caller submitted any result for it; override needs justification; any `documentChecks` hash differing from the on-chain one → `ERR_HASH_MISMATCH`; missing check for a non-removed document → `ERR_INVALID` |
| `FileAppeal` | service (on behalf of the importer of record) | Appeal | Only importer of record; status → UnderAppeal |
| `ResolveAppeal` | assessor (≠ original decider) | appealId, Decision(kind=appealOutcome) | Appeal resolved |
| `ChooseDisposition` | inspector, assessor | Disposition (type) | Requires final NonConforming; status → DispositionPending |
| `CompleteDisposition` | inspector | dispositionId, evidenceDocumentIds | Closed, or Sampling for rework |
| `DraftRuleSet` | techOfficer | RuleSet (status draft) | New version number |
| `ApproveRuleSet` | techOfficer | ruleSetId, version | `ERR_SOD` if caller drafted it; immutable afterwards |

## Read functions

| Function | Allowed role(s) | Output |
|----------|----------------|--------|
| `GetConsignment` | all except importer (importer via gateway filter) | Consignment |
| `GetHistory` | inspector, assessor, auditor, admin | Ordered entries: txId, timestamp, function, actor, payload |
| `GetRuleSet` | all | RuleSet (specific version or in force on a date) |
| `ReassessDryRun` | auditor, assessor | Verdict recomputed from recorded inputs and recorded rule-set version; no write |
| `VerifyDocument` | auditor, assessor, inspector | `{documentId, status: ok|mismatch|removed}` given a supplied sha256 |

## Determinism rules (enforced by tests and static check)

- No `float32/float64`, `time.Now`, `math/rand`, network or file access.
- Never range over Go maps when building output; sort keys first.
- Timestamps come only from `stub.GetTxTimestamp()`.

## Endorsement

`AND('StdOrgMSP.peer', OutOf(1, <every lab MSP>.peer))`. Updated through the chaincode
lifecycle when a lab joins or leaves.
