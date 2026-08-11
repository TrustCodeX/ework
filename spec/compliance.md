---
docname: draft-wilhelm-ework-compliance-00
title: "The e-work Protocol (EWP): Data Protection Compliance"
abbrev: EWP Compliance
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9420
---

This document maps data protection obligations onto e-work Protocol (EWP)
mechanisms: the consent object as processing record, subject rights as protocol
operations, erasure in a system with immutable history and end-to-end
encryption, and the account deletion flow.

It is written by an engineer, not a lawyer, and specialised legal review is a
prerequisite for treating any statement here as advice.

<!-- abstract -->

# Introduction

Most systems add compliance after the fact, as a layer of export buttons and
deletion jobs over a design that did not anticipate them. EWP's consent
mechanism happens to satisfy several requirements structurally, which is worth
stating precisely rather than claiming broadly.

## Conventions and Definitions

<!-- rfc2119 -->

# Roles of the Parties

| Party | LGPD / GDPR | HIPAA | Note |
|---|---|---|---|
| User | data subject | individual | |
| Issuer | controller of the data it processes for its own purpose | covered entity, when in health care | purpose declared in the consent object |
| Host | processor for the content, which it does not read; controller of the routing metadata it collects in order to operate | business associate when it serves a covered entity, under a BAA | the metadata list in the cryptography document bounds what it holds |
| Issuer gateway | processor for the issuer | business associate of the issuer | personal data outside the encryption boundary: see the retention section |
| Directory | controller of the attribute-to-host binding | | a legal basis of its own, an open question in the human discovery document |

The role follows from who determines the purposes and means of processing. The
table is the default arrangement, and specific contracts can shift it.

# Consent as Processing Record

The consent object already carries what a valid consent requires under the
common regimes: a specific purpose, granular scope, a term, revocation as simple
as granting, and a signed trail of state changes.

It is therefore simultaneously the auditable record of the processing
relationship. Accountability falls out of the design rather than being added.

The optional `regulatoryProfile` in the scope declares the issuer's regime and
makes visible, on the granting screen, the retentions that bind the issuer.

**This matters more than it appears.** What changes a person's decision to grant
is rarely the name of the statute; it is knowing what revoking will **not**
erase. An issuer under an invoicing retention obligation will keep the invoice
regardless of revocation, and saying so before the grant is the difference
between informed consent and a checkbox.

A host MUST NOT present an unregistered profile identifier as though it carried
meaning. An identifier the issuer invented, shown beside registered ones, would
borrow credibility it has not earned.

# Subject Rights

| Right | Typical basis | Mechanism in EWP |
|---|---|---|
| Access | LGPD art. 18; GDPR art. 15 | Full export, plus complete local view |
| Portability | LGPD art. 18; GDPR art. 20 | Export in a documented format, plus host migration |
| Rectification | LGPD art. 18; GDPR art. 16 | Correction as a new entry, visible, never a silent rewrite |
| Objection and withdrawal | LGPD art. 8 §5 and art. 18; GDPR art. 7(3) and art. 21 | `consent.revoke` with edge effect, plus silent retirement |
| Erasure | LGPD art. 18; GDPR art. 17 | As specified below |
| Information about processing | LGPD art. 9; GDPR arts. 13-14 | Record of discovery attempts, provenance by address, derived activity |
| Review of automated decisions | LGPD art. 20; GDPR art. 22 | Actor class plus human approval by key: the agent's decision is attributable and human review is demandable by construction |
| Incident notification | LGPD art. 48; GDPR arts. 33-34 | Notification through the protocol itself, as specified below |

The last row is worth noting. Regimes granting a right to human review of
automated decisions usually meet systems that cannot say whether a decision was
automated. EWP records the actor class of the signing key, so the question has
an answer that does not depend on the operator's word.

# Erasure With Immutable History and Encryption

The apparent conflict: a right to erasure against an immutable history and
against end-to-end encrypted copies held by counterparties. The resolution, in
layers:

