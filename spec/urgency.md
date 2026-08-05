---
docname: draft-wilhelm-ework-urgency-00
title: "The e-work Protocol (EWP): Urgency and Escalation"
abbrev: EWP Urgency
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9420
---

This document specifies urgency levels and escalation for the e-work Protocol
(EWP). Urgency governs demand on attention, which is a different axis from
priority, which governs importance.

For tasks that cannot simply sit unattended, the document specifies an
escalation mechanism that works with a host unable to read content: the client
pre-composes the envelope for the escalation contact and deposits it with a
release time, and the host releases it unless an acknowledgement arrives first.

<!-- abstract -->

# Introduction

Attention is the scarce resource a task protocol spends. A protocol with a
single notification behaviour spends it badly in both directions: everything
interrupts, so the user disables notifications, and then nothing interrupts.

## Conventions and Definitions

<!-- rfc2119 -->

# Urgency Levels

`ework:urgency` is one of `min`, `low`, `normal`, `high`, `critical`.

| Level | Normative client behaviour |
|---|---|
| `min` | No notification; appears in lists |
| `low` | Silent, grouped notification |
| `normal` | Standard notification |
| `high` | Prominent notification, re-notified until interaction |
| `critical` | Maximum alert, may pierce do-not-disturb where the platform permits, **requires acknowledgement**, triggers escalation |

Urgency is subject to the consent scope: an issuer granted `maxUrgency: normal`
that sends `critical` MUST be refused at the edge. Without that ceiling, urgency
becomes a field every marketer sets to maximum.

Urgency belongs to the attention axis; `priority`, specified in the data model
document, belongs to the importance axis. They are independent: a bill can be
important and not urgent.

# Acknowledgement Is Not Completion

- Action `ack` means "I am aware". It pauses re-notification and escalation. It
  does **not** complete the task.
- Actions `complete` and `fail` are outcomes. They end the alert cycle.
- **Execution confirmation is a distinct thing**, specified in the autonomous
  actors document: acknowledgement is "I saw it", confirmation is "it was done,
  and someone attests to that".

Conflating acknowledgement with completion makes the alert stop when someone
merely glanced at a screen, which is the failure mode this separation exists to
prevent.

For recurring tasks, each occurrence has its own cycle: tomorrow's eight o'clock
medication is born clean. `dedupKey` plus the occurrence identify the cycle, so
a reminder resent by the issuer does not create a duplicate alert.

# Escalation Policy

~~~
"ework:escalation": {
  "ackRequired": true,
  "ackTimeout": "PT15M",
  "repeat": 2,
  "steps": [
    { "notify": "self", "channels": ["push", "local-alarm"] },
    { "notify": "k7f3a2q9@ework.example", "after": "PT15M",
      "channels": ["push"] }
  ]
}
~~~

- Step 0 is always the executor themselves. Without an acknowledgement within
  `ackTimeout`, the policy advances one step. `repeat` bounds complete cycles,
  so escalation cannot become a perpetual alarm.
- **An escalation contact requires prior consent.** A relationship established
  once, through the ordinary consent path, is what makes a step valid. The
  contact MAY revoke at any time.
- What the contact receives is the title, the urgency, and the link ("critical
  task of X, unacknowledged for 15 minutes"), **not** the full content, unless
  the task's owner configured a more open policy. The contact was called to
  help, not to read everything.
- Durations are ISO 8601. An implementation that cannot parse a duration MUST
  fail rather than default: a silently zeroed `ackTimeout` escalates
  immediately, to everyone, in the middle of the night.

# Escalation With a Blind Host

The mechanism is designed for a host that cannot read content.

On arming a step, the client pre-composes the envelope for the contact,
encrypting it to that contact's group where end-to-end encryption is in use, and
deposits it with the host together with a `releaseAt` time. The host releases it
at that time unless it receives a cancellation first, which an entry carrying
`ack` produces, and which any of the owner's devices MAY issue.

The host learns only the pair (destination, release time). It learns neither
content nor reason, and that metadata appears in the exhaustive list of the
cryptography document.

The case "all the owner's devices are dead" is covered by construction: the
queue fires anyway, which is exactly the situation escalation exists for.

**The host MUST validate the pre-composed envelope before accepting it into the
queue**, because it is the host that applies the transport signature at release
time. Without validation the queue is blank letterhead: the client declares a
`from` belonging to another account at the same host, the host stamps the
domain's signature over it, and at the far end the signature checks out against
the declared sender. Three minimum bindings: `from` MUST be an address of the
requesting identity, `to` MUST be that step's destination, and the capability
MUST be the one issued for that relationship, not an arbitrary string.

# Local Alarms

Alarms for the owner's own devices fire **locally**, from the client, not from a
server push. A client that has the task has everything it needs to alarm, and
routing that through a server would tell the server which tasks are critical and
when they are due.

Server push remains available for the case the local alarm cannot cover, which
is a device that is not running the client. The trade-off is stated in the
cryptography document: a push carrying urgency reveals urgency to the push
provider.

# Security Considerations

Escalation is a mechanism for reaching a third party, so it is a mechanism an
attacker would like to abuse. The requirements that the contact has consented,
that the envelope is validated before queueing, and that `repeat` is bounded are
what keep it from becoming an amplification channel.

`urgencyHint` in the envelope header travels in clear text and is deliberately
coarse. A precise urgency in the header would tell every host on the path how
important each message is, which is a traffic-analysis gift.

# Privacy Considerations

The escalation contact learns that the owner has a critical task and that it
went unacknowledged. That is unavoidable, since it is the information that makes
the contact useful, and it is why the relationship requires consent from the
contact rather than being assignable unilaterally.
