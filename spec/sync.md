---
docname: draft-wilhelm-ework-sync-00
title: "The e-work Protocol (EWP): Client-Server Synchronisation"
abbrev: EWP Sync
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9110,RFC9421,RFC9530,RFC8259,RFC8030,RFC8291,RFC8292
---

This document specifies how an e-work Protocol (EWP) client talks to its own
host: session establishment by challenge and response against a device key,
batched method calls and the method catalogue, per-type state strings for
incremental synchronisation, the encrypted operation log for end-to-end boxes,
binary blob upload and download, events and push, and the export and import
format.

<!-- abstract -->

# Introduction

The client-server interface is deliberately unremarkable. It borrows the batched
call and state-string model from JMAP, because that model already solves the
problem of a client keeping several object types in sync over a high-latency
link without polling each one.

It does not borrow JMAP's generic `get`/`set`/`changes` per type, which the
first revision of this document also copied. The protocol came to treat every
change as an act, with its own signature, authority and consequence, as the
consent and history documents specify, and the reference host grew one method
per act. On 2026-11-06 it served 61 methods and this document named 8. The
revision of that day brought the text up to what is in use (ADR-0033): whoever
reads this document to write a client needs to find here the names the host
answers to.

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
consumed. The token it issues is bound to that device, with an expiry that
SHOULD be at most 30 days and renewal through this same mechanism; revoking
the device, as the identity document specifies, kills its tokens immediately.
OAuth is NOT a requirement.

## The session document

`Session/get`, in the authenticated session, answers:

~~~
{
  "ewp": "0.1",
  "identity": "did:key:z6MkCleia...",
  "boxes": {
    "box-personal": {
      "name": "Personal",
      "capabilities": ["urn:ework:core", "urn:ework:e2ee"],
      "mode": "e2ee",
      "state": { "Task": "s:8842", "Entry": "s:1580", "Consent": "s:31",
                 "Quarantine": "s:7", "Offer": "s:12" }
    }
  },
  "apiUrl": "https://ework.example.com/ework/v1/rpc",
  "blobUrl": "https://ework.example.com/ework/v1/blob",
  "push": { "vapidKey": "BJ8v...", "maxPayloadBytes": 4096 },
  "eventsUrl": "https://ework.example.com/ework/v1/events",
  "limits": { "maxEnvelopeBytes": 262144, "maxBlobBytes": 104857600, "maxCallsPerBatch": 32 }
}
~~~

`mode` names the regime of each box, `e2ee` or `assisted`: the capability
table of the core document defers to this field to say which one governs a
given box. `limits` is where the host announces the sizes it enforces. `push`
appears only when the host advertises `urn:ework:push`; `vapidKey` is the
P-256 public key with which the host signs its sends ({{RFC8292}}). `state`
carries one state per type listed under State Strings.

An earlier revision of this section had the document fetched at
`GET /ework/v1/session`, which the reference host never served: the session
opens with `POST /ework/v1/session`, with the signature described above, and
the document comes through the batch like everything else. The document's shape
also differs from the reference host's, which serves a single box and puts
`state`, `mode` and `capabilities` at the root (open question 18).

## Covered components

{{RFC9421}} signs the components the using specification tells it to sign, and
nothing more. A specification that does not fix that set has not signed what it
believes it signed: a signature that does not cover the destination is valid at
another destination, and one that does not cover the body says nothing about the
body.

The signature base MUST cover exactly these components, and receivers MUST reject
a signature whose covered set differs:

~~~
@method, @target-uri, content-digest, content-type, created, expires, nonce
~~~

Every POST with a body MUST carry `Content-Digest` {{RFC9530}}, and the receiver
MUST recompute it and reject a mismatch. `created` and `expires` MUST be present
and the window between them MUST NOT exceed 300 seconds. Receivers MUST NOT
accept a signature whose `@target-uri` names another authority, however correct
the rest may be.

The challenge response MUST carry the host's own instant in `now`, and the
client SHOULD use it as `created` instead of the device's clock. The host's
clock is what judges the window, so signing by it removes the wrong-clock device
as a case at all.

Clock tolerance MUST be equal in both directions. Accepting `created` in the
past with a wide margin and in the future with a narrow one locks out the fast
device and lets the slow one through, and there is no reason to treat them
differently: clocks err both ways, and running fast is as common as running
slow. The resulting failure is among the worst to diagnose, because whoever is
trying to sign in reads that the signature expired and concludes that their
credential expired.

Replay is defended by the `nonce`, not by the window: it is issued by the host,
valid once, and deleted on use. The window exists to bound the life of a
signature that was never presented, which is why clock tolerance can be generous
at no cost to security.

**A challenge expires after 15 minutes.** The host SHOULD forget a nonce not used
within that time, and the client SHOULD request the challenge only when it is
about to sign. Fifteen minutes is ample for a signature made with the challenge's
`now`, and a challenge requested and abandoned should not become a permanent row
at the host: requesting one proves nothing, and without a deadline every request
that never became a session was kept forever.