1. **What is under the user's control is actually deleted.** Deleting a box
   removes ciphertext, blobs, and backups at the host. The host MUST perform
   this and MUST NOT retain a copy beyond the edge log.

2. **Counterparty copies are that counterparty's processing**, like an email
   already sent. The protocol delivers the erasure request and records it;
   honouring it is the counterparty's legal duty under their own regime, not
   magic performed by the protocol. Claiming otherwise would be selling remote
   deletion that no end-to-end encrypted system can perform.

3. **Cryptographic erasure.** Destroying a group's keys renders all ciphertext
   in that context permanently unreadable. For hosts and backups, which hold
   only ciphertext, key destruction is erasure in practice. EWP's position is
   that cryptographic erasure is effective erasure. The legal debate about
   whether orphaned ciphertext remains personal data is recorded as open, and
   legal review sets the final language.

4. **Immutability operates within the live context; erasure operates on the
   container.** The task, the relationship, or the whole account is erased,
   never an entry from the middle of a history. Either the context exists intact
   and auditable, or it ceases to exist entirely for that party. The two
   properties coexist with no hidden exception.

5. **Minimisation at the source remains the best defence.** Compartments and
   classification limit who receives what, and a message that never carried
   unnecessary personal data has nothing to erase.

# Account Deletion

A normative flow, actionable by the subject without support intervention:

1. **Offer of full export.** Accepting is optional.
2. **Revocation of all consents.** Issuers receive `consent.revoke` and stop at
   the edge.
3. **Cooling-off window**, RECOMMENDED 30 days: the account is frozen, nothing
   enters or leaves, and it is reversible with the recovery kit. All devices are
   informed, and any verified device MAY cancel. This last point is protection
   against coerced deletion and against deletion with a stolen kit.
4. **On expiry:** key destruction, removal of ciphertext and blobs, removal of
   the identity document and handle proofs.
5. **Addresses respond `unknown-recipient`**, indistinguishable from an address
   that never existed. Deletion does not become an oracle. This applies to every
   public surface, not only to delivery: a host that refuses delivery while its
   handle-proof endpoint still serves the identity document delivers the same
   information through another door.
6. **A destroyed address MUST NOT be registered by another identity.** Parties
   that used to send to it hold grants still valid on their side, and a new
   holder of the same handle would inherit correspondence that is not theirs.
   The host MUST refuse registration of a handle already destroyed. To avoid
   trading erasure for a list of who once had an account, the host SHOULD
   retain only a cryptographic digest of the address, sufficient to refuse and
   insufficient to enumerate.

Emergency access and inheritance change the outcome only if the subject
configured them beforehand.

# Retention

- **User side:** retention is the owner's choice; the protocol default is to
  keep until deleted.
- **Issuer side:** legal obligations live in the issuer's systems. EWP is not
  the issuer's legal archive. `regulatoryProfile` declares those retentions at
  grant time, making clear that revocation stops future sends and does not erase
  what law requires the issuer to keep.
- **Hosts:** routing metadata and edge logs SHOULD live the minimum necessary
  for operation and anti-abuse, with 90 days as an upper bound, and with the
  record of discovery attempts retained under the user's control.
- **Gateways and directories**, which handle personal data outside the
  encryption boundary, MUST publish a retention policy. It is a conformance
  requirement of the profile, not a courtesy.

# Incident Notification

Hosts, gateways and directories MUST publish a security contact in
`/.well-known/ework`; MUST notify affected subjects of a security incident
**through the protocol itself** (a system envelope, without depending on a
parallel email channel); and MUST record the notification. Deadlines and
authorities follow the applicable regime; the protocol supplies the channel and
the proof of sending.


A host that detects a breach affecting boxes MUST notify affected subjects
through the protocol, and MUST do so even where the content was ciphertext,
because metadata exposure is exposure.

The envelope type for system notices and the case where the breached party is
the host itself remain open questions.

