---
docname: draft-wilhelm-ework-data-model-00
title: "The e-work Protocol (EWP): Task Data Model"
abbrev: EWP Data Model
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8984,RFC9253,RFC8259
---

This document specifies the task data model of the e-work Protocol (EWP). The
model is a profile of JSCalendar {{RFC8984}}, extended with the relationship
types of {{RFC9253}}, rather than a new vocabulary.

Extensions are confined to the areas where an actual gap exists: federated
negotiation, provenance and consent, binary attachments, typed payloads,
actions, urgency, and compartmentalisation.

<!-- abstract -->

# Introduction

A task has a title, a due date, a state, and a priority. JSCalendar already
specifies all of these, with the review and errata of a published standard
behind them. Inventing a parallel vocabulary would cost interoperability with
every calendar system in existence and buy nothing.

EWP therefore profiles JSCalendar `Task`, adds the relationship types of
{{RFC9253}}, and extends only where the gap is real. Every extension carries the
`ework:` prefix, so a JSCalendar consumer that does not know EWP can ignore them
and still read the task.

## Conventions and Definitions

<!-- rfc2119 -->

# Task Object

~~~
{
  "@type": "Task",
  "uid": "019fd1d5-e84c-76a0-9961-f1ccbcead44c",
  "title": "Pay electricity bill",
  "description": "Reference 8829-4471",
  "created": "2026-09-01T10:00:00Z",
  "updated": "2026-09-01T10:00:00Z",
  "due": "2026-09-10T12:00:00",
  "timeZone": "America/Sao_Paulo",
  "priority": 3,
  "progress": "needs-action",
  "relatedTo": { "019fd0...": { "relation": { "dependson": true }, "gap": "P2D" } },

  "ework:urgency": "normal",
  "ework:origin": { "issuer": "billing@utility.example",
                    "consent": "ework:consent/019f3c00-...",
                    "offeredAt": "2026-09-01T10:00:00Z",
                    "envelope": "019fd1d5-..." },
  "ework:negotiation": { },
  "ework:statusSharing": "milestones",
  "ework:dedupKey": "energy-2026-09",
  "ework:payloads": [ ],
  "ework:attachments": [ ],
  "ework:actions": [ ],
  "ework:execution": { },
  "ework:machine": { },
  "ework:projectUid": "019fd0...",
  "ework:compartment": "general",
  "ework:classification": { },
  "ework:sealed": [ ],
  "ework:escalation": { },
  "ework:updatesMilestone": null
}
~~~

## Inherited fields

`uid`, `title`, `description`, `created`, `updated`, `due`, `start`,
`timeZone`, `priority`, `progress`, `percentComplete`, `estimatedDuration`,
`recurrenceRules`, `participants`, `relatedTo`, `keywords`, `links`, and
`alerts` carry their JSCalendar {{RFC8984}} semantics unchanged.

`description` is limited Markdown, with no HTML. `start` marks the planned
start or window. `keywords` holds the user's free-form tags, and `links` holds
typed external URLs. `alerts` carries reminders; their interaction with urgency
is specified in the urgency document.

`progress` takes the values `needs-action`, `in-process`, `completed`,
`cancelled`, `failed`, plus two EWP additions: `blocked` and
`awaiting-confirmation`.

## The two state axes

Negotiation, held per recipient participant, runs `pending` to `accepted`,
`declined`, `countered` or `expired`. `countered` is not terminal: a counter
hands the ball back, and the other side accepts, declines or counters again,
until someone settles it or the offer expires.

Execution runs over `progress`. Each edge below is labelled with the action
(defined in the history document) that produces it:

| Action | From | To |
|---|---|---|
| `start` | `needs-action` | `in-process` |
| `complete` | `needs-action`, `in-process` | `completed`, or `awaiting-confirmation` when the policy requires it |
| `fail` | `in-process` | `failed` |
| `cancel` | `needs-action`, `blocked`, `in-process`, `awaiting-confirmation` | `cancelled` |
| `decline` | `needs-action` | `cancelled` |
| `confirm` | `awaiting-confirmation` | `completed` |
| `contest` | `awaiting-confirmation` | `in-process` |

Two transitions carry no action, because nobody signs them: `blocked` is
derived from the dependency graph, and `awaiting-confirmation` to `failed` is
the confirmation deadline expiring under `onTimeout: "fail"`.

Completing without starting is deliberate. Requiring `start` first would turn a
label into an obligation: someone who does the thing and marks it done never
passed through "in progress", and most of one person's tasks are like that.
`start` tells others that someone picked it up; it does not unlock completion.

`cancel` from `blocked` and from `awaiting-confirmation` is equally
deliberate. Those are the two states where the task waits on somebody else, and
without that edge the owner cannot end their own task without the other party
acting first.