That last rule closes the cross-host attack: without `@target-uri` covered, one
host requests a challenge from another in the user's name, returns the foreign
nonce to the user's client as its own, and uses the resulting signature to open a
session at the destination host. Since a single identity may live at several
hosts, this is the ordinary case rather than an exotic one. The same list governs
the host-to-host transport signature in the federation document. There is no
challenge there, so the nonce is generated by the signer and the receiver's
memory takes the challenge's place; the federation document says where each
parameter comes from.

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

## The API is per act

Every change the protocol defines as an act has its own method, with the
arguments of that act and the rules of the document that defines it: granting,
refusing, revising and revoking a relationship, knocking, sending an offer,
recording an entry. There is no generic `set`. A `Consent/set` with
`state: "revoked"` would leave the host guessing which of the two exits of the
consent document the person meant, and a `Task/set` over a shared task would go
around the entry, which is where authorship, authority and justification live
(the history document). Reading is `X/get` or `X/list`, and `X/changes` exists
where the collection grows without bound (see State Strings).

The name and semantics of every method in the catalogue below are fixed by this
specification. A host that offers the behaviour MUST offer it through the listed
method and with the semantics of the indicated document, and a client MUST NOT
depend on a method that is not in the catalogue. Methods marked with a
capability exist only when the host advertises it (see the core document), and
the client MUST hide the feature when it is absent. Which families every host
is required to serve is a matter for the conformance profiles of the core
document (open question 17).

## Catalogue

| Method | What it does | Defined in |
|---|---|---|
| `Session/get` | Returns the session document. | this document |
| `Account/export` | Produces the account export. | this document, the compliance document |
| `Account/delete` | Requests account deletion, which completes after the grace window. | the compliance document |
| `Account/deletionStatus` | Says whether a deletion is pending and when it completes. | the compliance document |
| `Account/cancelDelete` | Cancels the deletion within the window. | the compliance document |
| `Account/setPushHint` | Turns the clear urgency hint on or off. Capability `urn:ework:push`. | this document, the urgency document |
| `Push/subscribe`, `Push/unsubscribe`, `Push/list`, `Push/test` | The push subscription of the session's device. Capability `urn:ework:push`. | this document |
| `Events/ticket` | Single-use ticket for the event stream. | this document |
| `Blob/ticket` | Single-use ticket to download a blob, issued after the access check. | this document |
| `Task/get` | The tasks the session can see: those of the box and those of the circles it belongs to. | the data model document, the compartments document |
| `Task/changes` | The tasks that changed since a state. | this document |
| `Task/create` | Creates the person's own task, with no issuer (see below). | the data model document |
| `Task/relate`, `Task/unrelate` | Creates and removes a dependency between two tasks. | the data model document |
| `Task/arm` | Arms the pre-signed escalation of a critical task. | the urgency document, ADR-0023 |
| `Task/milestone` | Creates the public milestone of a task in a narrow compartment. | the compartments document |
| `Compose/parse` | Proposes a task from free text, creating nothing. Capability `urn:ework:compose`. | the core document |
| `Entry/list` | The history of a task. | the history document |
| `Entry/add` | Records an entry, with or without an action. It is how a task changes after it is created. | the history document |
| `Consent/list` | The account's relationships, in both directions (see below). | the consent document |
| `Consent/knock` | Delivers a `consent.request` the client composed and signed (see below). | the consent document |
| `Consent/grant` | Accepts a request from quarantine. | the consent document |
| `Consent/deny` | Discards a request, which stays recoverable for a while. | the consent document |
| `Consent/revise` | Revises the scope or the status level of a granted relationship. | the consent document |
| `Consent/revoke` | Ends a granted relationship, by polite revocation or by silent retirement (`silent`). | the consent document |
| `Consent/acceptOffer` | Accepts an in-person offer. | the consent document |
| `Consent/issueOffer`, `Consent/listOffers`, `Consent/revokeOffer` | Issues, lists and revokes the in-person offers of an organisation account. | the consent document |
| `Consent/block`, `Consent/unblock`, `Consent/blocked` | Blocks, unblocks and lists blocked issuers. | the consent document |
| `Quarantine/list` | The requests waiting in quarantine (see below). | the consent document |
| `Quarantine/discarded`, `Quarantine/restore` | The discarded requests still recoverable, and the return of one of them. | the consent document |
| `Refused/list` | The envelopes the edge refused, whole and signed, as evidence. | the federation document |
| `Address/list` | The addresses the account issued. | the identity document |
| `Address/rotate` | Rotates a relationship address. | the identity document |
| `Address/setAcceptsNew` | Opens or closes an address to new requests. | the identity document |
| `Offer/send` | Sends a task offer through a relationship in which the account is the sender (see below). | the consent document, the bridges document |
| `Offer/sent` | The record of what the account sent, and where it stands (see below). | this document, the consent document |
| `Project/create`, `Project/list` | Creates a project, and lists those of whoever asks, trimmed to their circles. | the compartments document |
| `Project/addCircle`, `Project/addMember` | Creates a circle, and puts a member in circles. | the compartments document |
| `Project/archive`, `Project/unarchive` | Archives and unarchives for whoever asks, and only for them. | ADR-0019 |
| `Policy/get`, `Policy/set` | The box's execution policy. | the actors document |

