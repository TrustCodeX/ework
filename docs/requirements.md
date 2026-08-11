# Requirements

Conventions: **RF** functional requirement, **RNF** non-functional requirement. MoSCoW priority: **[M]** must, **[S]** should, **[C]** could, **[W]** won't (in this version). The "Where" column points at the RFC that specifies it.

## Identity and accounts

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RF-01 | The root of the user's identity is a key pair belonging to them, not to the server | [M] | RFC 0003 |
| RF-02 | A readable address `name@domain` as a verifiable alias for the key, with proof in both directions | [M] | RFC 0003 |
| RF-03 | Multiple devices per identity, with per-device keys signed by the root (cross-signing) and verification between devices | [M] | RFC 0003, 0006 |
| RF-04 | Account recovery: an offline recovery kit, mandatory at creation; custody optional | [S] | RFC 0003 |
| RF-05 | Host migration with no loss of identity, with a complete export and import and redirection | [M] | RFC 0003, 0004 |
| RF-06 | One identity can have boxes on multiple hosts; one client aggregates N boxes | [S] | RFC 0003, 0004 |
| RF-07 | Organisation identity anchored in a domain, with a published, verifiable key | [M] | RFC 0003 |
| RF-08 | N contact keys per identity, one per relationship, each with its own address and never listed publicly | [M] | RFC 0003 |
| RF-09 | Key rotation with proof of continuity, selective migration of correspondents and silent retirement with no oracle | [M] | RFC 0003, 0011 |

## Task model

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RF-10 | A task with a title, description, deadlines, time zone, priority, recurrence and duration estimate | [M] | RFC 0002 |
| RF-11 | Typed dependencies between tasks forming a DAG, with a gap and computation of actionability | [M] | RFC 0002 |
| RF-12 | Multi-party projects (epics) with members from different providers, roles and visibility policies | [M] | RFC 0002, 0006 |
| RF-13 | Negotiation state (pending, accepted, declined, countered) separate from execution state (to do, in progress, blocked, completed, failed, cancelled) | [M] | RFC 0002 |
| RF-14 | Attachments: a list of typed attachments per task (files, data), as raw binary blobs addressed by hash, encrypted in E2EE mode | [M] | RFC 0002, 0004 |
| RF-15 | Registrable typed payloads: payment (bill or instant payment), scheduling, delivery, approval | [M] | RFC 0008 |
| RF-16 | Structured actions on the task (view, open link, pay, approve, copy), always executed with the user's confirmation | [S] | RFC 0002 |
| RF-17 | Delegating a task to another identity | [C] | RFC 0002 |
| RF-18 | Task update by the issuer (a reissue, a reschedule) as a revision of the same task, subject to acceptance | [M] | RFC 0002, 0005 |
| RF-60 | Compartments per project, each with its own cryptographic group, by subject or by relationship | [M] | RFC 0013 |
| RF-61 | A task restricted to a compartment: whoever is outside does not receive the packet and the task does not exist for them | [M] | RFC 0013 |
| RF-62 | Sealed section: sensitive fields of a shared task encrypted for a narrower compartment | [M] | RFC 0013 |
| RF-63 | Marking sensitive data by label, with the project's policy deciding the compartment | [M] | RFC 0013 |
| RF-64 | Default classification by payload type: closing is automatic, opening is an explicit and recorded act | [M] | RFC 0008, 0013 |
| RF-65 | Dependency between compartments through a public milestone, without leaking the existence of the private task | [M] | RFC 0002, 0013 |
| RF-66 | A visibility restriction with no key behind it is a hint, and the interface has to label it as such | [M] | RFC 0013 |
| RF-70 | Actor class declared per key (human, assisted, autonomous, system), signed by the root | [M] | RFC 0003, 0014 |
| RF-71 | Delegation to an agent with scope (circles, actions, payload types, amount, rate, validity) and immediate revocation | [S] | RFC 0014 |
| RF-72 | A requirement for human approval before execution, anchored in a key rather than in trust in the client | [M] | RFC 0014 |
| RF-73 | Execution confirmation, including by a third party who did not execute, with the possibility of dispute | [M] | RFC 0014 |
| RF-74 | An `awaiting-confirmation` state with a deadline and a configurable outcome | [M] | RFC 0002, 0014 |
| RF-75 | Evidence that can be required at completion (a supporting attachment or a structured result) | [S] | RFC 0014 |
| RF-76 | A machine-to-machine profile with acceptance criteria, a structured result and idempotency | [S] | RFC 0014 |
| RF-77 | Composition of policies by taking the stricter of issuer and recipient | [M] | RFC 0014 |
| RF-80 | The entry as the unit of history: message and action in the same signed object, a null action being an ordinary comment | [M] | RFC 0015 |
| RF-81 | The entry's audience by group: a private note, the relationship with the issuer, or the project circle | [M] | RFC 0015 |
| RF-82 | A registered vocabulary of actions, with an unknown action ignored without discarding the entry | [M] | RFC 0015 |
| RF-83 | Deterministic causal ordering, identical across all clients | [M] | RFC 0015 |
| RF-84 | Activity derived locally from the signed history, never transmitted as an assertion | [M] | RFC 0015 |
| RF-85 | Justification that can be required per action type, rejected by the peer when it is missing | [S] | RFC 0014, 0015 |
| RF-86 | An entry with an action is immutable: it is neither removed nor edited, by anyone. Correcting means adding | [M] | RFC 0015 |
| RF-87 | Verifiable export of a task's history, without depending on the host that stored it | [S] | RFC 0004, 0015 |
| RF-88 | History chained by content hash: editing, removing and withholding entries all detectable by mechanical verification | [M] | RFC 0015 |
| RF-89 | Freshness proof for the identity document and gossip of heads within groups, against host rollback | [S] | RFC 0003 |