## Action and state compatibility

The synchronization document requires an action incompatible with the current
state to be recorded as an entry with no effect. This is what "incompatible"
means:

**An action is compatible with the current state when the machine above has an
edge leaving that state labelled with that action.** Otherwise it is
incompatible, and receivers MUST record the entry and apply no effect.

The predicate matches on the action name, not on the destination state, and the
difference is not academic. `needs-action` reaches `completed` because of
`complete`; a predicate comparing destinations alone would admit `confirm` by
the same route, and whoever confirms would single-handedly complete a task
nobody ever executed. The confirmer role exists precisely so it is not the
executor role.

Where `blocked` is materialized it answers actions as `needs-action`: it is the
same state seen next to the dependency graph, not a state anyone signs. The
pending dependency already blocks `start` and `complete` on its own.

A task is **actionable** when its negotiation is accepted or absent, its
dependencies are satisfied, and its `start` or window has been reached. Clients
MUST compute actionability locally; it is what orders the list of what can be
done right now.

`priority` is 0 to 9 and expresses importance, with 0 meaning undefined and 1
the highest, as in iCalendar. It is a different axis from
`ework:urgency`, which expresses demand on attention. A utility bill can be
important and not urgent; conflating them produces either alarm fatigue or
missed deadlines.

# Participants and Roles

`Participant`: `{ "identity": "...", "roles": [...], "negotiation": ...,
"progress": ... }`, where a role is one of `owner`, `executor`, `issuer`,
`observer` or `escalation-contact`.

- `owner` controls the task, normally whoever accepted the offer.
- `executor` will do it; in a personal task this is the same person as `owner`.
- `issuer` made the offer, and receives status according to the policy.
- `observer` sees according to the project's policy and does not act.

Delegation, meaning transferring `executor` to a third party, is a registered
future extension; the role vocabulary already accommodates it.

# Projections

The same task list is read as a list, a calendar or a board. These are
projections of one object, never separate copies.

**List.** The default ordering is by actionability, then by `due`, with blocked
tasks grouped at the end. Every task appears.

**An archived project belongs to the reader.** Tasks in a project the READER
archived MUST NOT appear in any projection or in any counter. Archiving is the
preference of whoever reads, not a state of the project: it does not travel
between hosts, it does not change what the other members see, and it interrupts
neither delivery nor history. The tasks remain readable and exportable.

**Calendar.** Tasks with a `due` (or `start`) appear on the local date derived
from `timeZone`; with an `estimatedDuration` they occupy a block, and without
one they become all-day items. Tasks with no date do NOT appear in the
calendar, and the client SHOULD offer a separate area for them.

**Board.** The default columns are the values of `progress`, and moving a card
between columns is equivalent to changing `progress`. A project MAY declare its
own columns, mapping each to a base `progress`.

A task stores its column in `ework:column`. Clients that do not implement a
board MUST ignore `ework:column` and use `progress`, losing nothing: that is
why every column declares its base `progress`. Personal view preferences (which
view opens, filters, groupings) are local client state and MUST NOT travel in
the protocol, with the exception of a shared project's columns, which are an
agreement between members.

# Dependencies and Relations

`Relation` follows {{RFC9253}}: `{ "relation": { "dependson": true }, "gap":
"P2D" }`, with the relation types in lower case: `parent`, `child`, `dependson`,
`finishtostart`, `starttostart`, `finishtofinish`, `starttofinish`, `first`,
`next`, plus `blocks` as the materialisable inverse of `dependson`.

- The graph MUST be acyclic; receivers MUST reject a relation that would create
  a cycle.
- `gap` delays actionability: assembly starts two days after delivery, not at
  the same instant.
- References cross identities and hosts, since `uid` is global.
- **A dependency crossing compartments MUST use a public milestone as proxy.**
  Pointing at an invisible task would leak its existence and its lifecycle,
  turning the project graph into an inference channel. This is specified in the
  compartments document.

Implementations MUST enforce the acyclicity check **before** writing. Checking
afterwards leaves the damage in the store, and a cycle deadlocks both tasks with
no user-visible explanation.

# Extensions

## Provenance

`ework:origin` records who offered the task, under which consent, and in which
envelope. It is what makes a task auditable back to its source, and what allows
the recipient to answer the question "why is this in my box?" without trusting
the sender's own claim.

## Typed payloads

`ework:payloads` carries structured content the task is about, specified in the
typed payloads document. A payment payload carries amount, due date, and
beneficiary as fields, not as prose to be parsed.

## Attachments

`ework:attachments` references binary content by content hash:

