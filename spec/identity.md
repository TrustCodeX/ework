---
docname: draft-wilhelm-ework-identity-00
title: "The e-work Protocol (EWP): Identity, Addressing, and Rotation"
abbrev: EWP Identity
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8032,RFC8259,RFC8615
---

This document specifies identity and addressing for the e-work Protocol (EWP).
An EWP identity is a key pair under the sole control of its holder; a host never
possesses the private key. Human-readable handles are verifiable aliases, never
the identity itself.

The document also specifies per-relationship addressing, in which a recipient
issues a distinct opaque address to each counterparty, and the rotation protocol
that allows an identity to replace addresses while choosing, relationship by
relationship, who is carried forward and who is silently left behind.

<!-- abstract -->

# Introduction

Systems that identify people by an address controlled by a provider make
switching providers equivalent to changing identity. Systems that identify
people by a key held by the person avoid that, at the cost of key management.
EWP takes the second position and spends the rest of this document on making the
cost bearable.

## Conventions and Definitions

<!-- rfc2119 -->

# Key Layers

| Layer | Count | Storage | Purpose | Rotation |
|---|---|---|---|---|
| Root | 1 per identity | Offline, derived from the recovery kit | Being the identity; signing the other keys | Rare, with a contestation window |
| Device | 1 per device | On the device | Day-to-day signing | On loss or compromise |
| Contact | 1 or more per relationship | On the device | Deriving relationship addresses | Freely |

**The compromised key never authorises its own replacement.** If it sufficed,
whoever stole it would rotate first. Root replacement is authorised by the
recovery key, device replacement by the root or another verified device.

# Identity Document

The identity document is public and self-signed.

~~~
{
  "id": "did:key:z6MkCleia...",
  "seq": 12,
  "handles": ["cleia@ework.example"],
  "hosts": [ { "handle": "cleia@ework.example",
               "clientApi": "https://ework.example/ework/v1" } ],
  "devices": [ { "id": "dev-a1b2", "key": "ed25519:...",
                 "actorClass": "human", "addedAt": "2026-08-04T00:00:00Z" } ],
  "recoveryKeys": ["ed25519:..."],
  "previousKeys": [],
  "updatedAt": "2026-08-04T00:00:00Z",
  "proof": { "type": "ed25519-jcs-2026", "by": "did:key:z6MkCleia...", "sig": "..." }
}
~~~

Receivers MUST accept only documents whose `seq` exceeds the last one seen and
whose signature verifies against the root key, or against a valid rotation as
specified below.

`actorClass` on each device is load-bearing and specified in the autonomous
actors document: it is what allows a task to require human approval and have
that requirement be verifiable rather than advisory.

# Handles and Addresses

Two address types exist, with different purposes.

**Principal handle** (`cleia@ework.example`): public, stable, how people find a
person. Used for invitations between parties who know each other.

**Relationship address** (`k7f3a2q9@ework.example`): opaque, one per
counterparty, what organizations know.

## Bidirectional handle proof

A handle binds a name to a key, and the binding MUST be verifiable from both
directions.

1. **Domain to key:** `GET https://<domain>/.well-known/ework/handles/<local>`
   returns `{ "id": "did:key:z6MkCleia...", "seq": 12 }`.
2. **Key to domain:** the identity document lists the handle in `handles`.

A verifier MUST check both. Checking only the first lets a domain claim someone
else's key; checking only the second lets a key claim someone else's domain.

The handle proof lives at the **address domain**, not at the service domain.
This is deliberate: it is the one lookup that must work before anything is known
about where the service runs.

# Per-Relationship Addressing

**Default rule:** on granting consent to an organization, the client MUST
generate a new address for that relationship, unless the user explicitly chooses
to reuse an existing one.

~~~
{
  "@type": "ContactKey",
  "alias": "k7f3a2q9@ework.example",
  "key": "ed25519:...",
  "host": "ework.example",
  "peer": "utility.example",
  "consent": "ework:consent/019f3c00-...",
  "createdAt": "2026-08-04T11:58:00Z",
  "state": "active",
  "acceptsNewRequests": false,
  "boundBy": { "sig": "signed by the root" }
}
~~~

Four properties follow, and they are the reason the mechanism exists.

**Leak attribution.** Each issuer knows a different address. Spam arriving at
`k7f3a2q9` proves who leaked or sold the data. Clients MUST show where an
address came from, because the property is worthless if the user cannot connect
address to relationship.

