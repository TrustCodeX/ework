---
docname: draft-wilhelm-ework-federation-00
title: "The e-work Protocol (EWP): Server-to-Server Federation"
abbrev: EWP Federation
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9110,RFC8446,RFC8032
---

This document specifies how e-work Protocol (EWP) hosts exchange envelopes:
locating the destination host, authenticated delivery to its inbox, the retry
schedule, receipts, the consent edge, blob transfer between hosts, and the
minimum anti-abuse behaviour a host must implement.

<!-- abstract -->

# Introduction

Federation is the property that makes EWP a protocol rather than a product. It
is also where abuse lives, which is why the consent edge specified in the
consent document is enforced here, at the receiving host, before anything
reaches a box.

## Conventions and Definitions

<!-- rfc2119 -->

# Delivery

Delivery is an HTTP POST {{RFC9110}} over TLS {{RFC8446}} to the destination
host's `federationInbox`, discovered as specified in the core document.

~~~
POST /ework/v1/inbox HTTP/2
Host: ework.example
Content-Type: application/json

{ "ewp": "0.1", "type": "task.offer", ... }
~~~

## Transport signature

Every envelope carries, in addition to the author's signature, a transport
signature by the sending host's key, the `hostKey` published in its discovery
document. The receiving host MUST verify that the transport signature matches
the `hostKey` published by the domain in `from`.

The two signatures answer different questions. The author signature says who
composed the object; the transport signature says which host handed it over.
Both matter: without the first, a host could forge content for its own users;
without the second, anyone able to reach the inbox could claim to be any host.

**A host that applies its transport signature to an envelope composed earlier by
a client MUST validate that envelope first.** This case arises with scheduled
release, specified in the urgency document. Without validation, the queue is
blank letterhead: a client declares a `from` belonging to another account at the
same host, the host stamps the domain's signature over it, and at the far end
the signature checks out against the declared sender. The minimum bindings are
that `from` MUST be an address of the requesting identity, `to` MUST be the
intended destination, and any capability MUST be the one issued for that
relationship.

# The Consent Edge

Before an envelope reaches a box, the receiving host MUST check, in this order:

1. The target address resolves and is `active`.
2. For envelope types other than `consent.request`, a grant exists for that
   address in state `granted`.
3. The envelope carries the capability issued for that grant.
4. The payload type is within the granted scope.
5. Volume and urgency ceilings are not exceeded.

The only envelope type accepted without a prior relationship is
`consent.request`, which is quarantined rather than delivered.

Project membership is an additional basis, specified in the compartments
document: a task belonging to a project circle is authorised by the recipient's
membership in that circle rather than by a relationship capability, because such
a task arrives addressed to the principal handle, which has no capability bound
to it. This is not a hole: the membership exists only because the recipient
accepted a `project.invite`, and that invitation passed through the ordinary
consent path.

# Retry

Delivery failures divide into two classes, and treating them alike wastes both
parties' resources.

**Permanent.** `no-consent`, `consent-revoked`, `unknown-recipient`,
`malformed`, `too-large`. The sending host MUST NOT retry.

**Transient.** `over-rate`, `unavailable`, connection failures, 5xx. The sending
host MUST retry on the schedule below.

| Attempt | Wait before it |
|---|---|
| 1 | immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |
| 6 | 8 hours |
| later | 24 hours |

The sending host MUST give up after 72 hours **measured from the first attempt**,
not after a count of attempts. Age is the correct bound because attempts are not
uniformly spaced: counting them makes the effective deadline depend on how many
retries happened to fit, which varies with the failure pattern.

On giving up, the host MUST inform the originating identity. A delivery that
silently ceases is indistinguishable from one that succeeded.

`over-rate` carries `retryAfter`, and the sending host SHOULD honour it in
preference to the schedule above.

# Receipts

`receipt.delivered` confirms that an envelope reached the destination box.
`receipt.processed` confirms that the recipient's client processed it.

Receipts are subject to the status policy of the consent document: a
relationship at level `none` produces no receipts beyond the edge-level HTTP
response.

# Blob Transfer

Envelopes reference blobs by hash. The destination host fetches those it does
not have:

~~~
GET /ework/v1/federation/blob/<hash> HTTP/2
X-Ework-Host: dainner.example
X-Ework-Ts: 2026-09-01T10:00:00Z
X-Ework-Sig: base64...
~~~

The signature covers a material binding three things: which blob, which host is
asking, and when. Without the hash, one signature would serve for any blob;
without the timestamp, it would serve forever. The serving host verifies against
the requesting domain's published `hostKey`, so forging the header is not enough:
the private key of that domain would also be required.

**The origin host MUST maintain, per blob, the list of destination hosts of
accepted envelopes that reference it, and MUST serve the blob only to those.**
A request from a host outside the list MUST receive the same response as for a
nonexistent blob.

The rule exists because of one sentence: **knowing a hash is not
authorisation**. The hash is derived from the content, so anyone who already has
the file can compute it, and an endpoint that serves by known hash hands another
party's attachment to whoever can guess. The reference implementation shipped
this endpoint open before the rule was written down, which is why the rule is
written down.

The receiving host MUST verify the hash of what arrives. This is the only thing
preventing the origin from substituting a different file for the one the task
references.

Binary crosses federation raw, without re-encoding.

# Anti-Abuse Minimum

A conforming host MUST implement:

- **Rate limiting by origin domain** for `consent.request`, not by IP address.
  In federation what matters is who is sending, and the address is commonly a
  shared proxy.
- **Explicit refusal.** Never silent discard. A sender that cannot distinguish a
  limit from a loss will either retry forever or give up wrongly.
- **Envelope size enforcement matching the advertised limit.** Advertising one
  value and enforcing another produces failures a conforming client cannot
  diagnose.
- **Deduplication by `dedupKey`** so that redelivery of a series updates rather
  than accumulates.

# Security Considerations

## Host as a network proxy

Any host feature that fetches a client-supplied URL lends the client the host's
network position, which typically reaches internal services, cloud metadata
endpoints, and LAN gateways that the client cannot reach. Implementations MUST
refuse loopback, private ranges, link-local, carrier-grade NAT, and reserved
metadata names before issuing any such request. This applies to the issuer
gateway webhook of the bridges document and to any other client-supplied URL.

## Identical responses

The rule of the identity document applies at every federation surface: retired,
revoked, frozen, and nonexistent addresses MUST be indistinguishable in
response code, body, and timing.

## What the receiving host learns

Even carrying ciphertext, a host observes which addresses correspond, when, how
often, and how large the envelopes are. Padding to size classes reduces the size
channel. The remaining channels are inherent to store-and-forward and are
documented rather than denied.

# Privacy Considerations

Per-relationship addressing means a host learns only the addresses it routes.
An identity distributing its addresses across several hosts partitions that
knowledge, at the cost of key and document management across hosts.
