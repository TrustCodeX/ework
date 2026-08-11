# ADR-0021: Authority per action, separate from the authenticity of the signature

**Status:** accepted | 2026-08-08 | Michel Wilhelm

## Context

EW-RFC 0015 defined the entry as the unit of history and listed the action vocabulary in a three-column table: type, effect and parameters. The fourth was missing, and the gap was in no sentence: it was in the column the table did not have.

The security review of 2026-08-08 showed what that allowed. Inside a group they belonged to by right, any participant signed any transition, and the resulting entry was valid by every criterion the specification enumerated: the signature verified, `after` closed the chain, and the action was compatible with the current state, which was the only filter written down (EW-RFC 0004 §4.1). Every conforming client applied it.

In practice: the issuer signed `accept` on its own offer and produced a signed record that the charge had been accepted, usable in a dispute. The supplier signed `complete` to unblock its own delivery, `reassign` to capture someone else's execution, `cancel` on another person's task, `update` rewriting the deadline and the amount. EW-RFC 0015 §8 presents the list of signed entries as the audit artefact, and it was exactly that artefact which was being forged.

The only statement of authority in the whole specification was in EW-RFC 0015 §2.1 and covered `attach` alone. The defences RFC 0015 already had, hash chaining and immutability, protect against erasing the past and say nothing about writing in the present in someone else's name.

## Decision

**Authenticity and authority are separate questions, and both are verified.** The signature answers who wrote it. The role answers whether that person was allowed to write it.

1. The action vocabulary of EW-RFC 0015 §2 gains the normative column "Who MAY sign", with a default role per action, taken from the `Participant` vocabulary that EW-RFC 0002 §3 already defined (`owner`, `executor`, `issuer`, `observer`, `escalation-contact`).
2. Receivers resolve the author in the current task's `participants` and check the role before applying the effect.
3. An entry without authority becomes an **entry without effect**: it stays in the history, with author, message and signature, and changes no state. It is the same fate EW-RFC 0004 §4.1 already gave to an action incompatible with the state, and for the same reason.
4. The `owner` MAY widen the authority of an action per task, in `ework:execution.maySign`, and MAY NOT narrow it below the default nor assign themselves a role they do not have.
5. Seeing the compartment authorises commenting, and never transitioning.

## Alternatives considered

**Leave it as it was, treating authority as the client's responsibility.** Rejected because the specification is implemented by third parties and by agents, and the threat model forbids depending on someone else's client behaviour. A rule only the honest client obeys is not a rule, and here the dishonest client is precisely the adversary of the scenario.

**Refuse and discard the entry without authority.** Rejected because discarding erases the signal of abuse. Whoever tried to act outside their own role is valuable information for whoever administers the project, and making the attempt disappear returns, on a smaller scale, the problem that the immutability of §10 closes. Recording without effect preserves the trail and does not contaminate the state.

**Authority by access control list on the `Project` object, instead of by role on the task.** Rejected for reintroducing permission where ADR-0011 put a key: a list the server consults is exactly the model this project refused. A participant's role is a property of the task, travels signed with it, and each peer evaluates it alone.

**Leave the policy entirely configurable per project, with no default.** Rejected because the default is what protects whoever configures nothing, which is most people. Configuration without a safe default hands the user work they do not know exists.

## Consequences

The property that EW-RFC 0015 §8 already claimed is gained: the audit artefact comes to resist a legitimate but ill-intentioned participant, and not only a stranger.

It costs one extra resolution per received entry, against the task's `participants`, which the client already holds. It is cheap.

A new state appears in the interface, the entry without effect, which clients need to know how to display, saying which role was missing. Hiding the state would lose the signal, so the display requirement is normative.

An ordering dependency is created: the role change that authorises an action needs to be published before the action that uses it. A role granted and used in the same instant would stop the other participants from seeing the grant in time.

It remains open what happens to a participant who changes role mid-task, and whether authority is evaluated at the instant of signing or at the instant of application. EW-RFC 0015 records the question.
