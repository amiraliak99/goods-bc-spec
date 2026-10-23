<!--
Sync Impact Report
- Version change: (template, unversioned) → 1.0.0
- Modified principles: all template placeholders replaced (initial adoption)
  - [PRINCIPLE_1_NAME] → I. Immutable Record of Truth
  - [PRINCIPLE_2_NAME] → II. Deterministic, Versioned Conformity Rules
  - [PRINCIPLE_3_NAME] → III. Minimal On-Chain Data, Anchored Off-Chain Evidence
  - [PRINCIPLE_4_NAME] → IV. Permissioned Identity and Least Privilege
  - [PRINCIPLE_5_NAME] → V. Smart Contract Safety (NON-NEGOTIABLE)
- Added principles: VI. End-to-End Traceability and Auditability, VII. Simplicity and Proven Components
- Added sections: Domain and Compliance Constraints; Development Workflow and Quality Gates
- Removed sections: none
- Deferred TODOs: none (ratification date set to initial adoption date 2026-10-23)
-->

# Incoming Goods Conformity Ledger Constitution

A blockchain and smart contract-based system for immutable recording and automated
conformity assessment of incoming goods in a standards organization.

## Core Principles

### I. Immutable Record of Truth

- Every event in an incoming-goods lifecycle (registration, sampling, test result
  submission, conformity decision, certificate issuance, rejection, appeal) MUST be written to
  the ledger as an append-only, signed transaction.
- Recorded transactions MUST NOT be edited or deleted. Corrections MUST be new transactions
  that reference the record they supersede and state the reason.
- The current state of any consignment MUST be derivable solely by replaying its ledger
  history.

**Rationale**: Conformity decisions carry legal and commercial weight; trust depends on
records that no single party, including the operator, can silently alter.

### II. Deterministic, Versioned Conformity Rules

- Conformity criteria (limits, tolerances, required tests, sampling plans) MUST be encoded
  as smart contract logic or contract-readable rule sets that reference the exact standard
  and its edition.
- Given identical inputs and rule version, assessment MUST produce identical outcomes on
  every node. Contracts MUST NOT depend on wall-clock time, randomness, or external calls
  except through explicitly designated, signed oracle inputs.
- Every decision MUST record the rule-set version that produced it. Rule changes MUST be
  published as new versions; they MUST NOT retroactively change past decisions.
- Manual overrides by authorized assessors are permitted only as separate, signed
  transactions carrying a mandatory justification.

**Rationale**: Automated assessment is only defensible if any decision can be reproduced
and tied to the published standard in force at the time.

### III. Minimal On-Chain Data, Anchored Off-Chain Evidence

- Large or sensitive artifacts (lab reports, images, invoices, personal data) MUST be stored
  off-chain. The ledger MUST store only identifiers, cryptographic hashes (SHA-256 or
  stronger), and the minimum fields required for assessment.
- Any off-chain document used in a decision MUST be verifiable against its on-chain hash;
  a hash mismatch MUST be treated as tampering and block the decision.
- Personal data MUST NOT be written on-chain, since it can never be erased there.

**Rationale**: Keeps the ledger lean and preserves the ability to meet data-protection and
retention obligations without breaking immutability.

### IV. Permissioned Identity and Least Privilege

- The network MUST be permissioned. Every participant (importer, inspector, laboratory,
  assessor, administrator, auditor) MUST hold a verifiable identity issued by the
  organization's trust authority.
- Each role MUST be limited to the contract functions it needs; enforcement MUST happen
  in contract logic, not only in client applications.
- No single role or node MUST be able to both submit test results and approve conformity
  for the same consignment (segregation of duties).
- Key revocation MUST take effect on the ledger without invalidating historical signatures.

**Rationale**: A standards body is accountable for who decided what; anonymous or
over-privileged access undermines that accountability.

### V. Smart Contract Safety (NON-NEGOTIABLE)

- Contract code MUST be developed test-first: tests for each conformity rule and each
  access rule are written and seen failing before implementation.
- Every contract change MUST pass unit tests, scenario tests covering conforming,
  non-conforming, boundary, and malicious inputs, and static analysis before deployment.
- Contract upgrades MUST follow a documented, multi-party approved procedure, MUST preserve
  existing ledger state, and MUST be recorded on-chain with version and approvers.

**Rationale**: Deployed contract defects directly produce wrong legal decisions and are
costly to remediate on an immutable ledger.

### VI. End-to-End Traceability and Auditability

- Every consignment MUST be traceable from arrival to final decision through a single
  identifier linking all its transactions and anchored evidence.
- Auditors MUST be able to independently verify any decision (inputs, rule version,
  signatures, evidence hashes) using read-only access, without operator assistance.
- System components MUST emit structured logs correlated to ledger transaction IDs.

**Rationale**: External accreditation and dispute resolution require evidence that can be
checked by a third party.

### VII. Simplicity and Proven Components

- Use established, maintained blockchain platforms and libraries; custom cryptography or
  consensus MUST NOT be implemented.
- Add components, services, or abstractions only when a current requirement needs them;
  speculative generality MUST be rejected in review.
- Each added dependency or service MUST be justified in the implementation plan.

**Rationale**: Fewer moving parts mean fewer failure points in a system whose correctness
is legally significant.

## Domain and Compliance Constraints

- Assessment workflows MUST align with the organization's accredited procedures and the
  relevant conformity assessment standards (e.g., ISO/IEC 17020 for inspection,
  ISO/IEC 17025 for laboratory results, ISO/IEC 17065 for certification).
- Each conformity rule MUST cite its source standard number, edition, and clause.
- Retention periods for off-chain evidence MUST meet the organization's legal record-keeping
  requirements; deletion of off-chain data MUST be logged on-chain by hash reference.
- The system MUST remain usable for decision recording when automated assessment is
  unavailable, via the signed manual-assessment path defined in Principle II.
- All timestamps MUST be recorded in UTC; user interfaces MAY display local time.

## Development Workflow and Quality Gates

- Features follow the Spec Kit flow: specify, clarify, plan, tasks, implement. Plans MUST
  include a Constitution Check confirming compliance with every principle above.
- All changes MUST be reviewed by at least one person other than the author; smart contract
  and access-control changes MUST be reviewed by two.
- CI MUST run unit, contract scenario, and integration tests against a local multi-node test
  network; failing gates block merge.
- Release notes MUST list any rule-set or contract version changes and their effective dates.

## Governance

- This constitution supersedes other project practices. Conflicts are resolved in its favor.
- Amendments require a written proposal stating the change, rationale, and migration impact,
  approval by the project lead and a representative of the conformity assessment
  department, and an updated version line below.
- Versioning follows semantic versioning: MAJOR for removing or redefining a principle,
  MINOR for adding a principle or materially expanding guidance, PATCH for clarifications.
- Every plan and code review MUST verify compliance. Any deviation MUST be recorded in the
  plan's Complexity Tracking section with justification and the simpler alternative rejected.
- Compliance is reviewed at each release and at least once per year.

**Version**: 1.0.0 | **Ratified**: 2026-10-23 | **Last Amended**: 2026-10-23