## Issuers and consent

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RF-20 | Consent as a first-class object: id, scope, status policy, validity, states, auditable | [M] | RFC 0007 |
| RF-21 | An unknown issuer reaches only a minimal consent request, in silent quarantine (never an audible notification) | [M] | RFC 0007 |
| RF-22 | Revoking as simple as granting, taking effect immediately at the edge of the user's server | [M] | RFC 0005, 0007 |
| RF-23 | A status privacy policy per relationship and per task: nothing, receipt, milestones, full | [M] | RFC 0007 |
| RF-24 | A deduplication key per issuer: a resend updates, it does not duplicate | [M] | RFC 0007 |
| RF-25 | Open issuer verification (domain plus key), with no central gatekeeper | [M] | RFC 0003, 0007 |
| RF-26 | Auto-accept rules per issuer, defined by the user, off by default | [S] | RFC 0007 |
| RF-27 | A purpose declared in the consent scope and on every offer, from a registered vocabulary | [M] | RFC 0007, 0011 |
| RF-28 | Leak attribution: the client shows which address the task arrived through and who it was given to | [M] | RFC 0003, 0011 |
| RF-29 | First-contact rules: highlighted, with no auto-accept and no urgency above normal | [M] | RFC 0011 |

## Synchronisation and federation

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RF-30 | Multi-device delta synchronisation with exact state (no full resync in normal operation) | [M] | RFC 0004 |
| RF-31 | Server-to-server delivery with a signed envelope, retries with backoff defined in the specification, and a delivery receipt | [M] | RFC 0005 |
| RF-36 | Two-layer discovery: an SRV record `_ework._tcp` locates the host, an HTTPS document describes the key and capabilities | [M] | RFC 0001 |
| RF-37 | The host declares in `addressDomain` which addresses it serves, and the discoverer checks before trusting | [M] | RFC 0001 |
| RF-32 | Platform-agnostic push (Web Push, UnifiedPush), with no content in the push by default | [M] | RFC 0004, 0009 |
| RF-33 | Offline-first operation in the clients, with reconciliation on reconnecting | [S] | RFC 0004 |
| RF-34 | Complete account export (tasks, consents, blobs, identity) in a documented format | [M] | RFC 0004 |
| RF-35 | Self-authenticating objects: the author's signature on the object, in addition to the transport signature | [M] | RFC 0001, 0005 |

## Urgency and escalation

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RF-40 | Urgency levels (minimal, low, normal, high, critical) mapped to notification priority | [M] | RFC 0009 |
| RF-41 | Acknowledgement distinct from completion; the acknowledgement pauses escalation | [M] | RFC 0009 |
| RF-42 | An escalation policy: timeout, repetitions, escalation contacts with prior consent | [S] | RFC 0009 |
| RF-43 | The user's quiet hours, pierced only for authorised critical urgency | [S] | RFC 0009 |
| RF-44 | Critical urgency available only to issuers whose consent includes that scope | [M] | RFC 0007, 0009 |

## Discovery and human identifiers

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RF-50 | Verified attributes (E.164 phone and email) bound to the identity by a signed attestation, with validity | [S] | RFC 0012 |
| RF-51 | Discovery off by default, with opt-in per attribute, per requester class and per directory | [M] | RFC 0012 |
| RF-52 | Blind delivery: the directory forwards a consent request and never translates an attribute into an address | [M] | RFC 0012 |
| RF-53 | No reverse lookup, no bulk export, no list checking, no existence oracle | [M] | RFC 0012 |
| RF-54 | A record, visible to the user, of who tried to reach them, through which attribute and with what outcome | [M] | RFC 0012 |
| RF-55 | Civil identifiers (national ID and social security numbers and equivalents) out of scope: no part of the protocol indexes people by civil registry | [M] | RFC 0012 |
| RF-56 | No attribute is required in order to create or use an identity | [M] | RFC 0012 |

