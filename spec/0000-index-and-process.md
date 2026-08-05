# EW-RFC 0000: Index and specification process

**Status:** Draft | **Created:** 2026-08-04 | **Editor:** Michel Wilhelm

## 1. What an EW-RFC is

An EW-RFC is a normative document of the e-work protocol (EWP) specification. The set of EW-RFCs in Stable status defines what "implementing EWP" means. Documents in `docs/` are context and justification; in case of conflict, the EW-RFC prevails.

## 2. Normative keywords

The words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY**, in capitals, follow the semantics of RFC 2119 and RFC 8174. The Portuguese drafts use DEVE, NÃO DEVE, DEVERIA, NÃO DEVERIA and PODE with exactly these meanings; English is the canonical version.

## 3. Life cycle of an EW-RFC

`Draft` → `Proposed` → `Stable` → `Obsolete`

- **Draft:** under active writing; it may change without ceremony; implementations are experiments.
- **Proposed:** complete and internally consistent; changes require a recorded justification; the moment for two independent implementations to exist.
- **Stable:** frozen except for errata; incompatible changes require a new EW-RFC that supersedes it.
- **Obsolete:** superseded; it points at its successor.

The change process at this stage of the project: direct editing by the editor, with relevant decisions recorded in `docs/decisoes/`. Multi-party governance is a prerequisite for any RFC to become Stable (an open question, see section 7).

## 4. Mandatory structure

Every EW-RFC has these sections: Summary; Motivation; Specification; Security considerations; Privacy considerations; Open questions; References. RFCs that create extension points also include an Extension registry.

## 5. Protocol versioning

- The protocol version is a `MAJOR.MINOR` string (currently `0.1`). Every message carries the `ewp` field with the version.
- Within `0.x`, anything may break. From `1.0` onwards: MINOR only adds; MAJOR may remove.
- Capabilities and extensions are announced by URN (`urn:ework:core`, `urn:ework:payments` and so on) in the session (EW-RFC 0004) and in host discovery (EW-RFC 0001), allowing granular evolution without version lockstep.

## 6. Index

The English versions of RFCs 0001 to 0016 are the Internet-Drafts in `spec/drafts/`, which is what this site serves.

| RFC | Title | Status |
|---|---|---|
| [0000](0000-index-and-process.md) | Index and specification process | Draft |
| [0001](core.md) | Core: conventions, URIs, discovery, envelopes | Draft |
| [0002](data-model.md) | Data model: Task, Project, attachments, actions | Draft |
| [0003](identity.md) | Identity: keys, handles, devices, recovery | Draft |
| [0004](sync.md) | Client-server synchronisation and blobs | Draft |
| [0005](federation.md) | Server-to-server federation | Draft |
| [0006](crypto.md) | End-to-end encryption | Draft |
| [0007](consent.md) | Consent and issuers | Draft |
| [0008](payloads.md) | Typed payloads: payment, scheduling, delivery, approval | Draft |
| [0009](urgency.md) | Urgency, acknowledgement and escalation | Draft |
| [0010](bridges.md) | Bridges: CalDAV, email, issuer gateway, export | Draft |
| [0011](anti-abuse.md) | Anti-abuse: spam, marketing, fraud and urgency abuse | Draft |
| [0012](human-discovery.md) | Discovery and human identifiers (phone, email) | Draft |
| [0013](compartments.md) | Compartments and sensitive data | Draft |
| [0014](actors.md) | Autonomous actors and execution confirmation | Draft |
| [0015](history.md) | Entries, comments and task history | Draft |
| [0016](compliance.md) | Privacy and regulatory compliance (LGPD, GDPR, HIPAA) | Draft |

## 7. Open questions

1. **The protocol's final name.** `e-work` and EWP are a working name. Criteria for the decision: an available domain, pronounceable in Portuguese and English, no collision with trademarks. Candidates noted: e-work, taskwire, opentask (collides), tasknet (collides). To be decided before the translation.
2. **Governance:** which entity keeps the specification and the extension registry once third parties are implementing (the lesson of AT Protocol's PLC association).
3. **The licence** of the specification (proposed: CC BY-SA 4.0) and of the reference implementations (proposed: Apache-2.0).
4. **The extension registry:** at this stage it is a file in the repository; the registration process is to be formalised once a second implementation exists.
