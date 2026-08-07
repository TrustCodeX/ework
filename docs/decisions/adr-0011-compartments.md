# ADR-0011: Visibility by cryptographic compartment, not by permission on the server

**Status:** accepted (2026-08-04)

## Context

A real project has different kinds of task, with amounts, documents and personal data mixed together. The fitted furniture scenario makes it obvious: the fitter needs to know the delivery date and does not need to know how much the customer paid; supplier Beta must not see the price charged by supplier Gama; the customer wants to see everything. The origin requirement, as it was put: the owner has to be able to mark what is sensitive data and decide who sees what, both in the project and in the task, and not everyone needs to see every task.

The obvious implementation would be a permission list: every task and every field with a list of who may see it, enforced by the server.

That does not work here, and it is important to understand why before designing anything. In EWP the server is blind (ADR-0004): it routes ciphertext and reads nothing, so it cannot enforce a rule over content it cannot see. And if all the project's members share the same cryptographic group, all of them can open all the packets; hiding a field in the interface would be theatre, undone by any alternative client, by a ten-line script or by the build of the app with the export button. A privacy control that depends on the goodwill of somebody else's client is not a control, it is a notice.

## Decision

1. **Visibility is a key, not a permission.** Whoever can see a datum is exactly whoever has the key to decrypt it. There is no visibility rule enforced by the server, not even for convenience.
2. **A project has N compartments, each with its own MLS group.** The general compartment has every member; narrower compartments exist by subject (finance) or by relationship (the owner plus one specific supplier).
3. **Two granularities, distinct mechanisms:**
   - **A whole task in a compartment:** whoever is outside never receives the packet, and for that person the task does not exist.
   - **A sealed section inside a shared task:** everyone sees the essentials (title, deadline, state) and only the narrow compartment opens the sensitive fields.
4. **Marking data as sensitive is labelling, not choosing a key.** The author labels the field (`money`, `health`, `personal-data`); the project's policy maps the label to a compartment. The ordinary user never thinks about cryptography.
5. **The defaults protect on their own.** Payment payloads are born sealed; the author has to act in order to open, never in order to close.
6. **What the key does not protect is a hint, not a control**, and the interface MUST say so in those words. No "private" label that does not correspond to a compartment.

## Alternatives considered

- **An ACL enforced by the server:** rejected. Incompatible with a blind server, and even in assisted mode it would mean entrusting the secret to the host for precisely the most sensitive data in the system.
- **Hiding only in the interface, with everyone in the same group:** rejected. It is the cheapest alternative and the most dishonest: it creates a feeling of privacy with no privacy at all.
- **One project per circle of trust** (the carpenter creates one project with the customer, another with each supplier): rejected as the main mechanism. It is what people do today for lack of anything better, and it destroys what gives the project its value: dependencies between stages, a single schedule and a view of the whole.
- **Per-field encryption with a key per recipient** (PGP-style, per attribute): rejected. It does not scale with members joining and leaving, and it badly reimplements what MLS already solves with epochs.

## Consequences

- **A real implementation cost:** the client now manages several MLS groups per project, and has to decide, at the moment of writing, which compartment each piece goes into.
- **Nothing is retroactive.** Reclassifying a field protects the future; whoever has read it, has read it. The interface MUST say so before the person believes they have "fixed" a leak.
- **The existence of a sealed section is visible** to whoever is on the task. Whoever needs to hide even its existence uses a whole-task compartment, and the cost is losing coordination on that item.
- **Dependency between compartments** needs a pattern of its own, the public milestone as a proxy, otherwise the project graph becomes a leak channel by inference.
- **The decisive gain:** the promise "the fitter does not see the customer's payment" becomes verifiable rather than promised, and it holds even if the fitter uses a modified client.
