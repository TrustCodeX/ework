---
docname: draft-wilhelm-ework-payloads-00
title: "The e-work Protocol (EWP): Typed Payloads"
abbrev: EWP Payloads
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8259
---

This document specifies typed payloads for the e-work Protocol (EWP): the
mechanism by which a task carries structured domain content, the extension
model, and the rule that a client encountering an unknown type degrades to
something useful rather than failing.

<!-- abstract -->

# Introduction

A task about paying a bill and a task about confirming an appointment share
title, deadline, and state, and differ in everything else. Modelling that
difference in prose puts the parsing burden on the recipient. Modelling it in a
fixed set of task subtypes puts the burden on the protocol, which then needs a
new version for every domain.

Typed payloads take the third path: the task is generic, and it carries typed
content the way an email carries typed parts.

## Conventions and Definitions

<!-- rfc2119 -->

# Payload Structure

~~~
"ework:payloads": [
  { "type": "ework.dev/payload/payment@1",
    "data": { "amount": "412.87", "currency": "BRL",
              "dueDate": "2026-09-10",
              "beneficiary": { "name": "Utility Co." },
              "reference": "8829-4471" } }
]
~~~

`type` is a versioned identifier. `data` is an object whose shape the type
defines.

A task MAY carry more than one payload. Payload order is not significant.

# The Degradation Rule

**A client encountering an unknown payload type MUST still present the task.**
Title, deadline, issuer, and available actions come from the task itself and do
not depend on the payload.

This is the property that makes the extension model viable. If an unknown type
broke the task, every new type would be a breaking change for every client that
had not yet implemented it, and in a federated system with independent
implementations that is the same as saying no new types.

A client MAY show the raw payload, and SHOULD indicate that it does not
understand the type rather than pretending the task has no content.

# Naming

| Origin | Form |
|---|---|
| Core types | `ework.dev/payload/<name>@<major>` |
| Third-party types | reverse domain, `com.example.payload/<name>@<major>` |

The major version is part of the identifier. A backward-incompatible change is a
new identifier, so a client that understands `@1` is never handed `@2` content
under the same name.

Third parties MUST use a reverse-domain prefix. Registration is not required to
use a type, and the registry exists so that widely used types converge rather
than to gate what parties may exchange.

# Core Types

The following are defined by this document.

## payment@1

| Field | Required | Meaning |
|---|---|---|
| `amount` | yes | Decimal string, never a float |
| `currency` | yes | ISO 4217 code |
| `dueDate` | yes | Date |
| `beneficiary` | yes | Object with at least `name` |
| `reference` | no | Issuer's reference for reconciliation |
| `instrument` | no | Payment instrument details, regionally defined |

`amount` is a string because a float cannot represent decimal money exactly, and
a protocol that transports money as a float will eventually transport the wrong
amount.

The payload describes what is owed. **It does not carry payment credentials, and
it does not perform payment.** EWP transports the task; settlement happens in
whatever system the parties use.

## appointment@1

| Field | Required | Meaning |
|---|---|---|
| `start` | yes | Date-time |
| `duration` | no | ISO 8601 duration |
| `location` | no | Object or free text |
| `provider` | yes | Object with at least `name` |
| `preparation` | no | Free text: fasting, documents to bring |

## delivery@1

| Field | Required | Meaning |
|---|---|---|
| `window` | yes | Object with `start` and `end` |
| `items` | no | Array of descriptions |
| `tracking` | no | Carrier reference |

## document-review@1

| Field | Required | Meaning |
|---|---|---|
| `documents` | yes | Array of attachment identifiers |
| `deadline` | no | Date-time |
| `decision` | no | Array of permitted decisions |

# Payloads and Consent

The consent scope lists `payloadTypes`. The receiving host MUST refuse an
envelope carrying a payload type outside the granted scope.

This is what keeps a grant meaningful. Consent to receive bills is not consent
to receive appointment reminders, and without the check the scope would be
advisory.

# Payloads and Compartments

A payload MAY be sealed into a narrower compartment, as specified in the
compartments document. The sensitivity policy maps labels to compartments, and a
payment payload in a project labelled `money` routes to the financial
compartment automatically.

# Security Considerations

Payload content is attacker-controlled input from the recipient's perspective.
Clients MUST validate against the type's schema before rendering, and MUST NOT
execute anything a payload contains.

A payload MUST NOT carry credentials, tokens, or secrets. The protocol offers no
mechanism to protect them beyond the transport and the encryption mode, and a
type that requires them is misdesigned.

# Privacy Considerations

The payload type travels in the task, and in assisted mode the host reads it. A
type name can itself be sensitive: `ework.dev/payload/appointment@1` from a
clinic reveals something even without the content. The compartments document
specifies how to seal payloads away from parties who should not see them, and
the cryptography document states what the host observes in each mode.

# IANA Considerations

This document requests the creation of a registry, "EWP Payload Types", with
Specification Required as the registration policy, initially containing
`payment@1`, `appointment@1`, `delivery@1`, and `document-review@1` under the
`ework.dev/payload/` prefix.
