# ADR-0002: An HTTPS substrate, with attachments as raw binary

**Status:** accepted (2026-08-04)

## Context
A new protocol has to choose where it lives: its own port with its own framing (the classic SMTP and IMAP spirit), a profile over an existing protocol (Matrix, ActivityPub, JMAP), or a new application protocol defined over the web substrate. The origin requirement adds one item: support for sending and receiving raw binary files, because that is smaller and scales better.

## Decision
1. EWP is a new application protocol defined over **HTTPS (HTTP/2 or better)**, with WebSocket or SSE for real time. It is the path Matrix and JMAP took: it crosses NAT and firewalls, works in a browser and on mobile, and hosts behind any reverse proxy.
2. **Attachments and blobs always travel as raw binary**: PUT and GET with the real media type, SHA-256 hash addressing, resumable upload, ranged download. Base64 is forbidden on the data path (tolerated only for tiny inline values inside JSON objects).
3. Control envelopes are canonical JSON (JCS); **CBOR is a negotiable alternative encoding** for those who need minimal bytes.
4. Transport evolutions (QUIC, WebTransport) do not change the model: they are new bindings of the same semantics.

## Alternatives considered
- **A port and framing of our own:** rejected. Hostile to mobile and web, blocked by corporate firewalls, it makes every client more expensive, and it buys nothing that binary HTTP/2 does not already give.
- **A profile over Matrix, ActivityPub or JMAP:** rejected. The task model would be squeezed into the host format (Matrix's room DAG is far too heavy; ActivityPub has neither E2EE nor consent; JMAP does not federate), and the project's fate would be coupled to theirs.

## Consequences
- The specification spends its energy on what matters: the data model, federation, consent and cryptography, not on byte framing.
- Base64's roughly 33% overhead is eliminated from attachments; disk-to-disk streaming; deduplication by hash for free.
- Two envelope encodings (JSON and CBOR) require a well-defined canonicalisation for signing (handled in RFC 0001).
