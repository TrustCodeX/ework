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
  "ework:projectUid": "019fd0...",
  "ework:compartment": "general",
  "ework:classification": { },
  "ework:sealed": [ ],
  "ework:escalation": { },
  "ework:updatesMilestone": null
}
~~~

## Inherited fields

`uid`, `title`, `description`, `created`, `updated`, `due`, `timeZone`,
`priority`, `progress`, `percentComplete`, `estimatedDuration`,
`recurrenceRules`, `participants`, and `relatedTo` carry their JSCalendar
{{RFC8984}} semantics unchanged.

`progress` takes the values `needs-action`, `in-process`, `completed`,
`cancelled`, `failed`, plus two EWP additions: `blocked` and
`awaiting-confirmation`.

`priority` is 0 to 9 and expresses importance. It is a different axis from
`ework:urgency`, which expresses demand on attention. A utility bill can be
important and not urgent; conflating them produces either alarm fatigue or
missed deadlines.

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
  "kind": "document"
}
~~~

Bytes travel outside the envelope, raw, never base64 inside JSON. The reason is
arithmetic: base64 inflates by one third on every hop.

Addressing by content hash gives deduplication for free and makes the attachment
verifiable: a receiver that recomputes the hash knows it received what the task
referenced. Implementations MUST verify the hash on receipt. **Knowing a hash is
not authorisation**, and the security considerations of the federation document
specify what is.

## Actions and execution policy

`ework:actions` lists the actions offered on this task. `ework:execution`
carries the execution policy, including whether human approval is required,
specified in the autonomous actors document.

## Deduplication

`ework:dedupKey` identifies a series. Redelivery under the same key updates the
existing task instead of creating a second one. This is what stops September's
bill becoming two tasks because the issuer's system retried after a timeout.

Issuers SHOULD use the document's natural identity as the key, for example
`energy-2026-09`, rather than a fresh UUID per attempt, which defeats the
mechanism.

# Security Considerations

The task object is signed as part of the envelope that carries it. A task
modified in the recipient's store is detectable only through the entry chain of
the history document, not through the task object itself, because the task is
mutable state by design while history is not.

`ework:origin` is asserted by the sender. A receiver MUST NOT treat it as more
trustworthy than the envelope's signature chain: it records what the sender
claims about provenance, and the signature records who actually sent it.

# Privacy Considerations

Field-level classification, in `ework:classification`, and sealed sections, in
`ework:sealed`, are specified in the compartments document. Their presence in
this model is what allows a single task to have different shapes for different
readers, rather than requiring a separate task per audience.
