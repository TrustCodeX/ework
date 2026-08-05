---
docname: draft-wilhelm-ework-core-00
title: "The e-work Protocol (EWP): Core"
abbrev: EWP Core
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8259,RFC8785,RFC8446,RFC9110,RFC8615,RFC2782,RFC6335,RFC8032,RFC9562
---

This document specifies the core of the e-work Protocol (EWP), an open,
federated protocol for exchanging actionable tasks between people and
organizations that do not share a provider.

EWP addresses a gap left by existing protocols. Electronic mail carries
unstructured text and imposes no consent requirement, so the recipient absorbs
the cost of every unsolicited message. Calendar protocols model events well but
model neither delegation across organizational boundaries nor the negotiation
that a task requires. Proprietary task systems solve the problem inside a single
vendor and stop at its edge.

The core defined here covers the envelope, the two-layer host discovery
mechanism, the transport binding, canonicalisation and signing, and the error
vocabulary. Companion documents define the data model, identity, consent,
federation, and the remaining layers.

<!-- abstract -->

# Introduction

A person receives a utility bill, a medical appointment reminder, a request to
review a contract, and a delivery notice. Each of these is a task: something
that has a deadline, an issuer, a state, and a resolution. Today each arrives as
prose in an inbox, and the recipient performs the structuring by hand: reading,
extracting the date, transcribing it into a calendar, and remembering to act.

The structure exists at the origin. The utility company knows the amount, the
due date, and the payment reference as fields in a database. It discards that
structure to render an email, and the recipient reconstructs it. EWP preserves
the structure end to end.

## Design constraints

The protocol takes four positions, and each has a cost that this document states
plainly.

**Federated, not centralised.** Any party may run a host. Addresses look like
`name@domain`, and the domain locates the host, in the same way that MX records
locate a mail server. The cost is that federation makes abuse harder to contain
than in a system with a single operator, which is why consent is a protocol
primitive rather than an application feature.

**Consent before delivery.** An issuer cannot place a task in a recipient's box
without a prior grant from that recipient. The cost is friction on first
contact. The benefit is that unsolicited delivery has no path, rather than being
filtered after the fact.

**One address per relationship.** A recipient who grants consent issues an
opaque address usable only by that counterparty. The cost is that recipients
hold more addresses than a single handle. The benefit is leak attribution and
surgical severance, described in the identity document.

**End-to-end encryption as the target state.** Hosts are expected to carry
ciphertext they cannot read. This document specifies the assisted mode, in which
the host does read content, because interoperability of the envelope and the
discovery layer can be established before group encryption is in place. A host
operating in assisted mode MUST advertise that fact, so that no client can
mistake one for the other.

## Conventions and Definitions

<!-- rfc2119 -->

The following terms are used throughout the EWP document set.

**Host.** A server that stores boxes for identities and exchanges envelopes with
other hosts. Analogous to a mail server.

**Box.** The set of tasks, entries, and consent records belonging to one
identity on one host.

**Identity.** A key pair under the sole control of a person or organization,
identified by a `did:key` value. A host never possesses an identity's private
key.

**Handle.** A human-readable address for an identity, of the form
`local@domain`. A handle is an alias, verifiable in both directions, and never
the identity itself.

**Envelope.** The unit of transfer between hosts. Carries a type, a sender, one
or more recipients, and a typed body.

**Issuer.** A party that offers a task to another party.

# Protocol Overview

An EWP exchange has three phases, and the first is not optional.

~~~
   Issuer                     Recipient host              Recipient
     |                              |                          |
     |  1. consent.request          |                          |
     |----------------------------->|                          |
     |                              |  quarantine, no alert    |
     |                              |------------------------->|
     |                              |                          |
     |  2. consent.grant            |    grants, address is    |
     |<-----------------------------|    created here          |
     |                              |<-------------------------|
     |                              |                          |
     |  3. task.offer               |                          |
     |----------------------------->|------------------------->|
     |                              |                          |
     |  4. task.entry (filtered)    |                          |
     |<-----------------------------|<-------------------------|
~~~

Phase 1 and 2 happen once per relationship. Phase 3 and 4 repeat for the life of
that relationship. An envelope that arrives without a valid capability for the
address it targets is refused at the edge, before it reaches the box.