~~~
{
  "id": "att-3f9a1c",
  "name": "contract.pdf",
  "mediaType": "application/pdf",
  "size": 284913,
  "hash": "sha256:8fe99cb6...",
  "blob": "sha256:8fe99cb6...",
  "enc": { "alg": "A256GCM", "keyWrapped": "..." },
  "kind": "document",
  "inline": null
}
~~~

Bytes travel outside the envelope, raw, never base64 inside JSON. The reason is
arithmetic: base64 inflates by one third on every hop.

Addressing by content hash gives deduplication for free and makes the attachment
verifiable: a receiver that recomputes the hash knows it received what the task
referenced. Implementations MUST verify the hash on receipt. **Knowing a hash is
not authorisation**, and the security considerations of the federation document
specify what is.

`enc` is present under end-to-end encryption: the stored blob is AEAD
ciphertext, and the key travels wrapped inside the Task, which only members of
the group can read. `kind` guides the UI: `document`, `image`, `audio`, `data`
for raw structured data, `proof` for a receipt, and the vocabulary is
extensible. `inline` MAY carry tiny values, up to 4 KiB in base64, for cases
like a vCard; above that, a blob.

## Action targets

A `deeplink` target MUST use a scheme from the registered list (`https`, `pix`,
`tel`, `mailto`, `geo`), and clients MUST discard an action whose scheme falls
outside it while preserving the rest of the task. `javascript:`, `data:`, `file:`
and `intent:` MUST NOT be actioned under any circumstances, and web clients MUST
treat the target as data rather than as a navigation attribute built from
sender-supplied text. An action whose target points outside the issuer's verified
domain MUST be presented with an explicit warning, and payment actions follow the
stricter rule in the payloads document, which binds the target to the payload and
blocks on mismatch rather than warning.

## Actions and execution policy

`ework:actions` lists the structured actions the UI offers on this task,
inspired by ntfy and Adaptive Cards, without the gatekeeping:

~~~
{ "id": "pay", "kind": "deeplink", "label": "Pay at the bank",
  "target": "pix://qr/00020126...", "primary": true }
{ "id": "view", "kind": "view", "label": "View invoice", "target": "att-1" }
{ "id": "copy", "kind": "copy", "label": "Copy code",
  "target": "payload:payment.pix.brcode" }
{ "id": "approve", "kind": "respond", "label": "Confirm attendance",
  "respond": "accept" }
~~~

Core kinds: `view` opens an attachment or payload, `deeplink` opens an external
URI, `http` performs a simple GET or POST to an issuer URL, `copy` copies a
value, `respond` generates a response envelope, and `share` shares content.
Actions are always executed with the user's confirmation, NEVER automatically;
`http` and `deeplink` actions MUST display their destination; and clients MUST
ignore unknown kinds with graceful fallback.

`ework:execution` carries the execution policy, including whether human
approval is required, specified in the autonomous actors document.
`ework:machine` carries the acceptance criterion and the structured result for
execution without a human, also specified in the autonomous actors document.

## Deduplication

`ework:dedupKey` identifies a series. Redelivery under the same key updates the
existing task instead of creating a second one. This is what stops September's
bill becoming two tasks because the issuer's system retried after a timeout.

Issuers SHOULD use the document's natural identity as the key, for example
`energy-2026-09`, rather than a fresh UUID per attempt, which defeats the
mechanism.

## Project membership

A project's tasks are ordinary Tasks carrying `ework:projectUid` with the
project's identifier. The epic itself MAY have an umbrella Task, related by
`parent`, for the deadline and the aggregated progress. The Project object is
specified in the compartments document.

# Security Considerations

The task object is signed as part of the envelope that carries it. A task
modified in the recipient's store is detectable only through the entry chain of
the history document, not through the task object itself, because the task is
mutable state by design while history is not.

`ework:origin` is asserted by the sender. A receiver MUST NOT treat it as more
trustworthy than the envelope's signature chain: it records what the sender
claims about provenance, and the signature records who actually sent it.

A dependency graph spanning parties cannot force the reading of invisible
nodes: where the other end of a relation is not visible to the reader,
actionability degrades to minimal metadata.

# Privacy Considerations

Field-level classification, in `ework:classification`, and sealed sections, in
`ework:sealed`, are specified in the compartments document. Their presence in
this model is what allows a single task to have different shapes for different
readers, rather than requiring a separate task per audience.

`ework:statusSharing` and the consent policy limit what the `issuer` receives;
`keywords` and the user's free-form fields NEVER leave the user's box toward
the issuer.

# Open Questions

1. Are subtasks through `parent` and `child` enough, or is a lightweight
   checklist object inside the Task, JSCalendar style, worth having?
2. A normative maximum of payloads and attachments per task?
3. The `kind` vocabulary for attachments: closed, or open with a registry?
