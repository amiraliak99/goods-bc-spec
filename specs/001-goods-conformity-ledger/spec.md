# Feature Specification: Incoming Goods Conformity Ledger

**Feature Branch**: `001-goods-conformity-ledger`

**Created**: 2026-10-23

**Status**: Draft

**Input**: User description: "A Blockchain and Smart Contract-Based System for Immutable Recording and Automated Conformity Assessment of Incoming Goods in a Standards Organization"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Register an Incoming Consignment as a Tamper-Proof Record (Priority: P1)

An inspector at the standards organization receives a consignment of goods (e.g., imported
electrical appliances). The inspector registers it: importer, product category, applicable
standard, quantity, declared origin, and supporting documents (invoice, packing list, photos).
The system creates a permanent, signed record with a unique consignment identifier. From then
on, every action on the consignment is appended to its history and no one can alter or erase
earlier entries.

**Why this priority**: Without a trustworthy record of what arrived, when, and with which
documents, nothing downstream (testing, decisions, disputes) can be trusted. This alone
replaces paper or editable spreadsheet logs and delivers value immediately.

**Independent Test**: Register a consignment with attached documents, then attempt to modify
the record and a document. Verify the original stays intact, the change attempt is refused or
recorded as a new correction entry, and a modified document is flagged as not matching.

**Acceptance Scenarios**:

1. **Given** an authenticated inspector, **When** they submit a complete registration,
   **Then** the system issues a unique consignment identifier and a timestamped, signed record.
2. **Given** a registered consignment, **When** an inspector corrects a field (e.g., quantity),
   **Then** the original entry remains visible and a correction entry with reason and author
   is appended.
3. **Given** a registered consignment with an attached invoice, **When** the stored invoice
   file is altered outside the system, **Then** verification reports a mismatch for that
   document.
4. **Given** a registration missing a mandatory field (e.g., applicable standard),
   **When** the inspector submits, **Then** the system rejects it and names the missing field.

---

### User Story 2 - Automated Conformity Assessment from Test Results (Priority: P2)

After sampling, an accredited laboratory submits measured test results for the consignment's
samples (e.g., insulation resistance, leakage current, labeling check). The system
automatically compares each result against the published criteria for the applicable standard
and edition, and produces a verdict per test and an overall verdict: Conforming,
Non-Conforming, or Requires Review. The verdict records which rule-set version was used.

**Why this priority**: Automating the comparison removes manual calculation errors and
inconsistent judgments, and makes each decision reproducible. It depends on Story 1 records.

**Independent Test**: Load a rule set for one standard, submit results that are inside,
exactly at, and outside the limits, and check the verdicts match the expected outcomes and
cite the rule-set version.

**Acceptance Scenarios**:

1. **Given** a consignment with all required test results within limits, **When** the
   assessment runs, **Then** the overall verdict is Conforming with per-test outcomes shown.
2. **Given** one result outside its limit, **When** the assessment runs, **Then** the
   overall verdict is Non-Conforming and the failing test, measured value, and limit are shown.
3. **Given** a required test result is missing, **When** the assessment runs, **Then** the
   verdict is Requires Review and lists the missing tests.
4. **Given** the same inputs and rule-set version, **When** the assessment is re-run,
   **Then** the verdict is identical.
5. **Given** the laboratory that submitted results, **When** that same party attempts to
   approve the conformity decision, **Then** the system refuses.

---

### User Story 3 - Assessor Decision, Override, and Certificate Record (Priority: P3)

A conformity assessor reviews the automated verdict. They confirm it, or override it with a
mandatory written justification (e.g., a test was performed under a deviation approved by the
technical committee). On a final Conforming decision the system records a conformity
certificate reference; on Non-Conforming it records a rejection with reasons. The importer is
notified of the outcome.

**Why this priority**: The organization remains legally accountable for decisions; a human
sign-off and a controlled override path are needed before go-live, but they build on Stories
1 and 2.

**Independent Test**: Take an automated verdict, confirm one and override another with
justification; verify both decisions, their authors, and the certificate/rejection records
appear in the consignment history and the importer sees the outcome.

**Acceptance Scenarios**:

1. **Given** a Conforming automated verdict, **When** the assessor confirms it, **Then** a
   final decision and certificate reference are recorded and the importer is notified.
2. **Given** a Non-Conforming verdict, **When** the assessor overrides it without
   justification, **Then** the system refuses the override.