# Envelope

Every transfer between hosts is an envelope. Envelopes are JSON {{RFC8259}}
objects.

~~~
{
  "ewp": "0.1",
  "id": "019fd1d5-e84c-76a0-9961-f1ccbcead44c",
  "type": "task.offer",
  "from": "billing@utility.example",
  "to": ["k7f3a2q9@ework.example"],
  "sentAt": "2026-09-01T10:00:00Z",
  "capability": "cap_019fd047-0ca9-72a0",
  "dedupKey": "energy-2026-09",
  "body": { "task": { } },
  "proofs": [ { "type": "ed25519-jcs-2026", "by": "did:key:z6Mk...",
                "key": "ed25519:...", "sig": "..." } ]
}
~~~

## Fields

| Field | Type | Required | Meaning |
|---|---|---|---|
| `ewp` | string | yes | Protocol version of this envelope |
| `id` | string | yes | UUIDv7 {{RFC9562}}, unique per envelope |
| `type` | string | yes | Envelope type, see below |
| `from` | string | yes | Sender address |
| `to` | array of string | yes | Recipient addresses |
| `sentAt` | string | yes | RFC 3339 timestamp |
| `capability` | string | no | Credential bound to the target address |
| `dedupKey` | string | no | Series identifier for idempotent redelivery |
| `refs` | array of string | no | Envelope identifiers this one responds to |
| `urgencyHint` | string | no | Coarse hint, see the urgency document |
| `body` | object | yes | Typed payload |
| `proofs` | array of Proof | yes | One or more signatures |

Receivers MUST reject an envelope whose `ewp` major version they do not
implement. Receivers MUST reject an envelope without at least one valid proof.

## Envelope types

| Type | Direction | Body |
|---|---|---|
| `consent.request` | issuer to person | Scope being requested |
| `consent.grant` | person to issuer | Consent object and capability |
| `consent.revoke` | person to issuer | Consent identifier |
| `task.offer` | issuer to person | Proposed task |
| `task.entry` | author to counterparties | Message and optional action |
| `task.entry.retract` | author to counterparties | Retraction of an entry without an action |
| `project.invite` | member to member | Project, circles, role |
| `project.join` | member to members | Membership acceptance |
| `project.leave` | member to members | Membership termination |
| `key.rotation` | identity to selected peers | Address rotation declaration |
| `receipt.delivered` | host to host | Delivery receipt |

Accept, complete, and decline are deliberately **not** envelope types. They are
values of `action` inside `task.entry`. The choice has a privacy consequence and
not merely an organisational one: the type travels in clear text in the header,
so separate types would tell every host on the path whether a task was
completed, declined, or postponed. With a single type, the host observes that
activity occurred and nothing more.

New types are registered extensions. Third parties MUST use a reverse-domain
prefix, for example `com.example.type`.

## Size limits

A host MUST advertise the maximum envelope size it accepts and MUST enforce the
value it advertises. Advertising one limit and enforcing another produces
rejections that a conforming client cannot diagnose: the implementation that
originated this protocol advertised 100 MB for binary attachments while a
framework default rejected at 2 MB, and the failure surfaced to users as an
unexplained error.

Bodies above the advertised envelope limit MUST use a blob reference rather
than inline content. The recommended envelope body limit is 256 KiB.

# Canonicalisation and Signing

Signatures cover the canonical form of the object with proof fields removed.
Canonicalisation follows JCS {{RFC8785}}: keys sorted lexicographically, no
insignificant whitespace, and the JSON number and string rules of that document.

The signature algorithm is Ed25519 {{RFC8032}}. A proof has the form:

~~~
{
  "type": "ed25519-jcs-2026",
  "by":   "did:key:z6MkCleia...",
  "key":  "ed25519:base64...",
  "sig":  "base64..."
}
~~~

`by` identifies the signer. `key` carries the public key used, so that a
verifier can check the signature before resolving the signer's identity
document. Implementations MUST verify that `key` belongs to `by` according to
the identity document, and MUST NOT trust `key` on its own.

Implementations MUST produce byte-identical canonical forms. Independent
implementations verifying each other's signatures is the only practical test of
this property, and the reference implementation exists in three languages for
that reason.

# Host Discovery

