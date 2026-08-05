# ADR-0005: The data model as a JSCalendar profile, not a new format

**Status:** accepted (2026-08-04)

## Context
The temptation of every new protocol is to invent its own task model. The research showed the IETF already has one: JSCalendar (RFC 8984) defines `Task` in JSON with progress (including failed), percentComplete, estimatedDuration, recurrence and participants; RFC 9253 gives typed dependencies with a gap; draft-ietf-calext-ical-tasks adds states and modes designed for assignment. Twenty-five years of iCalendar prove that this vocabulary ages well.

## Decision
EWP's Task object is a **profile of JSCalendar Task**: a mandatory subset plus our own extensions under the `ework.dev` namespace (negotiation, consent and origin, urgency and escalation, binary attachments, actions, typed payloads, status policy). Dependencies use the RFC 9253 vocabulary mapped into JSON. The mapping to and from VTODO is defined for the bridges.

## Alternatives considered
- **Our own format from scratch:** rejected. High cost, no semantic gain, and it breaks the CalDAV bridge.
- **VTODO and iCalendar as the native format:** rejected. 1998 syntax, fragile escaping, poor for signing and canonicalisation; JSCalendar exists for exactly this.

## Consequences
- Interoperability with the calendar ecosystem comes almost for free (exporting VTODO is mapping, not translating between worlds).
- We stay aligned with the evolution of calext; if JMAP Tasks is revived (a 2027 milestone), the conversation is natural.
- The `ework.dev` extensions need disciplined registration and versioning (RFC 0000).