`Events/ticket` and `Blob/ticket` are methods rather than endpoints because it
is the authenticated call that performs the access check; the ticket only
carries its result.

The methods below also have their name and semantics fixed here, and the
reference host serves none of them yet. The first four are the path of
end-to-end boxes, which phase 1 of the roadmap does not have; `Task/query` is
the optional of assisted mode.

| Method | What it does | Defined in |
|---|---|---|
| `Envelope/send` | Submits to the federation an envelope the client composed. | the federation document, the cryptography document |
| `Envelope/pending` | Fetches the received envelopes the host keeps without reading. | the cryptography document |
| `Group/commit`, `Group/fetch` | MLS transport: the group's commits, and the devices' KeyPackages. | the cryptography document |
| `Task/query` | Filtering and ordering at the host, in assisted mode only. | this document, the cryptography document |

## The everyday methods

Six methods the reference clients call in every session, and which the earlier
revision of this document did not name. The arguments and responses below are
the ones the reference host already speaks.

**`Task/create`** creates the person's own task, with no issuer. Its arguments
are fields of the `Task` object of the data model document: `title`
(required), `description`, `start`, `due`, `timeZone`, `estimatedDuration`,
`urgency` and `escalation`, plus `attachments`, with blobs already uploaded by
`PUT` to the blob endpoint, which the host MUST check exist before creating.
To be born in a project, `project` and, optionally, `circle`, `sensitivity`
and `seal`, under the rules of the compartments document: the sensitivity
label decides the circle, not whoever creates. The response is
`{ "created": <Task>, "deliveredTo": [...] }`, and `deliveredTo` lists the
circle's members on other hosts to whom the task went out. After creation, the
task changes through `Entry/add` (ADR-0014). The reference implementation
still ignores `timeZone` and writes `America/Sao_Paulo` on every task with a
`due`: that is a host defect, and the data model document prevails.

**`Consent/list`** returns the account's relationships in both directions, in a
single list, with no arguments. Each item is the `Consent` object of the
consent document, with four fields only the account's host knows:

- `role`: `recipient` when the account receives through this relationship,
  because it issued the address, and `issuer` when it is the one sending. The
  host keeps both directions in the same shape, and `issuer` means the opposite
  in each, so without this field the client cannot tell which action to offer.
- `alias`: the relationship's address. With `role: issuer`, it is where
  `Offer/send` sends.
- `requested`: the scope the issuer requested, which is the ceiling of a
  revision.
- `capability`: the relationship's sending credential. On the sender's side, it
  is what the client uses to build the envelope the host does not compose, such
  as the pre-armed escalation of the urgency document.

`Consent/list` replaces the `Consent/get` of the earlier revision, and there is
no `Consent/changes`: the list is reread when the `Consent` state changes.

**`Consent/knock`** delivers a `consent.request` the client composed and
signed: `{ "envelope": <signed consent.request> }`. It is the only envelope the
client composes in phase 1 (the second form of envelope proof in the federation
document), and therefore the only one this method accepts. The host MUST refuse
an envelope of another type, an envelope whose `from` is not the account's
address, and an envelope signed by a key that is neither the root nor a device
of the account's identity document: without these three checks, the host would
put its own transport signature on someone else's knock. Before delivering, the
host MUST record which address it knocked on, which is what lets it recognise
the grant when it comes back from an opaque address. The response is
`{ "sent": true, "to": <destination>, "response": <what the destination answered> }`.

**`Quarantine/list`** returns the requests waiting in quarantine, newest first,
with no arguments:

~~~
{ "list": [ {
  "id": "018f6a2b-4c1d-7e2f-8a3b-9c0d1e2f3a4b",
  "issuer": "billing@acme-energy.example",
  "created": "2026-11-06T13:40:00Z",
  "revises": null,
  "currentScope": null,
  "request": { "type": "consent.request",
               "from": "billing@acme-energy.example",
               "...": "the whole signed envelope" }
} ] }
~~~

`id` is what `Consent/grant` and `Consent/deny` take. When whoever knocks
already has a live relationship, the request is a revision: `revises` carries
the relationship's `uid` and `currentScope` the scope in force today, because
the screen needs to show before and after. The request comes whole and signed
so that the client can check the issuer's identity without depending on the
host's summary.

**`Offer/send`** sends a task offer through a relationship in which the account
is the sender: `{ "to": <alias>, "task": <Task> }`. The host MUST refuse when
the account has no granted relationship with that `alias` on the sender's side
(`role: issuer` in `Consent/list`). It is the host that composes the
`task.offer`, presents the relationship's credential, grants the attachments'
blobs to the destination domain, signs with its `hostKey` (the first form of
envelope proof in the federation document) and writes the `Offer/sent` record.
The response is
`{ "sent": true, "to": <alias>, "response": <what the destination answered> }`.
It is the same operation as the issuer gateway's `POST /v1/offers` in the
bridges document, through the session of any account, a person's or an
organisation's.

