# ADR-0029: The issuer may propose itself as executor, and acceptance is what makes it one

**Status:** accepted | 2026-09-10 | Michel Wilhelm

## Context

EW-RFC 0002 §2 defined the roles of a task (`owner`, `executor`, `issuer`, `observer`, `escalation-contact`) and left it written that, in a personal task, executor and owner coincide, with transferring execution to a third party marked as a "registered future extension". ADR-0021 tied each action to a role: `start`, `complete` and `fail` belong to the `executor`, and the issuer, which holds only `issuer`, signs none of them.

That serves the founding scenario, the bill: the company offers, the person accepts, the person pays. It does not serve a scenario that showed up on 2026-09-10, in the first real use of an in-house issuer: an agent proposes an operational action (merge a pull request, restart a service, apply an investigated plan), the person approves or declines, and **the one who executes is the agent**. The task is hers, the decision is hers, and the work is on the other side. The protocol had no way to say so.

What exists today to cover the case are two workarounds, and neither is the right instrument:

- `maySign` (EW-RFC 0015 §2.2) widens authority per task, but it is the `owner`'s declaration, and arriving inside the offer it is discarded, rightly: the issuer cannot write its own authorisation. What remains is asking the person to widen authority on every task, in a field no client exposes.
- `reassign` transfers execution, but only after acceptance, as a second act, and also without a user interface. The intent "approve it, and I will do it" would become two taps, the second of them somewhere nobody finds.

And there was a hole on the way: the reference host stored the `participants` arriving in the offer **unfiltered**, and that map is where roles come from when it exists. An issuer writing itself as `owner` in the offer signed its own `accept`, which is exactly the record ADR-0021 exists to prevent. The hole was closed by luck: no reference issuer sent `participants`.

## Decision

**The offer MAY propose the issuer as executor, and the proposal only binds after the owner accepts.**

1. **Only the issuer, and only executor.** `participants` in an offer may only contain the issuer's own identity, with roles within `issuer` and `executor`. Receivers MUST discard any other participant or role arriving in an offer, in particular `owner` for anyone at all. Whoever receives is always `owner`, and is also `executor` unless the offer proposed the issuer.
2. **A proposal is not a role.** Until `accept`, the issuer holds only `issuer`. Its entry with `start`, `complete` or `fail` before acceptance is a void entry, for lack of role. Acceptance ratifies exactly what was in the accepted offer.
3. **A revision that changes the executor requires a new acceptance.** A revision by `dedupKey` that adds or removes the proposed executor MUST be treated like the revision that changes the payment instrument (EW-RFC 0007 §5): the negotiation returns to pending.
4. **The owner never loses execution.** `reassign` remains the `owner`'s, and `reassign` with `to` pointing at itself reclaims execution from a proposed executor, at any time.
5. **An executor who is not the owner hands the task back for confirmation.** When the `executor` is not the `owner`, and the execution policy declares no `confirmedBy`, the default becomes `["owner"]`: the executor's `complete` takes the task to `awaiting-confirmation`, and the owner confirms or contests (EW-RFC 0014 §5). The `executor` default of EW-RFC 0014 §3 only holds when whoever does the work is the owner, because then there is no third party to protect.

The federated edge starts accepting entries in the issuer-to-recipient direction (comment, `start`, `complete`, `fail`, `reschedule`, `update`), with the same relationship and role checks that already applied the other way.

## Alternatives considered

- **`maySign` declared by the owner in the client.** Rejected: it places on the person, on every task, a decision the issuer already knew when offering, and requires a screen that does not exist. It remains for the arrangement the default does not anticipate.
- **`reassign` after acceptance.** Rejected as the main path for the same reason: two acts for one intent. It stays as the way to reclaim, which is where it is irreplaceable.
- **Executor implied by payload type** (every `job@1` or `approval@1` executed by the issuer). Rejected: a role decided by payload type is invisible on screen, does not compose with `reassign`, and makes authority depend on a vocabulary that grows without migration.
- **Delegation from EW-RFC 0014 §2.** Rejected for being something else: delegation is the person giving a credential to her **own** agent. Here the executor is the **other end of the relationship**, and what bounds it is the consent, which already exists, not a new credential.
- **Leave it as is and let the agent only comment.** Rejected: the task state would stop saying what happened, and the timeline, which is the product, would show "accepted" for work completed days ago.

## Consequences

- The human keeps deciding at both ends: accepts before, confirms after. An agent of class `autonomous` never produces the confirmation, because it requires the owner's role (§5 above) and, when the policy asks, a `human` key (EW-RFC 0014 §4).
- Status returns to the issuer through the relationship policy as before; `completed` is already a milestone.
- Reference hosts: filter `participants` on offer arrival, resolve the issuer's identity to the form the receiver uses when checking authorship (ADR-0022), accept `task.entry` in the issuer-to-recipient direction, and apply the `confirmedBy` default of §5.
- Clients: show who executes ("done by: claude@imake.codes"), hide the executor's buttons from the owner when the owner is not the executor, and offer confirm and contest in `awaiting-confirmation`. SDKs: a way to propose the executor in the offer.
- The "future extension" sentence of EW-RFC 0002 §2 now applies only to transferring execution to someone **outside** the relationship, which remains without a defined form.
