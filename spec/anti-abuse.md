---
docname: draft-wilhelm-ework-anti-abuse-00
title: "The e-work Protocol (EWP): Anti-Abuse"
abbrev: EWP Anti-Abuse
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174
---

This document collects the anti-abuse posture of the e-work Protocol (EWP): what
the protocol prevents structurally, what it leaves to hosts, and what it
deliberately does not attempt.

The central claim is that consent before delivery, combined with
per-relationship addressing, removes the economic basis of bulk unsolicited
messaging rather than filtering it after arrival.

<!-- abstract -->

# Introduction

Anti-abuse in messaging systems is usually a filtering problem because delivery
came first. EWP changes the ordering, which changes what the attacker must do.

This document does not claim that abuse becomes impossible. It states which
attacks the structure removes, which it makes expensive, and which remain.

## Conventions and Definitions

<!-- rfc2119 -->

# What the Structure Removes

**Delivery to a harvested address.** An address obtained from a leak, a scrape,
or a purchase carries no capability. Delivery to it is refused at the edge. With
`acceptsNewRequests: false`, the default for relationship addresses, it does not
even accept a knock.

**Bulk unsolicited delivery.** Every delivery requires a grant, and every grant
requires an explicit act by the recipient. There is no volume at which this
becomes cheap.

**Cross-referencing by shared identifier.** Two issuers know two different
addresses, so they cannot determine they are talking to the same person without
out-of-band information.

**The unsubscribe confirmation.** In email, an unsubscribe request confirms the
address is live and read, which is why it is valuable to a sender who should not
have it. EWP offers silent retirement, which communicates nothing.

# What Hosts Must Implement

| Mechanism | Requirement |
|---|---|
| Rate limit on `consent.request` | By origin domain, not by IP address |
| Separate ceiling for established relationships | A single ceiling starves legitimate high-volume issuers |
| Rate limit on registration | Per origin, to bound account farming |
| Rate limit on session and RPC | Per identity |
| Explicit refusal | Always, with a code and `retryAfter` where applicable |
| Quarantine replacement | A new knock from the same issuer replaces the old |
| Size enforcement | Matching the advertised limit |

Rate limiting by origin domain rather than by IP matters in federation: the
address is commonly a shared proxy, and limiting by it punishes co-tenants while
leaving a determined sender free to rotate addresses.

**A single ceiling for known and unknown senders is a misconfiguration, not a
conservative default.** Measured on the reference implementation, a ceiling of
120 per minute per origin domain rejected 332 of 400 offers from an
already-consented relationship. The ceiling exists against parties that were not
invited; applying it to a party that was turns anti-abuse into an obstacle to the
use case the protocol exists to serve. Hosts SHOULD apply a substantially higher
ceiling to envelopes carrying a **verified** capability.

Verified is load-bearing. Selecting the higher ceiling on the mere presence of a
capability field is an amplification: an attacker adds a junk string and obtains
the larger budget, and while nothing is delivered, the host spends the larger
budget refusing.

**Refusal MUST be explicit.** A host that discards silently leaves the sender
unable to distinguish a limit from a loss, and a sender that cannot distinguish
them will either retry forever or give up wrongly. It also removes the operator's
own ability to diagnose.

# Attribution

Because each issuer holds a different address, unsolicited contact arriving at a
given address identifies which relationship leaked it. This is a property EWP
provides and email cannot.

Clients MUST show the provenance of each address, and MUST distinguish the
direction: an address issued by this party to a counterparty, versus an address
this party was given by a counterparty. Displaying the opaque identifier without
the relationship removes the benefit while keeping the cost.

# What Remains

Stating these plainly is part of the posture.

**A consented issuer that behaves badly.** Consent granted to a legitimate
issuer that later sends more than expected is not prevented by the structure.
Scope ceilings bound the volume and urgency, and revocation and silent
retirement end it, but the first bad message arrives.

**Compromise of the recipient's device.** No protocol survives this.

**A malicious host.** In assisted mode a host reads everything and can fabricate
content for its own users. End-to-end encryption removes the reading; the
history chain of the history document makes fabrication detectable by
counterparties, since their copies disagree.

**Traffic analysis.** The metadata list in the cryptography document is the
honest bound.

**Social engineering.** A recipient persuaded to grant consent has granted
consent. The protocol makes the grant scoped and revocable, which limits the
damage, and makes the record auditable, which helps afterwards.

# What the Protocol Does Not Attempt

**Reputation systems.** EWP defines no sender score. A reputation system requires
a party to compute and publish scores, which is a centralisation the protocol is
designed to avoid, and reputation attaches to identifiers, which
per-relationship addressing deliberately fragments.

**Content filtering.** In end-to-end encrypted mode the host cannot read content,
so host-side content filtering is not available by construction. Clients may
filter what they can read.

**Proof of work on delivery.** Considered and rejected: it taxes legitimate
high-volume issuers, such as a utility company billing a million customers, at
exactly the same rate as an abuser, and the consent requirement already imposes
a cost the abuser cannot amortise.

# Security Considerations

The anti-abuse posture depends on the absence of an oracle, specified in the
identity document. If a party can distinguish a nonexistent address from a
retired one, address lists become validatable, and the value of harvesting rises
again.

It also depends on hosts enforcing the consent edge. A host that delivers
without checking has removed the protection for its own users, and no other
party can compensate.

# Privacy Considerations

Rate-limit state is metadata: a host that records which domains knocked, and
when, holds a record of who attempted contact with whom. Hosts SHOULD retain it
for the minimum necessary for operation and anti-abuse, with 90 days as an upper
bound, and SHOULD place the record of discovery attempts under the user's
control.