# HIPAA in Particular

Health data raises requirements the other regimes do not. A host processing
identifiable health data on behalf of a covered entity is a business associate
and requires an agreement to that effect, which is a legal instrument outside
the protocol. End-to-end encrypted mode is the configuration in which the host
is not handling identifiable data at all, which is the cleanest posture and the
reason the mode exists.

The technical safeguards of 45 CFR 164.312 map directly onto what already
exists: access control is the keys and the compartments; the audit trail is
the immutable, chained history of the history document; integrity is the
signatures; transmission security is TLS plus end-to-end encryption. What the
protocol does not supply is the organisational part: the BAA and the
administrative policies.

Under the `us-hipaa@1` profile, PHI SHOULD travel only in end-to-end encrypted
mode; a collection holding PHI in assisted mode requires a host covered by a
BAA; and the six-year retention belongs to the covered entity, in its own
systems. Health payloads inherit the `health` classification by default.

# What the Protocol Does Not Promise

- It does not make an operator compliant. It provides mechanisms; compliance is
  an operational and legal property of the operator.
- It does not erase what counterparties hold.
- It does not adjudicate between conflicting regimes, for example an erasure
  request against a retention obligation. It makes both visible and leaves the
  resolution where it belongs.

An implementation MUST NOT advertise itself as "compliant by using EWP", and
official materials MUST NOT promise this.

# Security Considerations

Cryptographic erasure depends on key destruction actually destroying keys. An
implementation that keeps a key backup the host can decrypt has not erased
anything, and MUST NOT describe the operation as erasure.

Deletion also interacts with key backup: destroying the backup without
invalidating the recovery kit would leave a phantom recovery path, and the
ordering of the account deletion steps exists for that reason.

The cooling-off window is a period in which a frozen account still exists. The
requirement that it be indistinguishable from a nonexistent one, on every public
surface, is what keeps deletion from becoming an oracle for who is leaving.

The window is also a target for coercion and for deletion with a stolen kit.
Notice on every device, with cancellation by any of them, is the
countermeasure, and implementations MUST NOT allow the window to be shortened
remotely.

# Privacy Considerations

This document exists to turn rights into buttons. The cross-cutting rule it
consolidates: each party holds only what its function requires, for as long as
it requires, and the user can see what each party holds, through the metadata
list in the cryptography document, the record of discovery attempts in the
human discovery document, and the statement of what remains visible in the
compartments document.

# IANA Considerations

This document requests the creation of a registry, "EWP Regulatory Profiles",
with Specification Required as the registration policy, initially containing
`br-lgpd@1`, `eu-gdpr@1`, and `us-hipaa@1`.

Each profile entry states typical legal bases, minimum issuer-side retentions,
and additional requirements. Registration exists so that a profile identifier
displayed to a user means something specific.

# Open Questions

1. **Specialised legal review** before the public opening: the language of
   cryptographic erasure, the roles of the parties, and the content of the
   profiles. It blocks phase 4 of the roadmap.
2. Minors and accounts managed by a legal guardian: this depends on management
   delegation, an open question of the identity document, and carries
   requirements of its own under each regime.
3. International transfer (LGPD arts. 33 and following; GDPR chapter V): a
   host in one country serving a subject in another. The likely answer is
   jurisdiction transparency in `/.well-known/ework` plus the user's free
   choice of host.
4. Whether a reference impact assessment (RIPD under the LGPD, DPIA under the
   GDPR), as a companion document for host operators, is worth the effort.
5. **How long a destroyed address stays blocked**, and what happens if the
   subject wants to come back later with the same identity. Permanent blocking
   is the safe answer and wastes a small host's namespace forever; expiring
   the block reopens the window for inheriting someone else's correspondence,
   however narrow. The likely way out: permanent by default, with the original
   subject able to reclaim by proving possession of the old key, which
   requires keeping more than a digest and has to be weighed against the right
   to erasure.
