# ADR-0023: Escalation is a permission on a relationship that already exists, not a new handshake

**Status:** accepted | 2026-08-08 | Michel Wilhelm

## Context

EW-RFC 0009 §3 requires prior consent from the escalation contact and names the mechanism: "an `escalation.invite` invitation accepted once establishes the relationship (with its own MLS group); without it the step is invalid".

That envelope type does not exist. It is not in the core registry of EW-RFC 0001 §6, no RFC defines its body, and the host does not accept it. The consequence was measured while trying to build the screen: `Task/arm` **arms zero in every reachable configuration**. A step with `notify: "self"` is skipped by the code, and a step for another person has no way of holding valid consent, because the relationship it requires has no way of being born.

An inconsistency of direction came with it. The host's validation looks, in the consents table, for a row where `did` is the person arming and `alias` is the step's destination. But `alias` there is the address I issued to the counterparty, so the step would have to name an address of mine, and not the contact. Either the query is inverted, or the escalation relationship model is a different one and was never written down.

Two things already existed when this decision was taken, and it is their existence that changes the answer.

The first: EW-RFC 0007 §3.5 already establishes a relationship between people through a `consent.request` from a personal identity, with its own MLS group, and §3.6 already defines the `Relation` object that the accepting party keeps, with `alias`, `maxUrgency` and revocation with immediate effect at the edge. The `escalation.invite` describes that same handshake, under another name.

The second: EW-RFC 0001 §6 explains why accept, decline and complete did NOT become separate envelope types, and the reason is privacy, not tidiness: **the type travels in the clear in the header**. Separate types would tell the host what happened to the task. An `escalation.invite` tells the host that this person is setting up an emergency contact, which is sensitive information about their life and which no other handshake hands over.

## Decision

**Escalating is a permission on a relationship that already exists, and not a new relationship.**

1. `escalation.invite` **is not created**. EW-RFC 0009 §3 comes to cite EW-RFC 0007 §3.5: the contact relationship between people is the same one, and escalation is what is granted within it.
2. The `Relation` object (EW-RFC 0007 §3.6) gains `escalation`, a boolean, born **false**. Only whoever accepted the relationship raises it, by the same principle as the `maxUrgency` already there: whoever suffers the consequence decides.
3. The escalation step names the **address of that relationship**, and never the public handle. The policy travels inside the task, so a handle there would hand the contact's real address to everyone who sees the task, including the issuer that sent it.
4. **Enforcement is at the receiving edge**, not at the arming host. Escalation is delivery, and delivery without consent is already refused at the edge: a step for someone who did not grant it fails there, with the same answer as always. The arming host does NOT validate a credential, because in the personal model there is no sending credential (§3.5), and the validation it was doing was precisely the inverted query.
5. Step 0, `notify: "self"`, is always valid and requires no relationship: it is the person themselves, on their own device.

## Alternatives considered

**Create `escalation.invite` as the RFC describes.** Rejected for two reasons together. First, the handshake it would establish already exists in §3.5, with its own MLS group and relationship object: there would be two paths to the same thing, and the second would have to reimplement quarantine, silence, rate limits, expiry and revocation. Second, and decisively, the type travels in the clear: the registry of §6 avoided separate types precisely so the host would not learn what happened, and an emergency-invitation type is more revealing than any of the ones rejected there.

**Name the contact with no consent at all, only through the `escalation-contact` role of EW-RFC 0002 §2.** Rejected. It is the harassment tool the security review of 2026-08-08 describes: anyone names anyone and starts piercing their silent mode in the middle of the night, in series. The prior consent of §3 exists against that, and this decision keeps it, changing only where it comes from.

**Validate at the arming host.** Rejected because there is nothing to validate: the personal relationship issues no credential, and the state that authorises lives on the receiving side. Validating on the wrong side is what produced the inverted query. Failing at the edge is the same path as every delivery, and it is where the data is.

**Permission per task instead of per relationship.** Rejected. Granting per task would mean asking again for every critical task, which in practice becomes "grant always", and consent stops meaning anything. Per relationship it is a conscious decision, revocable in one go, and in the same place where the person already controls that contact's urgency ceiling.

## Consequences

`Task/arm` starts arming for real, including step 0, which was being skipped.

The escalation configuration screen offers only addresses of relationships the person already has, because there is no way to escalate to someone who is not a contact. That is a real limitation and it is the right one: escalating to a stranger is the abuse §3 refuses.

Whoever configures a step for a contact who has not granted it yet **does not find out at that moment**. The configuration is accepted, and the first delivery fails at the contact's edge. Clients MUST show that failure on the task, and not silently: finding out at three in the morning that the escalation was never going to work is the worst possible time. There is no way to anticipate it without creating an endpoint that answers "does this person accept escalation from you?", and that endpoint is an oracle.

The relationship's `maxUrgency` still applies on top: granting escalation does not raise the urgency ceiling, and the two are regulated separately because one is "you may wake me" and the other is "you may wake me on someone else's behalf".

It remains open whether the contact should be notified when someone names them in a step, before any escalation happens. Notifying gives them the chance to revoke before the emergency; not notifying avoids a new channel of unsolicited message inside an already-established relationship. EW-RFC 0009 records the question.