**`Offer/sent`** returns, with no arguments, the record of what the account
sent: one object per offered task, most recently updated first.

~~~
{ "list": [ {
  "id": "019f2c1e-7a40-7c55-9d1b-2b3c4d5e6f70",
  "title": "Check the contract",
  "due": "2026-11-10",
  "to": "k7f3a2q9@ework.example.com",
  "progress": "in-process",
  "negotiation": {},
  "updatedAt": "2026-11-06T14:02:11Z"
} ] }
~~~

- `id` is the `uid` of the offered task. `title` and `due` are what the account
  wrote when offering, kept here because the task lives in the recipient's box
  (ADR-0029); `due` changes when the issuer itself reschedules and the
  rescheduling holds.
- `to` is the relationship's address, and it changes when the other side
  rotates it (the identity document).
- `progress` and `negotiation` are the last state the other side let come back,
  under its policy (the consent document). With `none`, nothing comes back, and
  the record stays as it left. In `negotiation`, the recipient appears by the
  relationship's contact key, and NEVER by the root (the consent document,
  ADR-0022).

The record MUST NOT contain anything beyond what the account wrote when
offering and what the other side let come back: it is the sender's list, not a
copy of the recipient's box. On a host that serves both sides, the temptation
is to copy from the task, which sits in the same database, and that is what the
reference implementation did with `negotiation`, carrying the recipient's root
`did:key` into the sender's record. It has its own state, `Offer`, and "What was sent"
below says when it changes. It is the same reading as the issuer gateway's
`GET /v1/offers/{id}`, as a list and through the session.

## Reference host extensions

The reference host also serves methods that are not protocol:

| Method | What it does | Why it is not protocol |
|---|---|---|
| `History/verify` | The host recomputes the chain and the signatures of a task's history and gives the verdict. | The history document says verification MUST NOT depend on the host that stored the history. The host's verdict on its own history is a diagnostic, and whoever distrusts the host verifies at the client. |
| `Account/setNotify` | Turns on the email notice for when the app is closed. | The email goes out from the vault, which is an application layer (ADR-0024, ADR-0027). |
| `Regime/describe` | Returns the text of a regulatory profile for the grant screen. | The profiles are registered in the compliance document, and the client MAY carry them itself: the method only returns text. |
| `Gateway/createKey`, `Gateway/listKeys` | API keys of the issuer gateway of an organisation account. | The bridges document defines the gateway's REST API, not how its keys are administered. |
| `Calendar/feed`, `Calendar/revoke` | Read-only ICS feed of the tasks, and revocation of the link. | The outgoing calendar bridge is the first open question of the bridges document. |
| `Admin/get`, `Admin/set` | Who administers the host, and whether signup is open. | Administering the host is an operation of an instance, not of the protocol (ADR-0026). |

An extension is not protocol: a client MUST NOT depend on one for anything this
specification requires, and MUST work against a host that does not have it.
The reference host answers `malformed` to a method it does not know (open
question 16), which is what a client meets when it calls another host's
extension. How to tell an extension from protocol on the wire is open question
20.

## What left

`Task/set`, `Project/get|set|changes`, `Consent/get|set|changes` and `Blob/ref`
were in the first revision's list and never existed at the reference host. They
leave: a task is born through `Task/create` and changes through `Entry/add`;
projects and relationships have the act methods of the catalogue; and what a
client needs to download a blob is the ticket from `Blob/ticket`. With
`Task/set` go its `ifInState` and `stateMismatch`; concurrent writes are under
State Strings.

# State Strings

Every object type carries an exact, opaque state string per box. The types with
a state are `Task`, `Entry`, `Consent`, `Quarantine` and `Offer`.

~~~
"state": { "Task": "s:412", "Entry": "s:1580", "Consent": "s:33",
           "Quarantine": "s:7", "Offer": "s:12" }
~~~

Every write that changes what reading a type returns MUST change that type's
state, and every mutation response returns `oldState` and `newState` (open
question 18). A client holding a prior state calls `Task/changes` with it and
receives what was created, updated and destroyed, plus the new state, rather
than refetching the collection. The host MUST keep enough history to compute
that delta or answer `cannotCalculateChanges`, in which case the client falls
back to a full fetch rather than silently missing changes. The fallback is the
rare event, not the normal mode: the anti-SyncML lesson.

**Short collections have no `/changes`.** `Consent`, `Quarantine` and `Offer`
are lists a person scans by eye, and the client rereads the whole list
(`Consent/list`, `Quarantine/list`, `Offer/sent`) when the state changes.
`/changes` only pays off where the collection grows without bound, which is
`Task`. `Entry` changes per task, and its state says only that some history
moved, not which (open question 14).

**Concurrent writes.** In shared content the write is the entry, and whoever
signs it declares in `after` the tip they knew (the history document). The
reference host checks that the entry references the last one it holds for that
task and refuses one that does not, which is the history document's fork
detection applied at the box: the client rereads the history and signs again.
In an end-to-end box the rule is the fast-forward of the operation log below.

