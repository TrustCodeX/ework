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
2. **Revocation of all consents the account granted.** Issuers receive
   `consent.revoke` and stop at the edge. The envelope is that of a polite
   revocation, with `from` on the relationship's address and `to` on the
   issuer, and it MUST NOT say that the account is leaving: a deletion reason
   would tell each issuer, during the window, the same thing step 5 hides from
   every other surface. The relationships in which the account is the issuer
   produce no envelope at all, because ending belongs to whoever granted, as
   the consent document says. The sending capability the account holds in
   those relationships is its own local state: it stands during the window,
   when the frozen account sends nothing and cancelling gives it back intact,
   and it is destroyed with everything else in step 4. On the other side, the
   grantor's record stays as it was until the grantor ends it, and the
   relationship's address keeps waiting for a capability that no longer exists
   anywhere, issued to a handle that no one registers again (step 6).
3. **Cooling-off window**, RECOMMENDED 30 days: the account is frozen, nothing
   enters or leaves, and it is reversible with the recovery kit. All devices are
   informed, and any verified device MAY cancel. This last point is protection
   against coerced deletion and against deletion with a stolen kit. "Nothing
   leaves" applies to every door that acts on the account's behalf, not only to
   its session. A credential the account gave to an outside system (the issuer
   gateway's API key and the calendar feed's address, as the bridges document
   describes) MUST be refused with the same response as a credential that never
   existed, and MUST work again on cancellation. Whatever was already in the
   outbound queue on the account's behalf MUST be held, with no delivery
   attempt: it returns to the queue if the account cancels, and it is destroyed
   in step 4. Revocation is the exception and MUST keep leaving, both that of
   step 2 and that of any relationship the account ended before the request,
   because it is what makes the issuer stop and it does not say that the
   account is leaving. A scheduled release (urgency document) whose time falls
   inside the window MUST NOT be released, neither during the window nor after
   cancellation.
4. **On expiry:** key destruction, removal of ciphertext and blobs, removal of
   the identity document, the handle proofs, and the sending capabilities of
   the relationships in which the account is the issuer. The host MUST erase
   **everything** it holds on the account's behalf, not a list of categories:
   the requests it made that still await an answer, the trash, the blocks, the
   proofs of what the edge refused, the box's execution policy, the gateway
   keys and webhooks, the calendar feeds, whatever was held in the outbound
   queue (step 3), and the record of the deletion request itself. Only what
   protects another party survives, and not the account: the digest of the
   address (step 6), the revocations still in the queue (those of step 2 and
   those the account made before the request), which follow the retention of
   routing metadata, and, when the
   account belongs to an organisation, the `id` of its in-person offers on the
   domain's revocation list, as the consent document specifies. An offer that
   was still valid MUST enter the list at this step and not before, because
   revocation is final and whoever cancels within the window gets the account
   back with its plates still working. An offer already on the list MUST stay
   on it, because a list that shrinks resurrects codes.
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

Step 2 gained these details because the reference implementation broke it
without anything noticing. The `consent.revoke` envelopes of a deletion went
into the queue with no `from`, no `to`, and no proof, the queue could not read
them, and no issuer ever learned that the relationship had ended. The same pass
included the relationships in which the account is the issuer, which would have
carried a revocation in the wrong direction had delivery worked, and the
envelope said `account-closed`.

Step 4 gained the rule of everything for the same reason. The reference
implementation erased by list, and the list fell behind the tables created
after it: the account's identifier survived destruction in eleven tables, the
gateway API key still authenticated, and the calendar feed still answered,
empty, to whoever had its address. Both would have become valid again for a new
account had the same identity registered at the host again, because whoever
checks them looks up the credential, not the account. The obvious fix, erasing
the offers as well, would be wrong twice: the live plate would still be accepted
on behalf of an address that no longer answers, and the revoked one would leave
the public list, which is the list shrinking.

Step 3 gained its details on what leaves because the reference implementation
closed only the client's door. The REST gateway checked the key without asking
about the account, and the system of a frozen organisation kept sending tasks
with `from` on the address that delivery already answered as nonexistent: a
recipient only had to compare the two to learn that the account was leaving.
The outbound queue kept trying to deliver what the account had sent before the
request, during the window and after destruction, on behalf of an address that
no longer existed. And the scheduled release kept releasing the escalation of an
account that could no longer give the `ack` that would silence it. Holding the
queue and discarding the due release are different choices on purpose: an
envelope in the queue is something the account already did, and cancellation
has to give it back; an escalation step is an alarm bound to a time, and
releasing it after cancellation would alarm the contact about a moment that has
already passed.

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
  record of discovery attempts retained under the user's control. Each routing
  memory has its deadline in the document that defines it, and all of them fit
  under that bound: the outbound queue, the deduplication memory and the
  transport nonce memory in the federation document, and the session challenge
  in the sync document. The longest-lived is a given-up delivery, at 60 days at
  most.
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

Account deletion does not announce itself through relationships either. Whoever
used to send to the account receives a revocation like any other, and whoever
had granted it permission receives nothing: the other end learns at most that
the relationship ended, and never that the person left the host. Nor through the
bridges: during the window, the gateway key and the calendar feed are refused
as credentials that never existed, not with a reason (step 3). Whoever uses them
is a third party (the organisation's ERP, whoever operates it, the calendar
service), and a reason would tell them what the window hides.

What survives destruction is chosen by what another party needs, not by what
the host might find useful to keep. The digest of the address keeps anyone from
inheriting someone else's correspondence, the revocation still in the queue
tells whoever used to send that the relationship ended, and the offer `id`s keep
anyone from accepting the code of whoever left. None of the three says who the
account was: the digest cannot be enumerated, the revocation carries only the
relationship's address, which the sender already knew, and in-person offers
exist only for organisation accounts, whose `id` is already printed on the
plate. The proof that destruction happened
is the digest itself, and the record of the deletion request, which held the
account's identifier in clear, goes with everything else: keeping it would build
the list of who once had an account there that step 6 avoids.

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
6. **Which envelope carries the erasure request.** The erasure section says the
   protocol delivers the erasure request to counterparties and records it, and
   no type in the core envelope registry carries it. In account deletion it
   does not fit the `consent.revoke` of step 2, which cannot say that the
   account is leaving, nor anything sent during the window, which has to hide
   the departure. The likely place is a type of its own, sent in step 4 to the
   counterparties that received copies. What remains open is who receives it
   (only those who used to send to the account, or also those who had granted
   it permission) and how it avoids becoming, in step 4, the announcement that
   step 2 avoids.
7. **A relationship change still in the queue when deletion is requested.** Of
   what was in the queue, only revocation leaves during the window (step 3). If
   the account changed a relationship's address (identity document, address
   rotation) and the rotation statement had not arrived, it is held, and the
   step 2 revocation leaves from the new address, which the issuer does not know
   yet: on the issuer's side the relationship stays up until it tries to send
   and has the delivery refused. The same holds for a grant that had not arrived.
   Nothing leaks, because on the issuer's side the outcome is that of a silent
   retirement, but step 2 loses its effect of making the issuer stop. Letting
   the rotation statement and the grant of a relationship the account granted
   leave as well would solve it, at the price of more exceptions to "nothing
   leaves".
