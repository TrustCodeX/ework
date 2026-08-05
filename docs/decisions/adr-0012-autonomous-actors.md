# ADR-0012: Non-human actors as first-class citizens, with human confirmation anchored in a key

**Status:** accepted (2026-08-04, decided with Michel)

## Context

The specification had been assuming, without saying so, that behind a client there is a person. Michel pointed out that this is false and will become more false: a client can be a machine, an agent, an autonomous system or an AI, and the interesting part is that **there is not always a human in the loop, but some tasks require one**.

The problem, stated precisely: how does a task require a human to confirm, in an open protocol where anyone writes their own client and the server is blind? A `requiresHuman: true` flag that the other side's client is supposed to respect is exactly the kind of control ADR-0011 already rejected: a promise dependent on the goodwill of somebody else's software, undone by a script.

There is also the inverse side, and it is the one with practical value: when there is no human, the protocol has to be machine-executable without ambiguity, with readable acceptance criteria, a structured result and idempotency.

## Decision

1. **Actor class is a property of a key, not of an account.** Each device key declares its class (`human`, `assisted`, `autonomous`, `system`), signed by the root. An identity mixes them freely: the person's phone is `human`, the agent running on their server is `autonomous`.
2. **Human confirmation is a signature from a key of class `human`.** It is the same move as the circles: it turns policy into a key. An agent that holds no human key **cannot produce** the confirmation, no matter which client it uses. It stops being a rule you ask for and becomes a property of the system.
3. **Two distinct moments, because they solve different things:** approval **before** (a human in the loop authorises execution) and confirmation **after** (somebody attests that it was in fact done).
4. **Confirmation can come from a third party.** Whoever executes is not always whoever confirms: the carrier says it delivered, the customer confirms they received. That requires a new state, `awaiting-confirmation`, between executing and completing.
5. **An agent operates through a scoped delegated credential**, with its own key, limits (amount, rate, validity, circles) and instant revocation. An agent never receives the root key nor a human key.
6. **The stricter requirement prevails.** If the issuer requires a human and the recipient does not, a human is required. If the recipient requires one and the issuer does not, it is required just the same. Nobody loosens anybody else's policy.
7. **A machine-to-machine profile** with readable acceptance criteria, a structured and signed result, and idempotency by deduplication key.

## Alternatives considered

- **A confirmation flag respected by the client:** rejected, for the same reason as in ADR-0011. With no key behind it, it is a notice and not a control.
- **A separate identity for the agent** (the agent is another person): rejected as the default. It breaks the notion of acting on someone's behalf, forces redoing consents, and dissolves the chain of responsibility. It remains allowed when the agent really is an independent service.
- **Proof of human presence by CAPTCHA or biometrics in the protocol:** rejected. CAPTCHAs are hostile and fallible, biometrics belong to the device and must not become protocol data. The key class already delivers the essential part.
- **Hardware attestation** (a human key in an enclave, proving physical interaction): recorded as a possible extension for high-risk cases, outside the core because of cost and because it excludes ordinary devices.

## Consequences

- **A real gain:** "this requires human approval" becomes verifiable, and it holds even against a modified client. A bank can issue a payment task that no agent approves on its own.
- **An honest limit, which has to be written down:** the key proves **who** signed, never that the person paid attention. A human who approves everything automatically is still a human in the loop. There is no cryptography against inattention, only interface design.
- **Machine to machine becomes viable** without turning the protocol into a message queue: tasks remain tasks, with deadlines, dependencies and status.
- **New complexity for the client:** managing key classes, delegated credentials and the `awaiting-confirmation` state.
- **Composition with circles** (EW-RFC 0013): an agent joins only the circle it needs, so delegating does not mean handing over everything. A payments agent sees the finance circle and does not see the rest.
- **Auditable responsibility:** every action carries the signature of whoever performed it, with the actor class, so "it was the agent" and "it was me" stop being indistinguishable after the fact.
