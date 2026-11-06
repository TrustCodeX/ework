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

The levels map to ntfy's priorities 1 to 5, to the Web Push `Urgency` field
(the normative mapping is in the sync document), and to Android and iOS
notification channels.

**The sender proposes urgency; the receiver holds the ceiling.** Every
relationship carries one: the scope's `maxUrgency` for an issuer, and the
`Relation` object's for a person, which starts at `normal`. Hosts and clients MUST
lower any urgency above the ceiling, and the ceiling MUST NOT be raised by the
sender under any circumstances. Without that ceiling, urgency becomes a field
every marketer sets to maximum.

The two lower different things, and the client is the one that always holds:
`ework:urgency` travels inside the encrypted task, so the host reaches only the
`urgencyHint` in the header, which is optional and starts off. With the hint on,
an issuer granted `maxUrgency: normal` that declares `critical` MUST be refused
at the edge; with it off, the edge has nothing to read, and the ceiling is
applied by the recipient's client, over the real value.

The ceiling previously existed only inside the consent scope, and a relationship
between people creates no consent object, so after first contact `critical` was
free forever. Since `critical` means maximum alert, piercing do-not-disturb,
requiring acknowledgement and triggering escalation, that amounted to a
protocol-blessed instrument of night-time harassment available to any ex-partner
once accepted.

Urgency belongs to the attention axis; `priority`, specified in the data model
document, belongs to the importance axis. They are independent: a bill can be
important and not urgent.

# Acknowledgement Is Not Completion

- Action `ack` means "I am aware". It pauses re-notification and escalation. It
  does **not** complete the task.
- **An outcome ends the alert cycle of the occurrence**: the task reaching
  `completed`, `failed` or `cancelled`, through `complete`, `confirm`, `fail`,
  `cancel` or `decline`. What counts is the state the task reached, not who
  signed the entry: the owner, an executor who is not the owner, or the issuer
  speaking about its own offer across the federation edge. An offer the issuer
  cancelled must not keep waking the owner and the owner's contacts. A
  `complete` that leads to `awaiting-confirmation` is not an outcome yet, as
  the confirmation item below says.
- **`ack` pauses the cycle of whoever the notification reached**: the owner,
  whom step 0 reaches on the owner's own device, and the contacts of the
  following steps. The issuer is never among them, not even when executing: the
  escalation is armed in the owner's box, and the steps name the owner's device
  and addresses of the owner's personal relationships, never the relationship
  with the issuer. The issuer's `ack` therefore MUST NOT pause the escalation,
  and MUST be recorded as an entry without effect. What ends the cycle coming
  from the issuer is an outcome.
- **Execution confirmation is a distinct thing**, specified in the autonomous
  actors document: acknowledgement is "I saw it", confirmation is "it was done,
  and someone attests to that". A critical task MAY require both, and in that
  case sitting in `awaiting-confirmation` is a valid escalation trigger (the
  medication was marked as taken, but the caregiver has not yet confirmed).

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
      "channels": ["push"] },
    { "notify": "k7f3a2q9@ework.example", "after": "PT30M",
      "channels": ["push", "call-gateway"] }
  ]
}
~~~

- Step 0 is always the task's owner, on the owner's own device, and in a
  personal task the owner is also the executor. When the executor is the
  issuer, the escalation still belongs to the owner: the owner is the one who
  needs to know that nobody acted. Without an acknowledgement within
  `ackTimeout`, the policy advances one step. `repeat` bounds complete cycles,
  so escalation cannot become a perpetual alarm.
- **An escalation contact requires prior consent, and the consent is the
  relationship that already exists.** A contact relationship between people is
  born through the personal path of the consent document, with an MLS group of
  its own, and escalating is a permission inside it: the `Relation` object
  carries `escalation`, which starts **false** and which only whoever accepted
  the relationship raises. Without that permission the step is invalid, and the
  contact MAY revoke it at any time, together with the relationship or
  separately.

  There is no `escalation.invite`, and there will not be one. The handshake it
  would establish is already the one of the personal path, and creating it
  would cost metadata: the envelope type travels in clear text in the header,
  which is why the core document refused separate types for accepting,
  declining and completing. An emergency-invitation type tells the host more
  than any of those.
- **The step names the address of that relationship, never the public handle.**
  The policy travels inside the task, so a handle there would hand the
  contact's real address to everyone who can see the task, including the
  issuer who sent it.
- **The permission is enforced at the receiving contact's edge**, not by the
  arming host. Escalation is delivery, and delivery without consent is already
  refused at the edge with `unknown-recipient`, as specified in the federation
  document. The arming host MUST NOT require a capability: a personal
  relationship issues no send capability.
- **Step 0 (`notify: "self"`) is always valid** and requires no relationship:
  it is the person themselves, on their own device. It MUST be armed like any
  other step, and skipping it makes the escalation start at the wrong step.
