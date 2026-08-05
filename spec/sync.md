---
docname: draft-wilhelm-ework-sync-00
title: "The e-work Protocol (EWP): Client-Server Synchronisation"
abbrev: EWP Sync
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9110,RFC8259
---

This document specifies how an e-work Protocol (EWP) client talks to its own
host: session establishment by challenge and response against a device key,
batched method calls, per-type state strings for incremental synchronisation,
binary blob upload and download, and the event channel.

<!-- abstract -->

# Introduction

The client-server interface is deliberately unremarkable. It borrows the batched
call and state-string model from JMAP, because that model already solves the
problem of a client keeping several object types in sync over a high-latency
link without polling each one.

The one place EWP departs from convention is authentication: there is no
password. The client holds a device key, and the session is established by
signing a challenge.

## Conventions and Definitions

<!-- rfc2119 -->

# Session

## Registration

A client registers by presenting a self-signed identity document. The host
verifies the signature and stores the document. **The host never receives a
private key**, and a host implementation that accepts one is not conforming.

## Challenge and response

~~~
POST /ework/v1/challenge      ->  { "nonce": "019fd..." }
POST /ework/v1/session
     { "did": "did:key:z6Mk...", "device": "dev-a1b2",
       "nonce": "019fd...", "sig": "base64..." }
                              ->  { "token": "ews_...", "expires": "..." }
~~~

The host MUST verify the signature against the device key recorded in the
identity document, and MUST reject a nonce it did not issue or has already
consumed.

Session tokens are bearer credentials, and hosts MUST rate limit both endpoints.

# Batched Calls

~~~
POST /ework/v1/rpc
Authorization: Bearer ews_...

{ "calls": [
    ["Task/get", {}, "t"],
    ["Consent/list", {}, "c"]
] }
~~~

Each call is a triple of method name, arguments, and a client-assigned
identifier echoed in the response. Responses come back in the same shape.

A call that fails returns an error object in place of its result. **A failed
call MUST NOT abort the batch**: a client fetching six things and losing all of
them because one was rate-limited would have to serialise every request, which
defeats the purpose of batching.

## Method families

| Family | Purpose |
|---|---|
| `Session/*` | Session state and limits |
| `Task/*` | Tasks: read, create, relate, arm |
| `Entry/*` | History entries |
| `Consent/*` | Grants, revocation, knocking |
| `Address/*` | Relationship addresses, rotation |
| `Project/*` | Projects and compartments |
| `Blob/*` | Attachment access tickets |
| `Account/*` | Export, deletion |

# State Strings

Every object type carries a monotonic state string per box.

~~~
"state": { "Task": 412, "Entry": 1580, "Consent": 33, "Quarantine": 7 }
~~~

A client holding a prior state calls `Task/changes` with it and receives what
changed, rather than refetching the collection. A host that cannot compute the
delta from a given state MUST say so explicitly, so the client falls back to a
full fetch rather than silently missing changes.

# Events

The host offers a server-sent event stream so that a client learns of changes
without polling.

~~~
POST /ework/v1/rpc   ["Events/ticket", {}, "e"]  ->  { "url": "/ework/v1/events?ticket=..." }
GET  /ework/v1/events?ticket=...
~~~

**Events carry only the state strings, never content.** A client that receives
"Task changed to 413" fetches what changed over the authenticated channel. This
keeps the event channel free of anything worth intercepting and makes it safe to
open from contexts where a bearer token cannot be sent in a header.

The ticket is single-use and short-lived, because a stream URL is visible to
proxies and appears in logs.

## Ticket types must not be interchangeable

A host issuing tickets for more than one purpose, for example event streams and
blob downloads, MUST bind each ticket to its purpose and verify that binding.
Distinguishing them by the shape of the value is the kind of assumption a later
change breaks silently.

# Blobs

~~~
PUT /ework/v1/blob            body: raw octets
                              ->  { "blob": "sha256:...", "size": 284913 }
GET /ework/v1/blob/<hash>?ticket=...
~~~

Upload is raw octets with the media type in `Content-Type`. Never base64 inside
JSON.

Download requires a single-use ticket rather than the session token, because the
consumer is a browser following a link, and a link carries no authorisation
header. The ticket is issued by a method call that performs the access check.

**The access rule is that a party may read a blob if it can see a task that
references it.** Attachment visibility inherits task visibility rather than
becoming a second permission system that drifts out of step with the first.

The host MUST advertise its maximum blob size and MUST enforce the value it
advertises. Advertising one limit and enforcing another produces rejections a
conforming client cannot diagnose.

# Security Considerations

Session tokens are bearer credentials. Hosts SHOULD scope them to a device and
MUST allow revocation.

The blob download endpoint MUST NOT serve by hash alone. The hash is derived
from content, so anyone holding the same file can compute it; an endpoint
serving by known hash discloses another party's attachment to whoever guesses,
and discloses, to anyone holding a candidate file, whether that file is present
on the host.

Responses for a nonexistent blob and for an unauthorised one MUST be identical.

# Privacy Considerations

The host observes call patterns even where it cannot read content: which method
families a client uses, and how often. Batching reduces the resolution of that
signal as a side effect, since a batch reveals the set of calls but not their
order in time.
