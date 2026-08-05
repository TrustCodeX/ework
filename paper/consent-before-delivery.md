# Consent Before Delivery: A Protocol Primitive for Federated Task Exchange

**Michel Wilhelm**
michelwilhelm@gmail.com

*Preprint. Draft of 2026-08-05.*

## Abstract

Messaging systems deliver first and filter afterwards. This ordering places the
cost of unsolicited contact on the recipient, and two decades of spam filtering,
sender authentication, and reputation systems have improved the filtering
without changing the ordering. We present the e-work Protocol (EWP), a
federated protocol for exchanging actionable tasks in which consent is the
precondition for delivery, enforced at the receiving host before content reaches
a recipient's store.

Three mechanisms make the inversion practical rather than merely stated. First,
a scoped, revocable, signed grant binds an issuer to a purpose, a payload type,
a volume ceiling, and an urgency ceiling, all enforced at the edge. Second,
each grant issues a distinct opaque address to its counterparty, which yields
leak attribution, surgical severance, and non-linkability between issuers.
Third, an address may be retired without notice, so that ending a relationship
requires no interaction with a counterparty that has demonstrated it should not
receive one.

We report a reference implementation deployed across two independent domains,
and four defects it surfaced that are properties of the design space rather than
of the code: the absence of an enumeration oracle is a property of an endpoint
*set* and not of any endpoint in isolation; a status-sharing policy expressed on
the transported object hands the decision to the sender; a host that signs
client-composed envelopes without validating them becomes blank letterhead; and
a discovery document whose advertised limits diverge from enforced limits
produces failures that a conforming client cannot diagnose.

## 1. Introduction

A person receives a utility bill, a medical appointment reminder, a request to
review a contract, and a delivery window. Each is a task: it has an issuer, a
deadline, a state, and a resolution. Each arrives today as prose, and the
recipient performs by hand the structuring that the issuer already had: reading,
extracting the date, transcribing it, and remembering to act.

The structure exists at the origin. The utility company holds the amount, the
due date, and the payment reference as database fields, discards that structure
to render an email, and the recipient reconstructs it. This is a protocol gap,
not a product gap, and it has an obvious shape: a task protocol, standing to
task exchange as SMTP stands to message exchange.

The obvious shape is also the trap. A task protocol built like email inherits
email's ordering, and a system that delivers structured, actionable, deadline-
bearing objects to anyone who learns an address is worse than one that delivers
prose. The structure that makes the task useful to the recipient makes it more
useful to an abuser.

### 1.1 Contribution

We claim four contributions.

1. **Consent as a protocol primitive rather than an application feature.** We
   specify a grant object, a scope that bounds it, and a capability that binds
   it to a single address, all enforced by the receiving host at its edge. We
   discuss the cost, which is friction at first contact, and argue it is
   inherent rather than incidental.

2. **Per-relationship addressing as an anti-abuse mechanism**, with four
   properties that follow: leak attribution, surgical severance, worthlessness
   of harvested addresses, and non-linkability between issuers.

3. **Silent retirement**, an exit path that communicates nothing, and its
   composition with selective key rotation. We argue that the choice between
   informing a counterparty and not informing it is a design axis that existing
   systems collapse.

4. **Four empirical findings** from a deployed implementation, each of which
   generalises beyond this protocol.

### 1.2 Scope

We specify and implement; we do not yet evaluate quantitatively. Section 7
states what an evaluation must measure and why the current deployment does not
constitute one. We consider it more useful to publish the design and the
findings than to withhold both pending measurement.

## 2. Related Work

**Electronic mail** delivers unconditionally. SPF, DKIM, and DMARC authenticate
senders without authorising them, and the filtering that follows is
probabilistic. Adding structure to email, as with structured email proposals,
improves the payload without changing the ordering.

**ActivityPub** and the fediverse establish relationships through following,
which is closer to consent than email achieves, but the relationship is coarse:
following authorises a stream, not a scope, and there is no capability bound to
an address. Blocking is a filter applied after delivery.

**Matrix** provides federated, end-to-end encrypted rooms with invitation-based
membership. Invitation is consent-like, but the unit is a room rather than a
scoped relationship, and the protocol does not model tasks, negotiation, or the
status-sharing asymmetry between an issuer and a recipient.

**JMAP** contributes the batched-call and state-string synchronisation model we
adopt, and is a transport for mail rather than a consent mechanism.

**Solid** inverts data custody, giving the person a pod they control. It
addresses where data lives; we address whether an interaction may begin.

**Capability systems**, from object-capability models to macaroons, supply the
technical shape of our capability, bound to a single address and a single grant.
Our contribution is not the capability construct but its placement: at the edge
of a federated host, as the precondition for delivery.

**Calendar protocols**, iCalendar, CalDAV, and JSCalendar, model the object well
and model neither cross-organizational delegation nor negotiation. We profile
JSCalendar rather than inventing a vocabulary, and extend only where a gap
exists.