- What the contact receives is the title, the urgency, and the link ("critical
  task of X, unacknowledged for 15 minutes"), **not** the full content, unless
  the task's owner configured a more open policy. The contact was called to
  help, not to read everything.
- `call-gateway` (a voice call with text-to-speech) is a registered extension,
  outside the v0.1 core.
- Durations are ISO 8601. An implementation that cannot parse a duration MUST
  fail rather than default: a silently zeroed `ackTimeout` escalates
  immediately, to everyone, in the middle of the night.

Configuring a step for a contact who has not granted anything is accepted, and
the first delivery fails at their border. Clients MUST show that failure on the
task: discovering at three in the morning that the escalation was never going
to work is the worst possible time. Warning in advance would require an
endpoint answering "does this person accept escalation from you?", and that
endpoint is an oracle.

# Escalation With a Blind Host

The mechanism is designed for a host that cannot read content.

On arming a step, the client pre-composes the envelope for the contact,
encrypting it to that contact's group where end-to-end encryption is in use, and
deposits it with the host together with a `releaseAt` time. The host releases it
at that time unless it receives a cancellation first: an entry carrying `ack`
arrived from the owner, or the task reached an outcome through an entry from
any participant, including the issuer's entry arriving across the federation
edge. Any of the owner's devices MAY issue the cancellation. The owner's
account deletion window is the exception to the release time: a step that falls
due inside it is not released, then or after cancellation, as the compliance
document specifies, because the frozen account cannot give the `ack` that would
silence it, and releasing it later would alarm the contact about a moment that
has already passed.

The host learns only the pair (destination, release time). It learns neither
content nor reason, and that metadata appears in the exhaustive list of the
cryptography document.

The case "all the owner's devices are dead" is covered by construction: the
queue fires anyway, which is exactly the situation escalation exists for. The
owner's own devices keep their local alarms, specified below, as the first
line, and the assisted mode stops being necessary for escalation.

**The host MUST validate the pre-composed envelope before accepting it into the
queue**, because it is the host that applies the transport signature at release
time. Without validation the queue is blank letterhead: the client declares a
`from` belonging to another account at the same host, the host stamps the
domain's signature over it, and at the far end the signature checks out against
the declared sender. Three minimum bindings: `from` MUST be an address of the
requesting identity, `to` MUST be that step's destination, and the capability
MUST be the one issued for that relationship, not an arbitrary string.

# Urgency, Push and the Metadata Trade-off

Platform push is sometimes handled at low priority by the operating system when
the payload does not indicate urgency. Hence `urgencyHint` in the envelope
header, in clear text: enabling it improves delivery of `critical` with the app
dead, at the cost of the host and the push service seeing that "something
critical went past".

The default is **disabled**, and the escalation interface SHOULD offer the
choice with the explanation attached. With it disabled, `critical` depends on
local scheduling: reliable for recurring alarms, more fragile for an ad-hoc
critical with every app dead. The push subscription, and what the `Urgency`
header reveals even with the hint off, are in the sync document.

# Local Alarms

Clients MUST schedule alarms for known tasks on the device's local clock,
independent of push and of the network. Push accelerates news; it is not the
source of the alarms. The corollary is that the 8am medication rings with no
internet at all.

Alarms for the owner's own devices fire **locally**, from the client, not from a
server push. A client that has the task has everything it needs to alarm, and
routing that through a server would tell the server which tasks are critical and
when they are due.

Server push remains available for the case the local alarm cannot cover, which
is a device that is not running the client. The trade-off is stated in the
cryptography document: a push carrying urgency reveals urgency to the push
provider.

# Security Considerations

Urgency is a vector of psychological pressure in scams: `critical` from an
issuer requires the `maxUrgency: critical` scope in the consent, granted
through a dedicated screen of its own, and clients MUST display who asked for
the urgency. An issuer that abuses `high` or `critical` beyond its scope is
lowered to the ceiling by the recipient's client, always, and refused at the edge
with `no-consent` whenever the `urgencyHint` gives it away. What the defence
never does is depend on the hint: it is opt-in, and a protection against scams
that held only for people who enabled an extra piece of metadata would protect
exactly those who need it least.

Escalation is the recipient's safety net, and only those it reaches pause it.
Letting the issuer's `ack` pause it would give the offering party a way to
silence the owner's alert without doing anything, and the case where that
matters is exactly the one escalation exists for: whoever should have acted did
not. What comes from the issuer and ends the cycle is an outcome, which stays in
the history under the issuer's signature.

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

# Open Questions

1. Semantics of delayed release: the host went down at 8:14 and came back at
   8:40; release late, drop, or release with a lateness mark? Proposal: release
   with the mark, and the contact's client displays the delay.
2. Limits on `critical` per day and per issuer, as a protocol norm or as client
   policy.
3. Integration with the operating systems' focus regimes (Focus and DND APIs)
   as implementation guidance.
4. How an escalation contact's `ack` comes back. The contact is among those the
   notification reaches, and is not a participant of the task, so it signs no
   entry on it. It remains to be specified how that acknowledgement reaches the
   owner's queue, and whether it pauses the whole cycle or only the steps that
   reach the contact.
