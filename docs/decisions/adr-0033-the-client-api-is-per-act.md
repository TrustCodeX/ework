# ADR-0033: The client API is per act, and what was sent has a state in the stream

**Status:** accepted | 2026-11-06 | Michel Wilhelm

## Context

EW-RFC 0004 §2 was born on 2026-08-04 with JMAP's method list (RFC 8620): `get/set/changes` per type, plus `Envelope/send|pending`, `Group/commit|fetch` and `Blob/ref`. The reference host started the same day and took another path, because the protocol did:

- ADR-0014 made the entry the unit of change of a shared task, with authorship, authority (ADR-0021) and justification. Changing a task stopped being editing properties.
- EW-RFC 0007 made every consent transition a signed act with its own rule: a grant, a revision with a ceiling, a polite or a silent revocation, a block that does not announce itself.
- Each act got its own method at the host. Counted on 2026-11-06 in the dispatcher of `api.rs`: 61 methods served, of which EW-RFC 0004 named 8 and the whole suite 9 (the ninth, `Entry/add`, in EW-RFC 0015). Of the 13 that §2 listed, none existed at the host.

Three consequences showed up at once:

1. The English draft had traded the list for a table of families, and the Portuguese RFC fell behind its own translation.
2. The mobile app, which ADR-0031 publishes once to connect to any host, depended on methods no RFC defines. It tolerated `Offer/sent` being absent because it took it for a host extension. RNF-14, a minimal client from reading RFCs 0001 to 0004, did not hold: whoever read the RFC could not write the app.
3. The record of what an account sent had no state, and the app's bell only learned of acceptance and completion on a full synchronisation. Reading the host turned up three more defects. Between different hosts, the returned status stores the entry and does not update the record. With both sides on the same host, the sender's event goes out before the write, and the `negotiation` copied from the task carries the recipient's root `did:key` into the sender's record, against ADR-0022.

The audit of 2026-10-21 (`docs/by-gemini/01-furos-e-inconsistencias-da-spec.md`, item 8) had already pointed out the methods missing from EW-RFC 0004.

## Decision

1. **One method per act.** Every change the protocol defines as an act has its own method, with the arguments and the rules of the document that defines the act. There is no generic `set`. Reading is `X/get` or `X/list`.
2. **A state per type, and `/changes` only where the collection grows without bound.** `Task` has `Task/changes`. The short collections (`Consent`, `Quarantine`, `Offer`) are reread whole when the state changes.
3. **A closed catalogue in EW-RFC 0004 §2.1, with three classes.**
   - **Protocol**: name and semantics fixed by the specification.
   - **With a capability**: exists only when the host advertises the capability (EW-RFC 0001 §8).
   - **Reference host extension** (§2.3): no client depends on it for what the specification requires.
4. **The criterion for each class.**
   - A method is protocol when a client needs it to exercise behaviour some RFC already makes normative.
   - A method is an extension when no RFC requires it, or when an RFC prevents it from counting as protocol. `History/verify` is a case of the second kind, because EW-RFC 0015 §8 forbids verification from depending on the host that stored the history.
   - In the count of 2026-11-06: 46 protocol, 6 with a capability, 9 extensions.
5. **`Offer/send` and `Offer/sent` are protocol.** They are the operations the REST gateway of EW-RFC 0010 §3 already defines for organisations, through the session of any account.
6. **The record of what was sent has its own state, `Offer`, which the event stream carries** (EW-RFC 0004 §6.1.1).
   - The host changes it on every write to the record, and stores the write before the event goes out.
   - The sender's host updates the record from the signed entries that come back, applying the action to the last known state.
7. **The end-to-end path stays in the catalogue**, marked as not yet implemented: `Envelope/send|pending`, `Group/commit|fetch` and `Task/query` in assisted mode.
8. **The listed methods that never existed leave**: `Task/set`, `Project/get|set|changes`, `Consent/get|set|changes` and `Blob/ref`.

## Alternatives considered

- **Keep JMAP's `get/set/changes` and change the host.** Rejected: a generic `set` hides the act inside a property diff, and the host would have to reconstruct the intent.
  - `Consent/set` with `state: "revoked"` does not say which of the two exits of EW-RFC 0007 §6.1 the person meant: one informs the issuer, and the other must not.
  - `Task/set` over a shared task would go around the entry, where authorship, authority and justification live.
  - Rewriting the host only to translate each `set` back into an act would arrive at the same place, with one more layer of ambiguity.
- **Document only the methods the mobile app uses.** Rejected: §2 would keep describing an API the host does not serve, and RNF-14 would keep failing for whoever writes another client.
- **Treat everything the host serves as protocol.** Rejected: it would turn into an obligation of every host three things that cannot be one:
  - the operator's administration, which ADR-0026 took out of the protocol;
  - the calendar feed, which EW-RFC 0010 is still discussing;
  - a verification verdict that EW-RFC 0015 §8 says does not count.
- **`Offer/send|sent` as a host extension.** Rejected: they are the same operations as the issuer gateway's (`POST /v1/offers`, `GET /v1/offers/{id}`), and what comes back through them is already governed by EW-RFC 0007 §6. Without them, whoever sends a task through any client is left without seeing what they sent. And the protocol's stream carries no state for a type the protocol does not define.
- **The state of what was sent only as an open question.** Rejected: the failure had already been measured in the app, and the two host defects can only be described against a written rule.
- **`/changes` for every type.** Rejected for now: the short collections fit in one read, and the reference host never kept per-type history beyond `Task`. For `Entry`, where the read grows with the number of tasks, the question was recorded (EW-RFC 0004, open question 14).
- **A namespace in extension method names.** Deferred: it would settle the distinction on the wire, but forces renaming methods clients already use. It became open question 20 of EW-RFC 0004.

## Consequences

- EW-RFC 0004 §2 becomes a catalogue by family, with the six everyday methods described in full, and `spec/drafts/sync.md` follows. §3 gains the rule for short collections and concurrent writes through the entry. §6.1.1 brings the `Offer` state.
- No client loses anything from the methods that leave: no host served them.
- The reference host now owes the text five corrections, in code separate from this ADR:
  - emit the `Offer` state, in the session and in the stream;
  - update the record when status comes back from another host;
  - store the record before the event when both sides live on the same host;
  - name the recipient, in the record's `negotiation`, by the relationship's contact key rather than the root;
  - read the `timeZone` of `Task/create`, which it now ignores in favour of `America/Sao_Paulo`.
- The Python test that checks the set of states in the stream gains `Offer` in the same layer.
- The mobile app does not change to receive `Offer`: it already maps the type to `Offer/sent` and rereads whatever the stream says changed. The web client reloads on any change.
- These stay as open questions of EW-RFC 0004:
  - `Entry/changes`;
  - `Project` and `Refused` in the stream;
  - an error code for an unknown method;
  - which families the conformance profile requires;
  - the record of what was sent, in the export;
  - a namespace for extensions;
  - three differences of shape not yet decided: the session document, the response of `Task/changes` and `oldState`/`newState`.