3. **Given** an override with justification, **When** it is saved, **Then** both the
   automated verdict and the override remain visible, with the assessor's identity.
4. **Given** a final decision, **When** the importer appeals, **Then** the appeal is
   recorded against the consignment and its outcome is appended as a new decision.

---

### User Story 4 - Independent Audit and Verification (Priority: P4)

An auditor (internal quality unit or accreditation body) selects a consignment or a
certificate number and verifies, without help from operators, the full history: who did
what and when, which documents were used and that they are unaltered, which rule-set version
produced the verdict, and whether any overrides occurred. The auditor can also re-run the
assessment to confirm the verdict reproduces.

**Why this priority**: Required for accreditation and dispute resolution, but it reads data
produced by Stories 1-3.

**Independent Test**: With read-only access, look up a certificate number and confirm the
history, document integrity checks, and reproduced verdict all display correctly.

**Acceptance Scenarios**:

1. **Given** a certificate number, **When** an auditor looks it up, **Then** the full
   consignment history is shown in chronological order with actor and timestamp for each entry.
2. **Given** an audited consignment, **When** the auditor requests re-assessment, **Then**
   the system reproduces the original verdict using the recorded rule-set version.
3. **Given** auditor access, **When** the auditor attempts any write action, **Then** it is
   refused.

---

### User Story 5 - Manage Conformity Rule Sets (Priority: P5)

A technical officer defines or updates the conformity criteria for a standard: required
tests, limits/tolerances, units, sampling plan, and the standard's number, edition, and
clause references. A new rule-set version takes effect from a stated date after approval by
a second authorized officer. Past decisions keep referencing the version used at the time.

**Why this priority**: Needed for scaling to more standards; initial rule sets can be
loaded once during setup for the first product categories.

**Independent Test**: Publish version 2 of a rule set with a tighter limit; verify old
decisions still cite version 1 and new assessments after the effective date use version 2.

**Acceptance Scenarios**:

1. **Given** a draft rule set, **When** the same officer who drafted it tries to approve it,
   **Then** the system refuses.
2. **Given** an approved new version with a future effective date, **When** an assessment
   runs before that date, **Then** the previous version is applied.

---

### Edge Cases

- A result is exactly on a limit boundary: the rule set MUST state whether limits are
  inclusive; the verdict follows that definition.
- Results are submitted in a different unit than the rule set (e.g., mA vs A): the system
  converts known units or rejects unknown ones; it never compares mismatched units.
- Two labs submit conflicting results for the same sample: both are recorded and the verdict
  is Requires Review.
- A participant's credentials are revoked after they signed records: their past records stay
  valid; new actions by them are refused.
- The automated assessment is unavailable: assessors can record a signed manual decision,
  flagged as manual.
- A consignment is split or partially rejected: each lot gets its own identifier linked to
  the parent consignment and its own verdict.
- A document must be deleted for legal reasons: the file is removed from storage, the deletion
  is recorded in history, and the record shows the document as "removed" rather than
  "tampered".
- The applicable standard has no active rule set: assessment returns Requires Review.

## Requirements *(mandatory)*

### Functional Requirements

**Recording and integrity**

- **FR-001**: System MUST register incoming consignments with: importer, product description
  and category, applicable standard (number and edition), quantity and unit, origin, arrival
  date, and receiving inspector.
- **FR-002**: System MUST assign each consignment a unique identifier that links all its
  subsequent entries.
- **FR-003**: System MUST store every lifecycle event as an append-only entry with actor
  identity, role, UTC timestamp, and digital signature.
- **FR-004**: System MUST NOT allow editing or deletion of existing entries; corrections MUST
  be new entries referencing the corrected entry and stating a reason.
- **FR-005**: System MUST record a fingerprint (cryptographic hash) of every attached document
  and detect any later alteration of that document.
- **FR-006**: System MUST NOT store personal data in the permanent ledger; personal data is
  kept in separately managed storage referenced by identifier.

**Assessment**

- **FR-007**: System MUST maintain versioned rule sets per standard edition, each listing
  required tests, limits, units, inclusivity of limits, and clause references.
- **FR-008**: System MUST accept test results from accredited laboratories, per sample and
  per test, with measured value, unit, method, and test report reference.
- **FR-009**: System MUST automatically evaluate submitted results against the rule set in
  effect at the consignment's arrival date and output per-test and overall verdicts
  (Conforming, Non-Conforming, Requires Review).