## Cryptography and privacy

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RNF-01 | E2EE by default: content in encrypted groups (MLS); only members open the packet | [M] | RFC 0006 |
| RNF-02 | A member joining a group gains access to the future; access to history is an explicit and configurable re-share | [M] | RFC 0006 |
| RNF-03 | Assisted mode per collection, always explicitly opt-in, never the default | [S] | RFC 0006 |
| RNF-04 | Cleartext metadata minimised and listed exhaustively in the specification | [M] | RFC 0005, 0006 |
| RNF-05 | Forward secrecy and post-compromise security at group level | [S] | RFC 0006 |
| RNF-06 | Key backup encrypted with a recovery key, hosted without trusting the host | [M] | RFC 0006 |
| RNF-07 | Status returned to the issuer signed by the user and minimised by the privacy policy | [M] | RFC 0006, 0007 |
| RNF-08 | The client never fetches a remote resource referenced in task content (the end of the tracking pixel) | [M] | RFC 0011 |
| RNF-09 | Unlinkability between issuers: two issuers do not discover they are talking to the same person | [S] | RFC 0003 |

## Platform and operations

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RNF-10 | An HTTPS substrate (HTTP/2 or better) and WebSocket; no exotic port required | [M] | RFC 0001 |
| RNF-11 | Blobs travel as raw binary (no base64), addressed by hash, with resumable upload and ranged download | [M] | RFC 0004 |
| RNF-12 | Envelopes in canonical JSON; optional negotiable CBOR encoding | [C] | RFC 0001 |
| RNF-13 | A reference host operable by a hobbyist: one binary, one database, one domain | [S] | outside the spec |
| RNF-14 | A minimal client implementable in weeks by reading RFCs 0001 to 0004 | [S] | process |
| RNF-15 | Usable by non-technical people: every security decision has a simple UX path defined alongside it | [M] | all |
| RNF-16 | Nothing in the protocol depends on FCM or APNs specifically, nor on any single provider | [M] | RFC 0004 |
| RNF-17 | Encrypted envelopes padded to size classes: the host sees the class, not the exact size | [S] | RFC 0001, 0006 |
| RNF-18 | Declared cryptographic agility, with a hybrid post-quantum ciphersuite as the phase 2 target | [S] | RFC 0006 |
| RNF-19 | Authority per action verified by the peer: a valid signature is not enough to transition someone else's task | [M] | RFC 0015, 0004 |
| RNF-20 | No object destined for an issuer carries the root key: a pseudonym per relationship, not linkable between issuers | [M] | RFC 0003, 0006, 0007, 0015 |
| RNF-21 | The recovery kit derivation specified with test vectors, so that conforming clients agree | [M] | RFC 0003 |
| RNF-22 | Equality of edge responses covers a live address with no relationship, and not only a dead address | [M] | RFC 0003, 0005, 0011 |
| RNF-23 | Every relationship has an urgency ceiling, including between people, raised only by the recipient | [M] | RFC 0007, 0009, 0011 |
| RNF-24 | Every relationship between people has a specified path to end it, silent and immediate at the edge | [M] | RFC 0007 |

## Regulatory compliance

| ID | Requirement | Prio | Where |
|---|---|---|---|
| RF-90 | Data subject rights (LGPD, GDPR, HIPAA and analogues) mapped onto protocol mechanisms, with profiles per regime | [M] | RFC 0016 |
| RF-91 | Account deletion flow: export, revocations, a grace window and a final answer with no oracle | [M] | RFC 0016 |
| RF-92 | Cryptographic destruction as effective erasure, at container level, never by editing the middle of the history | [M] | RFC 0006, 0016 |
| RF-93 | The issuer's regulatory profile and legal retention periods declared in the consent, visible when granting | [S] | RFC 0007, 0016 |
| RF-94 | Notice of a security incident to data subjects through the channel itself, with a published security contact | [M] | RFC 0016 |
| RF-95 | PII outside E2EE (gateways, directories) with a published and minimal retention policy | [M] | RFC 0016 |

## Out of this version [W]

- On-call shifts and rotations (S4 covers personal escalation; organisational on-call is a product layer).
- Reputation and global directories of issuers (pluggable, outside the core).
- Chained subdelegation (an agent delegating to another agent): the simple scope of RFC 0014 §2 is enough for v0.1.
- Payment settlement inside the protocol.