The closest prior position is the observation, recurrent in anti-spam
literature, that the economics of unsolicited messaging depend on delivery being
free. Proof-of-work proposals attack that by taxing delivery. We attack it by
requiring authorisation, which taxes the abuser and not the legitimate
high-volume issuer.

## 3. Design

### 3.1 Consent before delivery

An issuer with no relationship to a recipient may send exactly one envelope
type, a `consent.request`. The receiving host places it in quarantine and
generates no notification. Quarantine is visible on demand and does not
interrupt: a mechanism whose purpose is to protect attention cannot itself
demand attention on arrival.

Granting produces a signed object carrying purpose, payload types, a monthly
volume ceiling, a maximum urgency, and a status-sharing policy. The recipient
signs it, not the host; a host able to forge grants could authorise delivery to
its own users, making the mechanism decorative.

At delivery, the receiving host verifies that the address resolves, that a grant
exists in state `granted`, that the envelope carries the capability issued for
that grant, and that payload type, volume, and urgency are within scope. Failing
any of these, the envelope never reaches the store.

**The cost is friction at first contact**, and we consider it inherent. A design
that made first contact free would reintroduce the problem. What the design can
do is make the act cheap, informed, and reversible, and Section 5.2 reports a
finding about what "informed" requires.

### 3.2 Per-relationship addressing

On granting, the client generates an opaque address for that counterparty alone.
Four properties follow.

*Leak attribution.* Each issuer knows a different address, so unsolicited
contact arriving at a given address identifies which relationship leaked it.
This property requires that clients display the provenance of each address; an
implementation showing the opaque identifier without the relationship has
removed the benefit and kept the cost.

*Surgical severance.* Retiring one address ends one relationship.

*Harvested addresses deliver nothing.* Delivery requires a capability. Without
one, the most an attacker achieves is a quarantined request, and with
`acceptsNewRequests: false`, the default, not even that.

*Non-linkability.* Two issuers cannot determine they correspond with the same
person, which undermines cross-referencing by shared identifier.

### 3.3 Silent retirement

Two exit paths exist, and the distinction is the design point.

*Polite revocation* informs the counterparty, which stops sending, and the
relationship may resume. This is correct with a counterparty that behaves.

*Silent retirement* retires the address without notice. There is no unsubscribe
request, and no link confirming the recipient exists. The host answers as though
the address had never existed.

The second exists because, against a counterparty that ignores revocation, an
unsubscribe request is an interaction, and an interaction confirms the address
is live and read. This is why unsubscribe links are valuable to senders who
should not have them. Silent retirement is the difference between asking to
leave and ceasing to be there.

It composes with rotation: an identity may replace every address it holds and
carry forward only chosen relationships, accomplishing in one operation what
per-counterparty unsubscribing cannot.

The mechanism depends entirely on the absence of an enumeration oracle, which
Section 5.1 shows is harder to achieve than it appears.

## 4. Implementation

A reference implementation exists: a host in Rust, client libraries in Python
and JavaScript, and a browser client where keys are generated and held. It is
deployed on two independent domains, `imake.codes` and `dainner.app`, which
exercise federation between distinct hosts with distinct host keys.

The implementation covers the consent flow, per-relationship addressing,
selective rotation, the hash-chained history, compartmentalised projects with
sealed sections and cross-compartment milestones, attachments transferred
host-to-host, escalation with a blind host, and bridges to iCalendar and to a
REST gateway for issuers.

Signatures are produced and verified in three languages, which is how
canonicalisation divergence is caught: it is the part of a signature scheme that
fails silently, and cross-implementation verification is the only reliable test.

## 5. Findings

Each finding arose from the deployment, and each generalises.

### 5.1 Absence of an oracle is a property of an endpoint set

Several mechanisms depend on a retired, frozen, or nonexistent address being
indistinguishable in response. We implemented that on the delivery path
correctly.

An account undergoing deletion was frozen, and delivery to it correctly returned
`unknown-recipient`. The handle-proof endpoint, a separate public surface used
for key discovery, continued to serve that account's identity document.
Comparing the two responses revealed both that the account existed and that it
was being deleted, which is exactly the oracle the freeze exists to prevent.

The general statement: **absence of an oracle is a property of the set of public
endpoints, never of one endpoint in isolation.** One honest endpoint and one
distracted endpoint add up to a whole oracle. Systems with this requirement
should enumerate their public surfaces and test the property across all of them,
which is not the same as auditing the code path where the requirement was
written down.

### 5.2 A policy expressed on the transported object is the sender's policy

The recipient sets a status-sharing level per relationship: nothing, receipt,
milestones, or full. The transported task may also carry a level, intended as a
per-task narrowing.

Our first implementation consulted only the field on the task. That field
arrives inside the offer, so the sender writes it, and an issuer requesting
`full` received everything regardless of the recipient's configuration.

