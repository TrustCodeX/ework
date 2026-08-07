# ADR-0014: The entry is the unit, with message and action in the same object

**Status:** accepted (2026-08-04). Supersedes [ADR-0013](adr-0013-comments.md).

## Context

ADR-0013 established that a comment is a signed record of the task and not a parallel chat, and that reason and act have to be atomic. The mechanism chosen was: the transition envelope carries an embedded comment.

The right inversion is this: underneath, a comment is `{ message, action }`, and an ordinary comment is simply the one where the action is null. Marking something complete is an entry with a message and an action; writing an observation is the same entry with a null action.

The difference looks cosmetic and is not. In the previous model there are two categories of thing (transitions, which may carry a comment, and comments, which have no action) and therefore two lists to reconcile in the interface, in storage and in auditing. In the chosen model there is **one** category, and auditing is literally reading the list of entries: who did what, when, and why, in the same signed record.

It also solves an open product problem: the question "why do you want to mark this complete?" stops being a feature to invent and becomes the natural consequence of the action requiring a non-empty message.

## Decision

1. **The entry (`Entry`) is the unit of the task's history:** `{ message, action, ... }`, always signed. A null action is an ordinary comment; a filled action is the transition, with the message as its justification.
2. **Atomicity by construction**, not by rule: message and action are fields of the same signed object, so there is no way to separate, reorder or reattribute them.
3. **The task envelope types collapse into `task.entry`.** Accept, decline, counter, start, acknowledge, complete, fail, cancel, confirm, dispute, defer and update become values of `action`, not distinct envelope types.
4. **An unplanned and substantial privacy gain:** the envelope type is in the clear in the routing header (EW-RFC 0001 §4). With separate types, the host learned that a task had been completed, declined or deferred. With `task.entry`, it sees only that there was activity. A metadata leak disappears through simplification.
5. **Policy may require a message** per action type (`messageRequired`), and that is what implements "why do you want to mark this complete?".
6. **An entry with an action is immutable.** It cannot be removed or edited, by anyone, including a project owner and a host administrator. Retraction now exists only for entries with a null action. Correcting means adding a new entry, never subtracting. The rule in one sentence: what you said can be withdrawn, what you did cannot.
7. **Still valid from ADR-0013:** audience by group reusing circles, attribution with the actor class, activity derived rather than transmitted.

## Alternatives considered

- **Keeping transition and comment separate** (ADR-0013): rejected. Two lists for one thing, atomicity by rule instead of by structure, and a type metadata leak.
- **Action as a field of a comment, but keeping the old envelope types:** rejected. The redundancy of the action appearing in two places would remain, with the risk of the two diverging, and without the privacy gain.
- **A generic entry with no closed vocabulary of actions** (any string): rejected. Interoperability requires a registered vocabulary; extension remains possible by namespace.

## Consequences

- **The core shrinks:** the envelope table of EW-RFC 0001 loses seven rows, and implementing a client becomes simpler.
- **Auditing becomes trivial to describe:** exporting a task's list of signed entries is the audit artefact, verifiable by third parties without trusting anyone.
- **High propagation impact:** every RFC that cited `task.accept`, `task.complete` and the like has to cite actions instead. Done in the same pass.
- **A risk to watch:** with one action per entry, an operation that changes several things at once (accepting a time slot and confirming attendance) becomes two entries. That is acceptable, and more honest than one entry with an ambiguous meaning.
- **A consequence of immutability:** a comment written in the heat of the moment alongside an action stays there forever. That is the price of auditing, and the interface has to warn before sending, not afterwards. It is also an argument for policy to require a justification only where it genuinely matters, rather than on every transition.
- Ordering, audience, anti-abuse and activity derivation remain exactly as in EW-RFC 0015, now with a single object.
