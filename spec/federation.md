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
host's `federationInbox`, discovered as specified in the core document. A POST
MAY carry more than one envelope, up to the destination's `maxCallsPerBatch`.

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

**Every other type, from an identity with no prior relationship, MUST receive the
same response as a nonexistent address:** `unknown-recipient`, permanent, identical
in code, body and timing. The `no-consent` code MUST be used only where the sender
already holds a capability issued for that address, meaning the relationship
existed and it is the scope or its state that fails, and never before the
capability has been compared.

While `no-consent` answered a sender who never had a relationship, it separated a
live address from a dead one at one request per guess. An oracle is a property of
the set of responses, and one careless endpoint annuls the equality every other
endpoint maintains.

**Replacing a pending knock MUST NOT silently widen scope.** A resend whose
declared scope differs from the pending request MUST reset the read state, and
clients MUST present it as new, saying what changed. Otherwise an issuer sends a
modest scope, waits for the person to read it and defer the decision, and swaps in
a wide scope before acceptance: what they accept is not what they read.

Project membership is an additional basis, specified in the compartments
document: a task belonging to a project circle is authorised by the recipient's
membership in that circle rather than by a relationship capability, because such
a task arrives addressed to the primary handle, which has no capability bound
to it. This is not a hole: the membership exists only because the recipient
accepted a `project.invite`, and that invitation passed through the ordinary
consent path.

# Contact Key Delivery

The `consent.grant` MUST carry the public contact key of that relationship, with a
proof binding it to the consent, and the counterparty MUST reject a relationship
group credential that does not match the key received there. Implementations MUST
NOT accept mere delivery by the routing host as proof of the other side's
identity.

Contact keys are deliberately absent from the public identity document, so this
directed delivery at grant time is the only path that authenticates one without
publishing it. Without it, the only thing attesting who sat on the other side of
the relationship group was the host, which is exactly the party the design means
to exclude: it would present its own key as the user's and sit permanently in the
middle of the channel carrying invoices, payment slips and test results.

# Retry

An accepted delivery is answered with `202` and `{ "accepted": ["<id>"] }`.
Acceptance means "persisted and queued for the recipient", not "read".

Delivery failures divide into two classes, and treating them alike wastes both
parties' resources.

**Permanent.** `no-consent`, `consent-revoked`, `unknown-recipient`,
`malformed`, `too-large`. The sending host MUST NOT retry.

**Transient.** `over-rate`, `try-later`, `server-error`, connection failures,
5xx. The sending host MUST retry on the schedule below.

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

The schedule's parameters are adjustable upward, never downward. Idempotency by
envelope `id` makes retries safe.

On returning from an outage longer than the retry window, a receiving host MUST
query the hosts it held current relationships with for envelopes discarded during
the period, and originating hosts MUST retain, for 30 days after giving up, the
list of envelope identifiers they abandoned per recipient. Clients MUST show the
recipient what was lost, with sender and time, even without the content.

Three days of unavailability are cheap to cause against a self-hoster, which is
this protocol's reference deployment, and happen unaided during a holiday. Without
reconciliation each issuer knew of the failure and the recipient never knew the
task had existed, which for a critical reminder is silent loss rather than delay.

On giving up, the host MUST inform the originating identity, as a local
`receipt.failed` envelope. A delivery that silently ceases is indistinguishable
from one that succeeded.

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

A push alternative, in which the origin sends a PUT alongside the delivery, MAY
be negotiated.

Binary crosses federation raw, without re-encoding.

# Fan-out and Groups

An envelope with multiple `to` addresses is delivered once per destination
host, grouping recipients served by the same host.

For MLS group messages (`group.commit`, `group.welcome`, `group.proposal`,
`group.message`), the sender sends the ciphertext once to its own host, which
replicates it to the hosts of every member; each host delivers to its own.
Hosts route from group to members using the routing list maintained by the
membership envelopes, and never read content.

# Anti-Abuse Minimum

A conforming host MUST implement:

- **Rate limiting by origin domain** for `consent.request`, not by IP address.
  In federation what matters is who is sending, and the address is commonly a
  shared proxy.
- **Rate limiting per pair**, issuer to recipient, for general traffic,
  answered with `over-rate` and `retryAfter`.
- **Explicit refusal.** Never silent discard. A sender that cannot distinguish a
  limit from a loss will either retry forever or give up wrongly.
- **Envelope size enforcement matching the advertised limit.** Advertising one
  value and enforcing another produces failures a conforming client cannot
  diagnose.
- **Deduplication by `dedupKey`** so that redelivery of a series updates rather
  than accumulates.
- **Abuse reporting.** A `report.abuse` envelope from the user to their own
  host, forwardable to the origin host; reputation semantics stay outside the
  core (pluggable).

Hosts MAY apply pre-delivery policy filters (shareable lists, in the spirit of
Matrix policy servers), but MUST NOT discard silently: an explicit error,
always.

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

# Open Questions

1. Direct host-to-host delivery of `urgency: critical` over a dedicated channel
   (a priority queue), or is the same queue with `urgencyHint` enough?
2. Live migration: the window in which two hosts serve the same identity, and
   the duplicate rules.
3. The format of `report.abuse` and the interoperable minimum of moderation.