Correcting it introduced the mirror-image bug. Giving the task field a default
value made the minimum always win, so an issuer authorised for `milestones`
never learned that a task completed, because every task carried `receipt`
implicitly.

The general statement: **when a policy has two sources, the effective value must
be an explicit combination, and absence must be distinguishable from a default.**
A field defaulted at the object level silently overrides a policy set at the
relationship level, in the restrictive direction, which is harder to notice than
the permissive failure and equally wrong.

### 5.3 A host that signs unvalidated client envelopes is blank letterhead

Escalation requires releasing an envelope at a future time to a third party,
with a host that cannot read content. The client pre-composes the envelope; the
host stores it and releases it on schedule, applying its transport signature at
release.

We accepted the pre-composed envelope without validating it. Because the host
applies the domain's signature at release, a client could declare a `from`
belonging to another account at the same host, and at the far end the signature
verified against the declared sender. Local impersonation, with the host's own
key as the instrument.

The general statement: **wherever a system signs on behalf of a party, it must
validate what it signs at the moment of acceptance.** Deferred signing separates
the authorisation decision from the signing act, and anything unchecked in that
gap is signed later with full authority.

### 5.4 Advertised limits must be enforced limits

The discovery document advertised a 100 MB attachment limit. The web framework's
default body limit rejected at 2 MB. A conforming client, reading the advertised
limit, received an undiagnosable rejection for a legitimate 5 MB file.

The general statement is mundane and the failure mode is not: **a capability
document is a contract, and a value advertised in one place and enforced in
another will diverge.** They should derive from one constant. The failure
surfaces to users as an unexplained error and to implementers as a protocol
question, which is the worst place for it to surface.

### 5.5 A secondary observation on federation testing

Federation between two processes on one host is not a test of federation. Ours
passed for months while the client derived the API endpoint from the address
domain, an assumption that holds only when the address domain and the service
domain coincide. The first deployment using delegation, which is the recommended
configuration, returned 404 on the first call.

Protocols with a delegation mechanism should assume implementations will derive
rather than resolve, and should state the requirement normatively. We now do.

## 6. Security and Privacy Analysis

**What the structure removes.** Delivery to a harvested address, bulk
unsolicited delivery, cross-referencing by shared identifier, and the
unsubscribe confirmation.

**What remains.** A consented issuer that behaves badly, bounded but not
prevented by scope ceilings; compromise of the recipient's device; a malicious
host, which end-to-end encryption addresses for content and the hash chain makes
detectable for fabrication; traffic analysis; and social engineering.

**Metadata.** Even carrying ciphertext, a host observes which addresses
correspond, when, how often, and how large. Padding to size classes reduces the
size channel. Distributing addresses across hosts partitions the graph. We state
the bound exhaustively in the specification rather than claiming more than the
design provides.

**What we do not attempt.** Reputation, which requires a scoring party and
attaches to identifiers that per-relationship addressing deliberately fragments;
host-side content filtering, unavailable by construction under end-to-end
encryption; and proof-of-work on delivery, which taxes a utility company billing
a million customers at the same rate as an abuser.

## 7. Limitations and Future Work

**No quantitative evaluation.** We report that the system works, not how well.
An evaluation should measure federated delivery latency against email and
ActivityPub in comparable scenarios; the cost of the consent step at first
contact, in time and in abandonment; the size of the metadata a host observes
per exchange; and the throughput ceiling the consent edge imposes.

**No formal threat model.** Our threat analysis is prose. The properties should
be stated against a defined adversary, and the no-oracle property in particular
is a candidate for formal treatment, since Section 5.1 shows informal reasoning
about it is unreliable.

**End-to-end encryption is specified, not deployed.** The implementation runs in
assisted mode, where the host reads content. The specification requires hosts to
declare their mode and forbids describing assisted mode as confidential against
the operator, and the deployment honours that, but the confidentiality claims
await MLS integration.

**Adoption is unaddressed.** Bridges to iCalendar, email, and a REST gateway
exist as adoption paths. Whether they suffice is an empirical question we cannot
answer from one deployment.

## 8. Conclusion

Consent before delivery is implementable as a protocol primitive, and the
mechanisms that make it operational, per-relationship addressing and silent
retirement, follow from taking the primitive seriously rather than being
independent additions.

The findings suggest that the difficulty lies less in the primitive than in the
surrounding surfaces. Three of our four defects were failures to apply, at every
surface, a rule stated correctly at one surface. That pattern is likely general
to protocols with global properties, and it argues for stating such properties
as properties of sets, and for testing them that way.

## Availability

Specification, reference implementation, and the deployment described here are
public. The specification is available as an Internet-Draft set.

## Acknowledgements

To be completed.

## References

To be completed in the submission version. The related-work sources are
maintained with URLs in the project's prior-art survey.
