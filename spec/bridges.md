---
docname: draft-wilhelm-ework-bridges-00
title: "The e-work Protocol (EWP): Bridges to Existing Systems"
abbrev: EWP Bridges
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC5545,RFC4791,RFC7208,RFC6376,RFC7489
---

This document specifies bridges between the e-work Protocol (EWP) and systems
that do not speak it: a CalDAV gateway exposing tasks as VTODOs, an email
fallback for recipients with no EWP account, and a REST gateway allowing an
issuer's existing systems to originate tasks with one HTTP call.

Bridges are adoption paths, not parity. Each one loses something, and this
document states what.

<!-- abstract -->

# Introduction

A federated protocol with no users is a specification. Bridges exist because
adoption does not begin with everyone switching clients; it begins with the task
appearing where the person already looks, and with the issuer integrating in an
afternoon rather than a quarter.

Every bridge crosses a boundary where guarantees stop. A bridge MUST NOT present
itself as carrying properties it cannot carry.

## Conventions and Definitions

<!-- rfc2119 -->

# CalDAV Gateway

A host, or a daemon running on the user's own machine, MAY expose boxes as
CalDAV {{RFC4791}} collections of VTODOs {{RFC5545}}.

## Mapping

| EWP | iCalendar |
|---|---|
| `title` | SUMMARY |
| `description` | DESCRIPTION |
| `due` plus `timeZone` | DUE |
| `progress` | STATUS |
| `percentComplete` | PERCENT-COMPLETE |
| `priority` | PRIORITY |
| `relatedTo` | RELATED-TO with RELTYPE |
| recurrence | RRULE |

`progress` maps to STATUS as `needs-action`, `in-process`, `completed`,
`cancelled`. **`failed` has no equivalent** and exports as CANCELLED plus
`X-EWORK-STATUS:failed`, because iCalendar does not distinguish "I cancelled it"
from "I tried and it did not work", and losing that distinction in a round trip
would be a silent data loss.

Implementations MUST fold lines at 75 octets and MUST NOT split a multi-byte
character. A client receiving an over-long line commonly discards the whole
component, and the symptom reaches the user as "the task did not appear".

RELTYPE is exported correctly and most clients ignore it. **A bridge MUST NOT
claim better support than actually exists.**

## What does not cross

Negotiation, consent, urgency, typed payloads, and actions do not cross. VTODO
has nowhere to put them, and inventing X-properties no client reads would be
decoration.

## Encryption rule

The bridge exists only (a) as a process on the client side, holding the user's
keys, or (b) over a collection in assisted mode. **A host offering CalDAV over
an end-to-end encrypted box without (b) is non-conforming**, because it would
have to decrypt to serve.

# Email Fallback

For a recipient with no EWP account, discovery having failed, the issuer MAY
send email carrying a readable body, an `.ics` attachment with the VTODO, and an
acceptance link hosted by the issuer.

The link offers two paths: accept through an EWP provider, or answer basically,
accept or decline, without an account. The basic answer maps back to an entry
with action `accept` or `decline` on the issuer's side.

Without an account there is no synchronisation and no continuing status. **The
bridge is adoption bait, not parity**, and presenting it otherwise sets an
expectation the mechanism cannot meet.

## Anti-phishing

The email MUST originate from the issuer's verified domain, with SPF
{{RFC7208}}, DKIM {{RFC6376}}, and DMARC {{RFC7489}} aligned, and the acceptance
link MUST be on that same domain.

A protocol whose adoption path is "click the link in this email" is training
recipients for exactly the attack it must resist, so the alignment requirement
is not optional.

# Issuer REST Gateway

A service, hosted by the issuer or by a provider, that speaks simple REST inward
and federation outward.

~~~
POST /v1/offers          { to, title, due, dedupKey, payloads, attachments }
GET  /v1/offers/{id}     -> { progress, negotiation }
POST /v1/webhooks        { url, events }
~~~

The design goal is stated as a criterion: **integration in one afternoon, with
one HTTP call to create the offer and one webhook to learn the outcome.** If the
simple case needs more than that, the bridge has failed its purpose.

## Authentication and identity

An API key authenticates the issuer's system against the gateway. **The key is
not the identity.** The gateway holds the organization's identity and signs
envelopes with it, so leaking an API key does not leak the identity: the key is
revoked and the organization remains the same party in the federation.

Issuers SHOULD create one key per system, so that revocation is surgical.

## The status filter runs at the gateway

Webhooks deliver only what each recipient's policy permits. **The gateway MUST
apply the filter; the issuer's system MUST NOT be trusted to do it**, which
would be asking the interested party to police itself.

The effective level is the minimum of the relationship policy and any task-level
override, as specified in the consent document.

## Webhook destinations

**The gateway MUST validate the webhook URL before any delivery.** The gateway
occupies a network position its client does not, typically reaching internal
services, cloud metadata endpoints, and LAN gateways. A free-form `url` field
lends that position to whoever holds an API key, and the behaviour of the
response is already the leak. Implementations MUST refuse loopback, private
ranges, link-local, carrier-grade NAT, and reserved metadata names.

This requirement applies to any client-supplied URL, not only to webhooks.

## Retry

Webhook delivery failures are not retried by the gateway. The status is already
in the issuer's box through federation, and the webhook is a convenience;
retrying here would create a second delivery queue with different semantics from
the first.

# Importing From Legacy Sources

Implementations MAY import from existing task and calendar systems. An imported
task MUST be marked as imported and MUST NOT carry an `ework:origin` claiming a
consent relationship that never existed.

Fabricating provenance to make imported data look native destroys the property
that makes `ework:origin` worth reading.

# Security Considerations

Every bridge is a place where the protocol's guarantees stop, and the boundary
must be visible to the user. A task arriving through the email fallback has not
passed the consent edge; a collection served over CalDAV is readable by whatever
holds the CalDAV credential; the REST gateway holds an identity that can sign
for the organization.

Bridges are also the most likely place for an implementation to accidentally
grant more than intended, because they translate between models with different
authorisation shapes. The SSRF requirement above is one instance of a general
rule: at a bridge, validate what crosses, in both directions.

# Privacy Considerations

A bridge necessarily sees plaintext. A CalDAV gateway reads tasks, an email
bridge reads the content it renders, and a REST gateway reads what it sends.
Users MUST be told which bridges are active on their box, since a bridge silently
enabled is indistinguishable, from the user's side, from an encryption mode
downgrade.