- **FR-010**: Every verdict MUST record the rule-set version and the input results used.
- **FR-011**: Re-running an assessment with the same inputs and rule-set version MUST produce
  the same verdict.
- **FR-012**: System MUST return Requires Review when required results are missing,
  conflicting, or in unconvertible units.

**Decisions**

- **FR-013**: Assessors MUST be able to confirm or override an automated verdict; overrides
  MUST include a written justification.
- **FR-014**: System MUST record a certificate reference for final Conforming decisions and
  a rejection record with reasons for Non-Conforming decisions.
- **FR-015**: System MUST notify the importer of final decisions and allow them to view the
  status and outcome of their own consignments.
- **FR-016**: System MUST record appeals and their outcomes as new decisions linked to the
  original.
- **FR-017**: System MUST support a signed manual-decision path, flagged as manual, when
  automated assessment is unavailable.

**Access and accountability**

- **FR-018**: Only enrolled participants with an organization-issued identity MUST be able to
  act; each role (inspector, laboratory, assessor, technical officer, importer, auditor,
  administrator) is limited to its permitted actions.
- **FR-019**: The party that submitted test results for a consignment MUST NOT be able to
  approve its conformity decision; the drafter of a rule set MUST NOT approve it.
- **FR-020**: Revoking a participant MUST block their new actions while keeping their past
  signed entries valid.
- **FR-021**: Importers MUST see only their own consignments.

**Audit and reporting**

- **FR-022**: Auditors MUST be able to retrieve full chronological history by consignment
  identifier or certificate number, with read-only access.
- **FR-023**: Auditors MUST be able to verify document integrity and reproduce any verdict
  independently.
- **FR-024**: System MUST provide reports of consignments by status, product category,
  standard, importer, and period, including override counts.

### Key Entities

- **Consignment**: A received shipment of goods; identifier, importer, product, applicable
  standard, quantity, origin, arrival date, status; may be split into Lots.
- **Lot / Sample**: A subdivision or sample of a consignment that is tested; links to parent
  consignment.
- **Evidence Document**: Invoice, packing list, photo, test report; stored separately,
  referenced by fingerprint.
- **Test Result**: Measured value, unit, method, sample, submitting laboratory, report
  reference.
- **Rule Set**: Versioned conformity criteria for one standard edition; tests, limits, units,
  clauses, effective date, drafter, approver.
- **Assessment Verdict**: Automated outcome per test and overall; rule-set version, inputs.
- **Decision**: Final assessor confirmation or override, justification, certificate or
  rejection reference; appeals produce new linked decisions.
- **Participant**: Enrolled person or organization with role(s) and status (active/revoked).
- **Ledger Entry**: Immutable signed record of any event, linked to a consignment or rule set.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of attempts to alter or delete a recorded entry or attached document are
  either refused or detected during verification.
- **SC-002**: Time from receipt of complete test results to automated verdict is under
  1 minute for 95% of consignments.
- **SC-003**: Average time from consignment arrival to final decision drops by at least 40%
  compared with the current manual process.
- **SC-004**: 100% of re-run assessments reproduce the original verdict.
- **SC-005**: An auditor can verify a consignment's full history and document integrity in
  under 5 minutes without operator help.
- **SC-006**: Decision errors caused by manual calculation or transcription fall to zero in
  the first 6 months after go-live.
- **SC-007**: An inspector can register a consignment in under 5 minutes.
- **SC-008**: Supports at least 500 consignments per day and 50 concurrent users without
  slowdown noticeable to users.

## Assumptions

- The ledger is a permissioned blockchain with smart contracts, as stated in the request and
  required by the project constitution; public networks and cryptocurrency are out of scope.
- Participants include internal staff (inspectors, assessors, technical officers,
  administrators, auditors), accredited external laboratories, and importers (view and
  appeal only); customs systems integration is out of scope for v1.
- Initial scope covers goods with measurable, rule-based criteria; criteria needing purely
  expert judgment always yield Requires Review.
- Initial rule sets cover a small set of pilot product categories chosen by the organization.
- Sampling is done by inspectors under existing procedures; the system records sampling but
  does not choose samples in v1.
- Certificates are issued under the organization's existing certification process; the
  system records the certificate reference and does not replace the certificate document.
- Off-chain evidence is retained for the organization's legal retention period (assumed
  10 years unless policy states otherwise).
- Participants have reliable network access at registration and laboratory sites.
- Notifications to importers use the organization's existing channels (e.g., email or portal).
