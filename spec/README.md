# The specification

The normative specification, as Internet-Drafts. `EW-RFC NNNN` is the
number used inside the project; the file name is the draft's short name.

Every document has the same seven sections: Summary, Motivation,
Specification, Security considerations, Privacy considerations, Open
questions, References. An empty section means the document is not ready,
not that the section does not apply.

[The process, the status ladder and the open questions](0000-index-and-process.md)
are in RFC 0000. [How these become IETF submissions](PUBLISHING.md) is in
PUBLISHING.md.

| RFC | Document | Subject |
|---|---|---|
| 0001 | [core.md](core.md) | Core: conventions, URIs, discovery, envelopes |
| 0002 | [data-model.md](data-model.md) | Task data model |
| 0003 | [identity.md](identity.md) | Identity, addressing and rotation |
| 0004 | [sync.md](sync.md) | Client-server synchronisation |
| 0005 | [federation.md](federation.md) | Server-to-server federation |
| 0006 | [crypto.md](crypto.md) | Cryptography and encryption modes |
| 0007 | [consent.md](consent.md) | Consent before delivery |
| 0008 | [payloads.md](payloads.md) | Typed payloads |
| 0009 | [urgency.md](urgency.md) | Urgency and escalation |
| 0010 | [bridges.md](bridges.md) | Bridges to existing systems |
| 0011 | [anti-abuse.md](anti-abuse.md) | Anti-abuse |
| 0012 | [human-discovery.md](human-discovery.md) | Discovery by human identifiers |
| 0013 | [compartments.md](compartments.md) | Compartments and sensitive data |
| 0014 | [actors.md](actors.md) | Autonomous actors and execution confirmation |
| 0015 | [history.md](history.md) | Entries and verifiable history |
| 0016 | [compliance.md](compliance.md) | Data protection compliance |

## Reading order

If you are implementing, read `core`, `data-model`, `identity` and `sync`
in that order: they are enough for a client that reads a box, accepts an
offer and completes a task. `federation` and `consent` come next, and they
are where this protocol differs from everything else.

If you are here to evaluate the idea rather than build it, read
[consent](consent.md) alone. It is the one document that carries the
argument.
