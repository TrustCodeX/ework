# ADR-0007: Bridges to the legacy world as first-class citizens

**Status:** accepted (2026-08-04)

## Context
A new federated protocol faces the fax problem: the first user has nobody to talk to. The research showed two facts: (a) there is a real installed base of interoperable tasks (CalDAV and VTODO: Nextcloud, Tasks.org, Thunderbird) with degraded but working interoperability; (b) issuers only integrate what fits into their ERP with minimal effort.

## Decision
The bridges are part of the specification (RFC 0010), not accessories:
1. **A CalDAV gateway:** boxes exposed as VTODO collections (reading and writing the subset the real world supports). With E2EE, the gateway runs on the client side (a local daemon) or requires the collection to be in assisted mode: the spec forbids a gateway that silently breaks E2EE.
2. **An email fallback:** the offer becomes a readable email plus an `.ics` attachment (VTODO) plus an acceptance link; replies map back. It adopts structured email (the sml working group) once that matures.
3. **An issuer gateway:** a minimal REST API (create an offer, receive status) for plugging in ERPs without speaking native federation.
4. **Complete export and import** in a documented format, which is also the mechanism for migrating between hosts.

## Alternatives considered
- **Purity (no bridges):** rejected. That is the Solid path: beautiful architecture, zero adoption.
- **Being merely an extension of CalDAV:** rejected. CalDAV accommodates neither federation, nor consent, nor E2EE (ADR-0002).

## Consequences
- VTODO's lowest common denominator (status and recurrence break between clients) limits what the bridge can promise; the spec documents the losses rather than pretending equivalence.
- The issuer gateway becomes the entry product for companies: it has to be ridiculously simple (one HTTP call with an API key).
