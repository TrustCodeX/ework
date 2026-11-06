# ADR-0025: The composed policy is published, and the private one judges no one else's entry

**Status:** accepted | 2026-08-11 | Michel Wilhelm

## Context

EW-RFC 0014 §3 requires the issuer's and the recipient's execution policies to be composed field by field, strictest wins, and states that the recipient's policy lives in `ework:executionPolicy`, in their own box and never in the object that arrives. The rule exists for the most ordinary case of all: whoever receives an offer wants to be able to harden what arrived, not merely swallow what the other party declared.

The composition was written and was not implemented. Item `lacuna-15` of the security review inventory had said so on 2026-08-08, in these words: "the strictest prevails" is not computable, because the recipient's policy does not exist in the data model and the ordering is not total. The field-by-field table of §3 closed the ordering. The rest was missing.

While implementing it, open question 6 of EW-RFC 0015 pointed at the design problem the table alone did not solve. `messageRequired` composes as the union of the two lists, and one of the lists is private. So whoever writes satisfies everything they could see and still produces, at the other end, a void entry for a requirement they had no way of knowing.

The worst case is not the wasted record, and it is the one that decided this ADR. The recipient's private policy forbids agents; the recipient's own agent, running a modified client, signs `complete`. The issuer applies it, because the policy the issuer can see allows it. The recipient does not. For one of them the bill is paid and for the other it is not, and neither has any way of knowing they disagree. EW-RFC 0015 §6 requires the history to be the same for everyone, and an input kept in one side's box breaks that by construction: state stops being a function of the shared log.

## Decision

**A requirement nobody can see binds nobody.** The private policy is an input to the composition, never a judge.

1. `ework:executionPolicy` stays in the recipient's box and stays out of the object that arrives. What changes is what it can do on its own: nothing against someone else's entry.
2. Whoever hardens publishes the result. The composed policy enters the task through an entry carrying the `harden` action (EW-RFC 0015 §2), signed by the `owner`, in the group of the participants it binds. A composition that hardens nothing is not published, and publication happens no later than acceptance, which is when it starts to bind the other end.
3. An entry is judged by the policy published in its **causal past**. Hardening published afterwards does not reach an earlier entry, and the author's `after` proves what they knew.
4. The private policy binds at **signing** time: the recipient's client, and their own host, do not sign or publish an entry that violates it. It is the first of the two moments of EW-RFC 0015 §4, applied to the whole policy.
5. `harden` only tightens. A `harden` entry that loosens the published policy is a void entry (EW-RFC 0015 §2.2), and any participant checks that alone, because both policies are in the history.
6. `harden` is not status: the status sharing policy (EW-RFC 0007 §6) does not filter it. A requirement that the recipient's host could choose not to deliver would be an invisible requirement again.

## Alternatives considered

**Let the private policy judge, and accept the void entries.** That is what a literal reading of §3 allowed, and what was about to be implemented. Rejected because of the state divergence described above: the damage is not the wasted record, it is two peers disagreeing about whether the task was completed, with no signal at all. The protection the person believes they have does not exist, because the other side applied the entry anyway.

**Announce the whole private policy inside the relationship**, which is the way out that open question 6 of EW-RFC 0015 suggested. Rejected for costing more privacy than it needs to: the recipient's rule covers every issuer and every payload, and announcing it tells the bank what it demands of the school. The projection of that rule onto that task is enough to bind, and it is the minimum that is enough.

**Compose on the recipient's host, at delivery, silently rewriting the task's `ework:execution`.** Rejected twice over. First, because under the end-to-end encryption of ADR-0004 the host does not read the policy, so the rule would vanish in the encrypted deployment, which is the default. Second, because rewriting the object with no entry recording the rewrite is what EW-RFC 0015 §7 and §10 forbid: the history would stop saying who hardened, when, and with what justification.

**Require the issuer to ask for the recipient's policy before offering.** Rejected for turning what is one envelope into a mandatory round trip, and for creating an oracle: asking someone's policy before having a relationship with them reveals that the box exists, which is what EW-RFC 0003 §7.4 closes.

**A new state, "entry pending policy", instead of a void entry.** Rejected for inventing a machine where ADR-0021 had already decided: an entry that does not count is a void entry, recorded and visible. Two ways of refusing the same thing double what clients must know how to display.

## Consequences

The composition of EW-RFC 0014 §3 becomes checkable by third parties: both parts are in the history, so recomposing is the verification, and whoever exports the task (EW-RFC 0015 §8) carries with it the proof of which policy applied to each entry.

The issuer now sees the projection of the recipient's policy onto that task. That is the cost, and it is declared in EW-RFC 0014, under privacy considerations: the projection, never the rule, and nothing at all when the composition hardens nothing.

A new action appears in the vocabulary of EW-RFC 0015 §2, and every new action is weight: clients must know how to display it, and the conformance suite must cover it. That is the price of the policy being a recorded act instead of a side effect.

Hardening stops being silent. Anyone who wants to demand without telling still can, and what they get is a requirement that holds only for their own keys, which is exactly what an unannounced requirement always was.

Left open, in EW-RFC 0015 itself: who besides the `owner` hardens a project task, and what to do when two hardenings overlap.