Discovery has two layers with distinct roles: **DNS locates, HTTPS describes**.
The separation exists because the two questions differ ("where is the server for
`example.com`?" and "what does it support and what is its key?") and have
different consumers.

## Location: SRV record

A domain participating in EWP SHOULD publish:

~~~
_ework._tcp.example.com.  3600  IN  SRV  0 1 443 ework.example.com.
~~~

This plays the role MX plays for mail, and solves a problem that `.well-known`
alone creates: **announcing a task host should not require modifying the
domain's website**. Many domains point their apex at a landing page, a CMS, or a
third-party service where serving a new path is expensive or impossible. A DNS
record has no such coupling. Priority and weight provide failover and load
distribution per {{RFC2782}} without inventing anything.

**A limitation measured in the field, which this specification must admit:** SRV
does not compose with reverse proxies that hide the origin. At least one major
CDN, when the SRV target is a proxied record, rewrites the target to an internal
name that resolves only inside that provider's network, and a peer following the
SRV cannot connect. Because a broken SRV is worse than no SRV, in that it leads
the client to a dead path instead of letting it fall through to the next layer,
a domain in that situation MUST simply not publish SRV and MUST rely on the
HTTPS layer.

## Description: document at the host

Having resolved a target, the discovering party fetches
`GET https://<target>/.well-known/ework` {{RFC8615}}:

~~~
{
  "ewp": "0.1",
  "versions": ["0.1"],
  "capabilities": ["urn:ework:core", "urn:ework:e2ee"],
  "addressDomain": "example.com",
  "clientApi": "https://ework.example.com/ework/v1",
  "federationInbox": "https://ework.example.com/ework/v1/inbox",
  "web": "https://ework.example.com",
  "hostKey": "ed25519:mB5c...",
  "securityContact": "security@example.com",
  "delegatedFrom": "example.com",
  "software": { "name": "ework-host", "version": "0.1.28" }
}
~~~

`addressDomain` is the domain that appears in addresses. `clientApi` and
`federationInbox` are where the service actually answers, and the two MAY differ
from `addressDomain`. That difference is **delegation**, and it is the same
mechanism by which an MX record points elsewhere: addresses stay at
`person@example.com` while the service runs at `ework.example.com`.

`software` is optional and identifies the implementation. Publishing it has a
real cost, in that it tells anyone looking which known defect to try, and a
concrete benefit: in a protocol whose premise is independent implementations
interoperating, diagnosing incompatibility without requesting access to
another party's server depends on it, and an operator running more than one
host can verify externally that all of them run the same build. Implementations
SHOULD publish it and MUST allow the operator to suppress it.

## Resolution order

To discover the host for `name@domain`, in order:

1. **SRV** `_ework._tcp.<domain>`. If answered, the target is the highest
   priority entry, and the document of the previous section is fetched there.
2. **`.well-known` at the domain itself**, `https://<domain>/.well-known/ework`.
   This is the path for parties that cannot publish SRV, and the only one
   available to clients running in a browser, since JavaScript cannot perform
   SRV lookups.
3. With neither, the domain does not participate in EWP.

Implementations MUST support path 2 and SHOULD support path 1.

**Clients MUST use the addresses advertised in `clientApi`, `federationInbox`,
and `web`, and MUST NOT derive them from the address domain.** This appears
obvious when written down, and is precisely the error the reference
implementation made: constructing `https://<domain>/ework/v1` works in
development, where the address domain and the service domain coincide, and
breaks against the first host using delegation, which is the recommended
deployment. The sole exception is the handle proof of the identity document,
which by definition lives at the address domain.

## Trust

Authenticity does not come from DNS. It comes from TLS {{RFC8446}} at the
target, plus the transport signature with `hostKey`, plus the author signature
inside the object. A poisoned SRV leads to a host that cannot produce a valid
signature for the sender it claims to represent, so the attack fails at
validation rather than at resolution. DNSSEC improves the situation and is NOT
a requirement.

Results SHOULD be cached respecting the SRV TTL and, for the document, a
reasonable TTL, with 24 hours RECOMMENDED.

# Transport

EWP binds to HTTP {{RFC9110}} over TLS {{RFC8446}}. Two endpoint families exist:
the client API, used by a client to talk to its own host, and the federation
inbox, used by a host to deliver to another host.

Binary content does not travel base64-encoded inside JSON. Attachments are
addressed by content hash and transferred as raw octets. The reason is
arithmetic: base64 inflates by one third, and a protocol whose reference
scenario includes contracts, invoices, and photographs cannot afford that on
every hop.

# Errors

Errors are JSON objects with a stable code, a human-readable message, and an
explicit indication of whether retrying can succeed.

~~~
{
  "error": "over-rate",
  "message": "rate limit reached; retry in 3487s",
  "retryable": true,
  "retryAfter": 3487
}
~~~

| Code | Retryable | Meaning |
|---|---|---|
| `unknown-recipient` | no | Address does not resolve |
| `no-consent` | no | No valid grant for this address |
| `consent-revoked` | no | Grant existed and was revoked |
| `malformed` | no | Envelope failed validation |
| `too-large` | no | Above the advertised limit |
| `over-rate` | yes | Rate limit, with `retryAfter` |
| `unavailable` | yes | Transient host failure |

A host MUST NOT discard an envelope silently. Refusal is always explicit, with a
code. Silent discard is indistinguishable from loss, and a sender that cannot
tell the two apart will retry forever or give up wrongly.

# Security Considerations

## No oracle

Several EWP mechanisms depend on a single property: **an address that was
retired, frozen, or never existed MUST produce identical responses**. Identical
means the same status code, the same body, and comparable timing.

This property belongs to the **set** of public endpoints, never to one endpoint
in isolation. The reference implementation leaked here: delivery correctly
refused for an account undergoing deletion, while the handle proof endpoint
continued to publish that account's identity document. Comparing the two
responses revealed that the account existed and was on its way out, which is
exactly the oracle the mechanism exists to prevent. One honest endpoint and one
distracted endpoint add up to a whole oracle.

## Host as a network proxy

Any field in which a client says "go fetch this" or "notify that" lends the
client the host's network position, which typically reaches things the client
cannot: internal services, cloud metadata endpoints, LAN gateways. This is
server-side request forgery, and in EWP it appears in the issuer gateway webhook
and in any client-supplied URL. Implementations MUST refuse loopback, private
ranges, link-local, carrier-grade NAT, and reserved metadata names before
issuing any request.

## Signing on behalf of clients

Where a host applies its transport signature to an envelope the client composed
earlier, the host MUST validate that envelope before accepting it. Without
validation the queue becomes blank letterhead: a client declares a `from`
belonging to another account on the same host, and the host stamps the domain's
signature over it, after which the signature checks out against the declared
sender at the far end. Minimum bindings are that `from` MUST be an address of
the requesting identity, `to` MUST be the intended destination, and the
capability MUST be the one issued for that relationship.

## Metadata visible to hosts

In assisted mode the host reads content. In end-to-end encrypted mode it does
not, but it still observes: which addresses exchange envelopes, when, how large,
and how often. Padding to size classes reduces the size channel. The remaining
channels are inherent to any store-and-forward system and are documented rather
than denied.

# IANA Considerations

This document requests three registrations.

## Well-Known URI

IANA is requested to register the following in the "Well-Known URIs" registry
{{RFC8615}}:

| Field | Value |
|---|---|
| URI suffix | `ework` |
| Change controller | IETF |
| Reference | This document |
| Status | permanent |

## Service Name

IANA is requested to register the following in the "Service Name and Transport
Protocol Port Number Registry" {{RFC6335}}:

| Field | Value |
|---|---|
| Service Name | `ework` |
| Transport Protocol | tcp |
| Description | e-work Protocol host |
| Reference | This document |

No port number is requested. EWP runs over HTTPS on the port carried in the SRV
record, with 443 expected in practice.

## Media Type

IANA is requested to register `application/ework+json` in the "Media Types"
registry, with encoding considerations of binary, security considerations as in
this document, and this document as the reference.

# Implementation Status

This section is to be removed before publication as an RFC.

A reference implementation exists in Rust, with client libraries in Python and
JavaScript, and is deployed on two independent domains that exercise federation
between them. Several statements in this document, in particular the SRV
limitation, the discovery derivation error, the advertised-versus-enforced size
limit, and the oracle leak across endpoints, are recorded because that
deployment produced them.