**Surgical severance.** Retiring one address ends one relationship without
touching the others.

**A scraped address delivers nothing.** Delivery requires a valid capability for
that address. Without one, the most an attacker achieves is a
`consent.request` in quarantine, and with `acceptsNewRequests: false`, the
default, not even that.

**Non-linkability.** Two issuers cannot, by default, discover that they are
talking to the same person. This is what undermines cross-referencing by shared
identifier.

Registration is private: the host keeps the address list in private storage,
never in a public document. A host learns the addresses it routes and only
those; distributing addresses across hosts partitions that knowledge as well.

People invited personally use the principal handle by default, because there
linkability is desired: you want your sister to know it is you.

# Rotation

## Authority

| Rotating | Authorised by | Immediate effect |
|---|---|---|
| Contact key | Root or verified device | Old address dies; relationships migrate by choice |
| Device key | Root or another verified device | Removal from all encryption groups |
| Root key | Recovery key, with contestation window | Continuity chain in `previousKeys` |

## Rotation declaration

~~~
{
  "@type": "KeyRotation",
  "subject": "k7f3a2q9@ework.example",
  "oldKey": "ed25519:...",
  "newKey": "ed25519:...",
  "newAlias": "k9b1c4x2@ework.example",
  "reason": "compromise",
  "at": "2026-09-01T10:00:00Z",
  "graceUntil": null,
  "proofs": [ { "by": "old key or root" }, { "by": "did:key:z6MkCleia..." } ]
}
~~~

`reason` is one of `compromise`, `hygiene`, `migration`. With `compromise`,
`graceUntil` MUST be null: the old key dies immediately, because keeping the
address alive for convenience keeps alive the access of whoever stole it. With
`hygiene` or `migration`, the host SHOULD accept both addresses for up to 30
days so that envelopes in flight are not lost.

The continuity proof is what prevents hijacking: without a signature from the
old key or the root, nobody redirects another party's address.

## Selective migration

The declaration goes to **always** the identity's own hosts, and **only the
counterparties the user chooses**.

Those who do not receive it are left with a dead address. There is no
negotiation, no unsubscribe request, and no link confirming that the recipient
exists.

This is the mechanism that gives the recipient leverage. In one operation, an
identity can replace every address it holds and carry forward only the
relationships it wants, leaving the rest with identifiers that resolve to
nothing. Clients MUST make the set of those left behind explicit before
executing, by name, because the operation is not reversible from the recipient's
side: restoring a relationship requires the counterparty to knock again.

## No oracle

A retired address enters state `retired`. The host MUST respond to envelopes
addressed to it exactly as it would for an address that never existed:
`unknown-recipient`, a permanent error. The host MUST NOT distinguish, in its
response, between retired, nonexistent, and revoked.

The rule applies to **every public surface**, not only to delivery. In
particular, the handle proof of this document MUST return `unknown-recipient`
for a retired, frozen, or destroyed address. A host that refuses delivery while
continuing to publish the identity document delivers the same information by a
different door, and anyone wishing to validate a list compares the two responses
instead of one. Absence of an oracle is a property of the set of endpoints,
never of one in isolation.

# Multiple Hosts

An identity MAY register at more than one host. The identity document lists all
of them, and delivery to a handle at any of them reaches the same identity.

The property this buys is that provider migration does not change identity: the
key stays, the document is updated, and correspondents follow the document
rather than the address.

# Security Considerations

Keeping the root key out of the hot path limits the damage from a compromised
device. The requirement that replacement be authorised by a superior key, never
by the suspect key itself, prevents an attacker from rotating first. The
contestation window limits malicious rotation via a stolen recovery key.

The bidirectional handle proof prevents both a domain claiming another party's
key and a key claiming another party's domain.

The absence of an oracle is a security decision, not a usability one. Any
differentiation in response would be turned into a list-validation tool.

# Privacy Considerations

Per-relationship addressing is the principal privacy mechanism of the protocol
and its cost is real: the user holds more identifiers than a single handle, and
clients must present them in a way that keeps the mapping legible. A client that
displays an opaque address without saying which relationship it belongs to has
removed the benefit while keeping the cost.

# IANA Considerations

This document requests registration of the well-known URI suffix
`ework/handles` in the "Well-Known URIs" registry {{RFC8615}}, with change
controller IETF and this document as reference.
