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

**Directory enumeration.** Scanning phone numbers to list who has an account
returns nothing: discovery of people is blind delivery, and no query returns an
address.

**The unsubscribe confirmation.** In email, an unsubscribe request confirms the
address is live and read, which is why it is valuable to a sender who should not
have it. EWP offers silent retirement, which communicates nothing.

# What Hosts Must Implement

**Hosts MUST enforce the consent edge** before accepting any envelope that is
not a `consent.request`.

**Hosts MUST respond indistinguishably** to a nonexistent, a retired and a
revoked address, as the identity document specifies. No validation oracle.

| Mechanism | Requirement |
|---|---|
| Rate limit on `consent.request` | By origin domain, not by IP address |
| Separate ceiling for established relationships | A single ceiling starves legitimate high-volume issuers |
| Rate limit on registration | Per origin, to bound account farming |
| Rate limit on session and RPC | Per identity |
| Explicit refusal | Always, with a code and `retryAfter` where applicable |
| Quarantine replacement | A new knock from the same issuer replaces the old, and a changed scope resets the read state |
| Urgency ceiling per relationship | Including between people, where it starts at `normal` and only the receiver raises it |
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
budget refusing. And if the verification is by substring, it is worse than
amplification: it becomes the oracle described in the consent document.

**Refusal MUST be explicit.** A host that discards silently leaves the sender
unable to distinguish a limit from a loss, and a sender that cannot distinguish
them will either retry forever or give up wrongly. It also removes the operator's
own ability to diagnose.

**Hosts MAY apply pre-delivery policy lists**, in the spirit of Matrix policy
servers, provided the refusal remains explicit and the user can inspect and
disable what their host filters.

# What Clients Must Implement

1. **Clients MUST NOT fetch a remote resource referenced in task content.**
   Images, fonts, stylesheets and any third-party URL in a description or
   payload MUST NOT be loaded automatically; everything displayed comes from a
   blob inside the envelope. This is the structural end of the tracking pixel:
   the issuer does not learn whether, when, how often, or from where a task was
   opened.
2. **Clients MUST show which address a task arrived through**, and to whom that
   address was given. This is what turns a leak into something actionable.
3. **Clients MUST mark first contact.** An issuer's first offer is displayed
   distinctly, and auto-accept rules MUST NOT apply to an issuer with no
   history.
4. **Clients MUST NOT allow urgency above `normal` on first contact**, whatever
   scope was granted.
5. **Every relationship has an urgency ceiling, including between people.** For
   an issuer it is the scope's `maxUrgency`; for a person-to-person
   relationship it is the `Relation` object's `maxUrgency`, which starts at
   `normal`. Hosts and clients MUST lower anything arriving above the ceiling
   to it, and raising it is NEVER the sender's decision.

   The ceiling used to live only in the consent scope, and a person-to-person
   relationship creates no consent: after first contact, `critical` was free
   forever. Since `critical` means maximum sound, cutting through silent mode,
   demanding acknowledgement and triggering escalation, that was a night
   harassment tool with the protocol's stamp on it, available to any ex-partner
   who had once been accepted.
6. **A comment opens no new door.** Whoever may not send a task may not
   comment; an issuer's comment counts against the rate limit and must fall
   within the granted purpose; a mention does not raise urgency above the
   scope.
7. **Clients MUST warn about confusables.** An issuer domain visually close to
   a known one (homographs, IDN, swapped letters) requires an explicit warning
   before any action.
8. **Clients MUST show the beneficiary of payment payloads** and warn when it
   diverges from the verified issuer.
9. **Clients MUST NOT execute an action without the user's confirmation**, and
   MUST show the destination of actions that open a link or make a request.
10. **Clients SHOULD offer the cut-off in one tap.** "Stop receiving from this
    company" belongs one gesture away, inside the task itself, performing
    revocation or silent retirement as the user chooses.

# What Issuers Must Do

1. **Declare the purpose.** Every offer carries the purpose it was sent under,
   within the purposes granted in the consent. Sending outside the purpose is a
   violation, not a grey area.