# End-to-End Boxes: the Operation Log

In `mode: "e2ee"` the host reads no objects; the synchronised unit is the
**version**: an encrypted blob containing a list of operations
`{op: create|update|destroy, uid, props...}` over the box, pointing at its
`parentVersion`.

- The version chain is **linear**: the host accepts a new version only if its
  `parentVersion` is the current tip; otherwise it answers `not-fast-forward`,
  and the client downloads the new versions, **rebases its operations
  locally**, and tries again (the TaskChampion model).
- `parentVersion` MUST be the sha256 hash of the previous version's
  ciphertext, and every version MUST be signed by the key of the device that
  produced it. Clients MUST verify the signature and the continuity of the
  whole chain they receive, and MUST refuse a chain whose tip does not descend
  from the last one they knew.
- Conflict resolution during rebase: per property, the last `updated` wins,
  and a tie is broken by the greater operation `uid`. Deterministic across all
  clients.
- Periodic snapshots, encrypted, compact the chain; the host MAY prune
  versions older than the oldest snapshot referenced by an active device.
  Pruning MUST NOT reach a version some active device has not yet confirmed,
  and the host MUST serve, alongside the snapshot, the signature of the device
  that produced it.

`parentVersion` sits outside the AEAD because the host needs to read it to
apply the fast-forward rule. Outside the AEAD and unsigned, it would be a
field the host chooses: presenting a freshly added device with an old tip
would be enough for the client, holding no material to distrust it, to rebase
onto it and rewrite the box with the past. Signing the version and verifying
descent turns that manoeuvre into a refusal by the client, and it is the same
reasoning the history document applies to shared history.

The `Task/*` methods keep existing for the client locally; on the wire they
become operations inside a version. Server-side `Task/query` does not exist in
an end-to-end box: search is local.

## Concurrency in shared content

Compartments and relationships share content through history entries, as the
history document specifies, not through the per-box log of this section. The
state of a shared task is derived by applying the actions of its entries in
causal order. For property revisions (accepted `update` and `counter` actions),
last-writer-wins per property holds along that same order, with the history
document's tiebreak, which is the entry hash. Where the comparison involves
time, the compared value MUST be `min(sentAt, receivedAt)`, never raw `sentAt`:
whoever chooses their own timestamp would also choose which revision wins
forever. The rule is deterministic: all clients arrive at the same state with
no extra coordination.

**An action incompatible with the current state, OR signed by an identity
without authority for it, MUST be recorded as an entry with no effect, never
discarded:** the history tells what each party attempted, and the state tells
what held. Completing a task another member had already completed is the state
case; an issuer signing `accept` over its own offer is the authority case, and
the two deserve the same fate, which is to stay written without holding.

# Events and Push

## Foreground: the event stream

While the application is open, the client receives changes over an **event
stream** (Server-Sent Events, `text/event-stream`) at the session document's
`eventsUrl`. The stream emits a `changed` event whose data is the state strings
per box and per type:

~~~
event: changed
data: {"changed": {"box-personal": {"Task": "s:8843"}}}
~~~

- **Events carry only the state strings, never content.** The client compares
  them with the last states it knows and fetches, over the authenticated
  channel, only the types that changed. This keeps the event channel free of
  anything worth intercepting.
- The first event after connecting carries every current state, and the client
  MUST NOT treat it as a change: comparing it with what it already knew is what
  avoids reloading the whole box on every reconnection.
- The connection is authenticated by a **single-use ticket** obtained through
  `Events/ticket`, not by the session token. `EventSource` sends no headers,
  and a durable token in a query string ends up in proxy and server logs; the
  ticket is good for one connection, expires within a minute and is consumed
  on use, which is what a URL can carry at no cost. Event tickets and blob
  tickets MUST NOT be interchangeable.
- The stream drops whenever the operating system suspends the application, and
  that is the normal case rather than the exception: the client MUST obtain a
  fresh ticket on every reconnection and SHOULD back off exponentially, capped
  at 30 seconds, when reconnecting fails.
- A host MAY offer the same stream over WebSocket, with the same ticket and the
  same data. SSE is the minimum every host MUST serve, because it crosses any
  HTTP proxy and needs no library on the client.

An earlier revision of this section said WebSocket, and the reference
implementation never had one: it served SSE from the first day, for the reasons
above. The text now describes what exists, and WebSocket stays the optional it
always was in practice.

### What was sent

The `Offer/sent` record changes because of what the other side does, and that
change is the one the sender wants to see at once. That is why it has its own
state, `Offer`, which the stream carries like the other types of State Strings.

- The host MUST change the account's `Offer` state on every write to the
  record: the send, the status the other side returns, the issuer's own entry
  about the task (ADR-0029) and a change of `to` by rotation.
- The write MUST be stored before the event goes out. An event emitted earlier
  carries the old state, and the client, which compares, discards it as a
  repeat.
