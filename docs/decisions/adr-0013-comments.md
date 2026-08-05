# ADR-0013: A comment is a signed record of the task, not a parallel chat

**Status:** superseded (2026-08-04) by [ADR-0014](adr-0014-entry-as-the-unit.md), which keeps the principle below in full and inverts the mechanism: instead of the transition carrying a comment, the entry carries a message and an action. The text is preserved because the analysis that rejects the parallel chat still holds.

## Context

There was no way for the counterparties to talk about the task. Michel asked for comments synchronised between the parties, and gave the example that defines the design: "Bot says: I have just done the fix on the system, therefore Status: closed".

That example does not describe a message followed by an action. It describes **one single thing**: the change of state and its reason. If they were two separate events, the reason could be lost, reordered, attributed to another author, or the transition could happen with no justification at all. It is the difference between a chat next to the task and the history of the task.

The lazy alternative would be embedding a chat: a timeline of messages, with actions appearing as events. It works for conversation and fails at what matters: you cannot audit afterwards who changed what and why, the comment does not inherit the task's privacy, and in a federated system with end-to-end encryption the ordering becomes a problem with no owner.

## Decision

1. **A comment is an object of the task**, encrypted in the same group as the circle it belongs to, synchronised between the counterparties like everything else.
2. **A transition may carry a comment in the same envelope**, and then the two are atomic: the envelope is signed as a whole, so the reason does not become separated from the act, neither later nor through reordering.
3. **Two natures in the same timeline:** a note (text somebody wrote) and activity (what happened). Activity is **derived locally from the signed history of operations**, never transmitted as somebody's assertion. Nobody can invent a "so-and-so changed the deadline" that never happened.
4. **A comment's audience is a group**, reusing circles (EW-RFC 0013): a private note is a comment in the group where only you are; a conversation with the issuer goes in the relationship group; a project discussion goes in the circle. One mechanism, not three.
5. **Authorship carries the actor class** (EW-RFC 0014): "Bot says" is not a text convention, it is a verifiable attribution.
6. **Retraction is a request, not a guarantee**, and the interface says so. Whoever already received it, received it.

## Alternatives considered

- **A separate chat attached to the task:** rejected. It loses the atomicity between reason and act, duplicates the privacy and ordering problem, and reintroduces exactly the WhatsApp group the protocol exists to eliminate.
- **A comment as a single text field on the task** (an overwritten annotation): rejected. It has no authorship, no history and is useless for more than one party.
- **Activity transmitted as a message** (the client sends "so-and-so changed the deadline"): rejected. It would be a forgeable and redundant assertion, since the signed operations already contain that information.
- **Reactions and emoji:** out of scope. Noise with no record value.

## Consequences

- The task's history becomes **proof**: every change has a signed author, an actor class and, where one exists, the justification attached to it.
- The bot case becomes natural: the agent completes and explains, and whoever reads it knows it was an agent.
- Ordering under end-to-end encryption needs explicit causal references, because the server is blind and does not order for us.
- A comment becomes one more issuer abuse vector (marketing inside the task's thread), and for that reason it is covered by the rules of EW-RFC 0011.
- A private note requires a group with a single member, which is the degenerate case of circles and has to be cheap to create.