2. **Do not package advertising as a task**, by the test below.
3. **Honour revocation without friction.** Requiring a phone call, a form or a
   login in order to revoke is a compliance violation.
4. **Honour a rotation declaration**: a new address replaces the old one,
   without renegotiating the relationship.
5. **Respect the interval after revocation**: no asking for consent again
   within 30 days.
6. **Do not condition service on excessive scope.** Asking for more than the
   purpose requires, as a condition of doing business, is a dark pattern.

# What Is Not a Task

An operational rule, because "marketing" cannot be defined semantically:

> An offer whose completion produces no consequence for the user is not a task:
> it is advertising.

Paying a bill has a consequence (the supply is not cut). Confirming an
appointment has a consequence (the slot exists or it does not). "Check out our
promotion" has none: doing nothing changes nothing in the life of whoever
received it.

Issuers MUST NOT send offers that fail this test. Hosts MAY block a repeat
offender. Users retire the address, which is the most effective answer and asks
nobody's permission.

The protocol cannot detect this automatically and does not pretend to. What it
does is make every abuse attributable to a verified domain and cuttable in one
tap, inverting the economics: in email, persisting is free; here, persisting
costs the whole relationship, without warning and without appeal.

# Reporting

~~~
{
  "@type": "AbuseReport",
  "against": "acme-marketing.example",
  "alias": "k7f3a2q9@ework.example",
  "consent": "ework:consent/018f3c00-...",
  "kind": "out-of-purpose",
  "envelopes": ["018f4d2c-..."],
  "at": "2026-09-01T10:00:00Z"
}
~~~

`kind` is one of `spam`, `out-of-purpose`, `scam`, `urgency-abuse`,
`revocation-ignored`, `other`. The report is signed by the user and MUST NOT
carry task content beyond what the user chooses to attach.

Hosts MUST accept a signed `report.abuse` from their user and forward it to the
origin host. What the origin host does with it is its own policy; the protocol
guarantees the path, not the outcome.

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
per-relationship addressing deliberately fragments. Shareable policy lists are
not this: a host may apply them, but the refusal stays explicit and the user can
inspect and disable them.

**Content filtering.** In end-to-end encrypted mode the host cannot read content,
so host-side content filtering is not available by construction. Clients may
filter what they can read.

**Proof of work on delivery.** A cost on every delivery taxes legitimate
high-volume issuers, such as a utility company billing a million customers, at
exactly the same rate as an abuser, and the consent requirement already imposes
a cost the abuser cannot amortise. Whether a cost belongs somewhere narrower, on
consent requests from issuers with no history, is an open question below.

# Security Considerations

The anti-abuse posture depends on the absence of an oracle, specified in the
identity document. If a party can distinguish a nonexistent address from a
retired one, address lists become validatable, and the value of harvesting rises
again. Implementations MUST test for this explicitly, including for differences
in response time.

The ban on remote fetching closes the classic tracking vector and an
exfiltration vector as well: malicious content cannot phone home from the
client.

It also depends on hosts enforcing the consent edge. A host that delivers
without checking has removed the protection for its own users, and no other
party can compensate.

# Privacy Considerations

Rate-limit state is metadata: a host that records which domains knocked, and
when, holds a record of who attempted contact with whom. Hosts SHOULD retain it
for the minimum necessary for operation and anti-abuse, with 90 days as an upper
bound, and SHOULD place the record of discovery attempts under the user's
control.

The status policy of the consent document limits what the issuer knows, about
reading and progress, to what the user granted.

# Open Questions

1. **Pluggable reputation:** a minimal format for shareable policy lists,
   without reinventing email's opaque reputation. The likely shape: signed,
   with a declared scope, and always inspectable by the user.
2. **Sending cost:** is some cost (proof of work, a deposit) worth requiring
   for consent requests from issuers with no history, or is the consent itself
   already enough?
3. **Confusable detection:** which library or normative table (UTS 39), and how
   to avoid false positives with legitimately similar brands.
4. **Genuinely consented marketing:** should the person who actually wants
   promotions from a favourite shop have a dedicated purpose for that, or is
   that a case for another protocol?
5. Privacy-preserving aggregate reporting: how many people reported an issuer,
   without revealing who.