- The sender's host MUST update the record when an entry with an action arrives
  through the relationship: it applies the signed action to the last recorded
  `progress`, through the same state machine as the box (the data model
  document), and MUST NOT copy a `progress` that travels outside the entry,
  because the signature does not cover that field (the history document). This
  holds also when both sides live on the same host.
- A client that shows what the account sent MUST reread `Offer/sent` when the
  `Offer` state changes. A host predating this revision emits no `Offer`, and
  the client MUST tolerate its absence by rereading `Offer/sent` on a full
  synchronisation.

This section was born from the mobile app's bell, on 2026-11-06: the sender
only learned of acceptance and completion on a full synchronisation, because
the record had no state and a client that rereads only the types that changed
had nothing to reread. Reading the host showed two more defects. Between
different hosts, the returned status stored the entry and did not update the
record, so not even a full synchronisation showed the acceptance; and, with both
sides on the same host, the sender's event went out before the write. All three
are defects of the reference implementation, and its correction follows this
text.

## Background: push

With the application closed, what wakes the client is a **content-free push**:
it says that something changed in a box, and the client fetches over the
authenticated channel. What reaches the push service, and any intermediary, is
ciphertext for the device plus the transport headers. Critical reminders and
alarms fire from **local scheduling** on the device; push is acceleration, not
the source of truth, as the urgency document specifies.

The protocol speaks **Web Push only**: {{RFC8030}} for delivery, {{RFC8291}}
for payload encryption and {{RFC8292}} (VAPID) for host identification.
UnifiedPush is Web Push, and needs no special case. The protocol MUST NOT
depend on FCM or APNs specifically: where the platform requires the vendor's
service, the bridge is a relay chosen by whoever publishes the application
(see Relays below), and the host does not know it exists.

### Announcement

A host that delivers push advertises `urn:ework:push` in `capabilities` (see
the core document) and publishes in `push.vapidKey`, in the session document,
the P-256 public key with which it signs its sends ({{RFC8292}}). Clients MUST
hide every push setting when the capability is absent, and MUST subscribe
again when the published key changes.

### Subscription

