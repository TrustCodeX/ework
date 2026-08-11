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
batched method calls, per-type state strings for incremental synchronisation,
the encrypted operation log for end-to-end boxes, binary blob upload and
download, events and push, and the export and import format.

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
consumed. The token it issues is bound to that device, with an expiry that
SHOULD be at most 30 days and renewal through this same mechanism; revoking
the device, as the identity document specifies, kills its tokens immediately.
OAuth is NOT a requirement.

## The session document

`GET /ework/v1/session` (authenticated) answers:

~~~
{
  "ewp": "0.1",
  "identity": "did:key:z6MkCleia...",
  "boxes": {
    "box-personal": {
      "name": "Personal",
      "capabilities": ["urn:ework:core", "urn:ework:e2ee"],
      "mode": "e2ee",
      "state": { "Task": "s:8842", "Project": "s:112", "Consent": "s:31" }
    }
  },
  "apiUrl": "https://ework.example.com/ework/v1/rpc",
  "blobUrl": "https://ework.example.com/ework/v1/blob",
  "pushUrl": "https://ework.example.com/ework/v1/push",
  "eventsUrl": "wss://ework.example.com/ework/v1/events",
  "limits": { "maxEnvelopeBytes": 262144, "maxBlobBytes": 104857600, "maxCallsPerBatch": 32 }
}
~~~

`mode` names the regime of each box, `e2ee` or `assisted`: the capability
table of the core document defers to this field to say which one governs a
given box. `limits` is where the host announces the sizes it enforces.

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

That last rule closes the cross-host attack: without `@target-uri` covered, one
host requests a challenge from another in the user's name, returns the foreign
nonce to the user's client as its own, and uses the resulting signature to open a
session at the destination host. Since a single identity may live at several
hosts, this is the ordinary case rather than an exotic one. The same list governs
the host-to-host transport signature in the federation document.

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
| `Envelope/*` | `send` submits to the federation, `pending` fetches what arrived |
| `Group/*` | MLS transport (`commit`, `fetch`), see the cryptography document |
| `Blob/*` | Attachment access tickets |
| `Account/*` | Export, deletion |

Server-side `Task/query`, filter and ordering at the host, exists only in
assisted mode.

# State Strings

Every object type carries an exact, opaque state string per box.

~~~
"state": { "Task": "s:412", "Entry": "s:1580", "Consent": "s:33", "Quarantine": "s:7" }
~~~

Every mutation response returns `oldState` and `newState`. A client holding a
prior state calls `Task/changes` with it and receives what was created, updated
and destroyed, plus the new state, rather than refetching the collection. The
host MUST keep enough history to compute that delta or answer
`cannotCalculateChanges`, in which case the client falls back to a full fetch
rather than silently missing changes. The fallback is the rare event, not the
normal mode: the anti-SyncML lesson.

Writes are optimistic: `Task/set` accepts `ifInState`, and a conflict answers
`stateMismatch`, upon which the client redoes its change on top of the new
state.

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

In the foreground, the WebSocket at the session document's `eventsUrl` emits
change notices:

~~~
{ "changed": { "box-personal": { "Task": "s:8843" } } }
~~~

**Events carry only the state strings, never content.** A client that receives
one fetches what changed over the authenticated channel. This keeps the event
channel free of anything worth intercepting.

In the background, push is Web Push {{RFC8030}} with encrypted payload
{{RFC8291}} and VAPID {{RFC8292}}; UnifiedPush is an accepted equivalent. The
push carries at most `{box, stateHint, urgencyHint}`: it wakes the client,
which fetches over the authenticated channel. The protocol MUST NOT depend on
FCM or APNs specifically; where the platform forces one, the mapping is the
client's problem, not the wire's.

Critical reminders and alarms fire from **local scheduling** on the device;
push is acceleration, not the source of truth, as the urgency document
specifies.

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

# Privacy Considerations

The host observes call patterns even where it cannot read content: which method
families a client uses, and how often. Batching reduces the resolution of that
signal as a side effect, since a batch reveals the set of calls but not their
order in time.

In an end-to-end box the host sees only the cadence of versions, the sizes,
and the blob hashes. That server-side `Task/query` exists only in assisted
mode is deliberate: a capability is announced, never implicit.

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