- `Push/subscribe {endpoint, keys: {p256dh, auth}, expires?}` registers the
  subscription of the session's device and answers `{id, device, expires,
  replaced}`. `endpoint` MUST be `https`; `p256dh` and `auth` are the public
  key and the authentication secret of {{RFC8291}}, generated on the device.
  The host MUST NOT accept `device` as an argument: the subscription belongs
  to the device that opened the session, and to it alone.
- A device has **one** active subscription: subscribing again replaces the
  previous one, and `replaced` says that it happened.
- `Push/unsubscribe {id?}` removes the subscription (without `id`, the
  session device's). `Push/list {}` answers, per subscription, `id`, `device`,
  `endpointHost`, `created`, `lastUsed`, `expires` and `failures`, and MUST
  NOT answer the endpoint URL: a session token stolen from one device cannot
  gain the power to wake another. `Push/test {}` sends a test push to the
  session device's subscription, under a rate limit.
- The host MUST delete a device's subscriptions when the device leaves the
  identity document, when the account deletion completes, when the service
  answers `404` or `410`, after `expires` and after repeated failures.

### Payload, triggers and headers

- The cleartext payload is a JSON object with at most `box` (the box's
  handle), `stateHint` (the same object the event stream emits) and
  `urgencyHint`, the last only when the user enabled the clear hint (see the
  urgency document). It MUST be encrypted with `aes128gcm` ({{RFC8291}}),
  padded to a size class (256, 512 or 1024 bytes), and the total MUST NOT
  exceed 4096 bytes. Clients MUST treat the payload as a hint: they fetch,
  they do not trust.
- The host MUST send in three cases: an arrival through the federation inbox
  for that identity (an offer, a consent request, an entry from a peer, a
  project invitation, status coming back), the release of escalation step 0
  and the final give-up of a delivery the person needs to know about. The
  host MAY, with opt-in per subscription, announce changes made by another
  device of the same identity, always with `Urgency: very-low`. The host MUST
  NOT send a push for reminders or alarms, which are local, nor for the
  person's own actions on the same device.
- {{RFC8030}} headers, by cause: arrival `Urgency: normal` and `TTL` 86400;
  escalation release `Urgency: high` and `TTL` 3600; sync hint `Urgency:
  very-low` and `TTL` 21600. `Topic` is a random value per subscription,
  which makes the service replace a pending push with the newest one, and it
  never derives from the `did`. With the clear hint enabled, `Urgency`
  follows the hint: `min` and `low` become `low`, `normal` stays `normal`,
  `high` and `critical` become `high`.
- Limits: at most one push every 30 seconds per subscription for `normal` and
  below (`high` is immediate), and the host MUST NOT exceed 100 per
  subscription per day. The excess is dropped, and the client catches up on
  its next synchronisation.

### Relays

A relay is a service, chosen by whoever publishes the application, that
exposes one Web Push endpoint per device and forwards what it receives to the
vendor's service. To the host it is an endpoint like any other, which is why
the protocol still depends on nobody. A relay MUST NOT be able to read the
payload (it never sees `p256dh` nor `auth`), SHOULD pin the VAPID key seen on
the first send to each endpoint, MUST answer identically for an unknown
endpoint and a deleted one, and MUST NOT store the VAPID `sub`.

# Blobs

~~~
PUT /ework/v1/blob            body: raw octets
                              ->  { "blob": "sha256:...", "size": 284913 }
GET /ework/v1/blob/<hash>?ticket=...
~~~

Upload is raw octets with the media type in `Content-Type`. Never base64 inside
JSON.

Upload is resumable: `PATCH` with `Upload-Offset`, the tus model, and support
is mandatory for files above 8 MiB. Download MUST support `Range`.

Download requires a single-use ticket rather than the session token, because the
consumer is a browser following a link, and a link carries no authorisation
header. The ticket is issued by a method call that performs the access check,
is valid for a single blob, and expires. Tickets MUST NOT be interchangeable
with tickets issued for another purpose: a host issuing tickets for more than
one purpose MUST bind each ticket to its purpose and verify that binding.
Distinguishing them by the shape of the value is the kind of assumption a later
change breaks silently.

**Knowing the hash is not authorisation.** A party may read a blob if it can see
a task that references it. Attachment visibility inherits task visibility rather
than becoming a second permission system that drifts out of step with the first.
A request without such a reference MUST be answered `unauthorized`, identical in
code, body and timing to the response for a blob that does not exist.

A content address is a predictable, shareable identifier and never a secret: it
appears in exports, logs, backups and every copy of the task that references it.
Treating it as a credential hands one user's attachments to anyone who repeats a
hash.

Deduplication MUST be per box rather than global across users, and the upload
response MUST be indistinguishable between a new blob and one the host already
held: same code, same body, same timing. Otherwise upload becomes a possession
oracle, and confirming that a person holds a specific file is a disclosure in
itself, even without ever opening it.

In an end-to-end box the stored blob is the ciphertext: AEAD, with a per-blob
key wrapped inside the object that references it. The hash is the hash of the
ciphertext.

A blob without an object reference for more than 30 days MAY be collected.
Quotas are host policy, announced in the session document's `limits`.

The host MUST advertise its maximum blob size and MUST enforce the value it
advertises. Advertising one limit and enforcing another produces rejections a
conforming client cannot diagnose.

# Export and Import

`GET /ework/v1/export`, or an asynchronous generation of it, produces a tar
file with `identity.json` (the public document), `boxes/<id>/tasks.jsonl`,
`projects.jsonl`, `consents.jsonl`, `envelopes.jsonl` (the history), and
`blobs/` by hash. In an end-to-end box the export is the pair of encrypted
version chain and snapshot, plus the encrypted blobs: readable only with the
user's keys.

Import at the new host accepts this format in full: it is the mechanism of
the migration the identity document specifies. The format is stable, versioned
and documented: leaving has to be as easy as entering.

# Security Considerations

Session tokens are bearer credentials. Hosts SHOULD scope them to a device and
MUST allow revocation.

The blob download endpoint MUST NOT serve by hash alone. The hash is derived
from content, so anyone holding the same file can compute it; an endpoint
serving by known hash discloses another party's attachment to whoever guesses,
and discloses, to anyone holding a candidate file, whether that file is present
on the host.

Responses for a nonexistent blob and for an unauthorised one MUST be identical.

Fixing the covered component set is an obligation of every specification that
uses {{RFC9421}}, and the conformance suite MUST test that a signature with a
different covered set is refused.

The signed version chain takes from the host the last position from which it
decided alone what the client sees. It can still delay and refuse, which is
denial of service in plain sight, and it can no longer revert, truncate or
fork in silence.

The URL of a push subscription is a capability: whoever holds it wakes the
device with garbage, and reads nothing. Clients MUST discard a push that does
not decrypt, and a device receiving repeated garbage SHOULD subscribe again
with a new endpoint. VAPID prevents a third party from reusing the endpoint
only where the push service enforces it; where it does not, key pinning at the
relay covers the rest.

Act methods are what keep each check of the originating document in a single
place. With a generic `set`, the same state change would arrive by different
paths, and each path would have to redo the checks of authority (the history
document), of ceiling and of the relationship's side (the consent document)
that the act's method does once. The three checks of `Consent/knock` are the
example: the envelope arrives ready from the client, and it is the method that
stops the host from putting its own transport signature on someone else's
envelope.

The record of what was sent is written by the sender's host from what arrives
through the relationship, and the rule that stops the envelope from choosing
the task's state applies to it too: `progress` comes from the action signed by
the relationship's contact key (the consent document), applied to the last
known state, and never from a field of the body. Without that, whoever managed
to speak through the relationship's address would write "completed" into the
sender's list.

An extension carries no guarantee of the protocol. `History/verify` is the case
that motivated the class: a host that verifies its own history and answers
"intact" proves nothing to whoever distrusts it, because it is the host that
stored what it verifies.

# Privacy Considerations

The host observes call patterns even where it cannot read content: which method
families a client uses, and how often. Batching reduces the resolution of that
signal as a side effect, since a batch reveals the set of calls but not their
order in time.

In an end-to-end box the host sees only the cadence of versions, the sizes,
and the blob hashes. That server-side `Task/query` exists only in assisted
mode is deliberate: a capability is announced, never implicit.

Push adds metadata to what the host sees, and it enters the exhaustive list of
the cryptography document: the host learns the host name of each device's
endpoint, which reveals the platform or the distributor the person uses; the
push service and any relay learn the cadence of wake-ups, the size class and
the `Urgency` header, which reveals an escalation in progress and, with the
hint enabled, the urgency. None of them sees box, sender or task.

The `Offer` state creates no new metadata. The host already wrote the record of
what was sent, the event stream goes only to the account's own sessions, and in
a push it travels inside the encrypted payload. The record itself says nothing
the sender did not already know or did not receive under the other side's
policy, and it names the recipient by the relationship's contact key, like
everything else that reaches the issuer: a record that kept the root would
undo, at the sender's host, the pseudonym the relationship exists to keep
(ADR-0022).

# Open Questions

1. Transport compression (zstd) as a negotiation.
2. Session token lifetime and rotation: do the values of the session section
   become the norm, or stay a recommendation?
3. The 300-second window of the covered components and the five-minute clock
   tolerance of the history document were chosen by analogy with other
   protocols, not by measurement. What is the right value for mobile clients
   with bad clocks, and should it be negotiated per host instead of fixed?
4. Per-box deduplication costs storage for a large attachment repeated across
   compartments of the same user. Is an intermediate level worth it,
   deduplication per identity instead of per box, or does that reopen the
   possession oracle inside the user's own account?
5. Verifying the whole version chain is cheap after a snapshot and expensive
   on a new device downloading everything. What is the acceptable verification
   floor when the client trusts its own signed snapshot?
6. The sync hint between devices of the same identity starts off. Should it
   start on, given that the cost is only cadence?
7. The 30-second coalescing, the cap of 100 per day and the `TTL` values were
   chosen by analogy, not by measurement. What is the right value on a real
   device, and are they worth negotiating per host?
8. One subscription per device excludes running two distributors at once. Is
   more than one worth allowing?
9. `urgencyHint` inside the payload and `Urgency` in the header say almost the
   same thing. The header is needed for the system's priority; is the payload
   redundant, or does it have value of its own for the client?
10. A relay accepts registration from any device. Is app attestation (App
    Attest, Play Integrity) worth the dependency it brings?
11. After the wake-up, a client in assisted mode may fetch the content and
    enrich the notification locally. That respects RF-32, which is about what
    travels in the push; should the specification say whether local enrichment
    is in scope?
12. May devices of class `autonomous` subscribe to push? And organisation
    accounts, which already have webhooks?
13. On a host with several boxes per identity, `box` in the payload names the
    box. Does the host emit one push per box or one per identity?
14. **`Entry/changes`.** The `Entry` state says that some history moved, not
    which. A client that wants its histories current rereads `Entry/list` for
    every open task, or waits for a full synchronisation. Is an `Entry/changes`
    that returns the tasks whose history moved worth it, or should the `Task`
    state change along with the task's history?
15. **`Project` and `Refused` in the stream.** The reference host changes a
    `Refused` state when the edge refuses something and does not emit it, and
    projects have no state at all, although the mobile app already expects
    `Project` in the stream. Do both join State Strings, under the rule "What
    was sent" gives `Offer`?
16. **Unknown method.** The reference host answers `malformed` (the core
    document) to a method it does not know, and the client cannot tell "this
    host lacks the method" from "I sent a bad argument". JMAP has its own error
    for this, `unknownMethod` (RFC 8620, section 3.6.2). Is registering an
    `unknown-method` in the core document worth it?
17. **Families and conformance.** The catalogue fixes names and semantics, and
    does not say which families every host must serve. The host profile of the
    core document covers the core; do projects (the compartments document),
    escalation (the urgency document) and the execution policy (the actors
    document) need a registered capability, or do they join the core?
18. **Three differences of shape found in the revision of 2026-11-06.** The
    reference host's session document is flat and single-box, with `state`,
    `mode` and `capabilities` at the root, while this document's is per box;
    `Task/changes` answers `changed`, with whole objects, and `newState`, with
    no `created`, `updated` or `destroyed` and no `cannotCalculateChanges`;
    and mutation responses carry no `oldState` or `newState`. For each, it
    remains to decide whether the text follows the host or the host follows
    the text.
19. **What was sent, in the export.** The export does not carry the
    `Offer/sent` record, and whoever migrates host loses the list of what they
    sent, together with the address each offer went out through. Does it join
    the archive, as `offers.jsonl`?
20. **Extensions on the wire.** A client knows a method is an extension only if
    it carries the list of extensions with it. A namespace in the method name,
    like that of capabilities (the core document), would settle it on the
    wire, and would force renaming methods clients already use. Is it worth it?
