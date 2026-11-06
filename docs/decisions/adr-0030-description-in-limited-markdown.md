# ADR-0030: The description is Markdown, and the subset is the defence

**Status:** accepted | 2026-09-11 | Michel Wilhelm

## Context

EW-RFC 0002 §1 has described a task's `description` as "limited Markdown (no HTML)" since the first draft, in August 2026. The sentence is there, and nothing else: no RFC says **which** Markdown, nor what a client does with it.

A rule without a list is not implementable. The result was the predictable one: the reference client shows the description as raw text inside a `<p>`, and whoever writes learns to use no markup at all. Meanwhile task bodies grew past "pay the bill": the first autonomous issuer in the house sends an investigation with commands, output, a list of steps and a link to the dashboard, and all of it arrives as a single paragraph with the line breaks collapsed.

This is not a request about looks. A task body that does not tell a command from prose, nor a step from an aside, is a body the person reads wrong on a phone, and reading it wrong is what produces the wrong action.

What makes the decision non-trivial is that **the description is the only large field a stranger writes and the person's client draws**. It arrives from any issuer holding consent, and consent exists to exchange tasks, not to trust. Rendering third-party Markdown, the way almost every product does, opens three doors at once: inline HTML becomes script execution; a remote image becomes a read receipt with network address and time; and a link whose text is chosen by the writer becomes the oldest scam there is.

The third door is the interesting one, because the protocol already closed it once: EW-RFC 0007 §3.3 requires showing the **verified domain** rather than the `name` on a counter offer, for exactly this reason.

## Decision

**The description is limited Markdown, the subset becomes normative in EW-RFC 0002 §1.1, and what was left out was left out for safety, not for laziness.**

1. **The subset is closed and listed**: headings of one to three `#`, emphasis, inline and fenced code, list and numbered list, quote, rule and link. It is what fits a body read on a phone.
2. **HTML is not interpreted**, anywhere. It shows as the text it is.
3. **Images do not load.** `![alt](https://example.org/photo.png)` becomes the alt text with the address beside it. No network request leaves on opening a task.
4. **Links only in `http`, `https` and `mailto`**, and the client MUST show the host when the link text does not contain it.
5. **`descriptionContentType`** (RFC 8984 §4.2.2, from the JSCalendar the model already inherits) declares the type. **Absent means `text/markdown`**, against the JSCalendar default, and an unknown value means `text/plain`.
6. **The defence is the construction, not a filter**: the renderer produces interface elements, never an HTML string. There is no path to a tag, so there is no sanitiser to keep up to date.

## Alternatives considered

- **Stay on plain text.** Rejected: it contradicts what RFC 0002 has said for a year, and pushes structure into the paragraph, where it becomes a convention for the writer and guesswork for the reader.
- **Full Markdown, with an off-the-shelf library.** Rejected for two reasons, the second heavier than the first. It brings tens of kilobytes into a bundle that ships inside the host binary; and it produces HTML, which forces the client into sanitising at every version, forever, against whoever writes the attack. A subset that never becomes HTML carries no such debt.
- **Accept `text/html` as JSCalendar suggests.** Rejected: the RFC 8984 SHOULD is about calendars between peers who already know each other. Here the author of the text is a stranger, and a stranger's HTML is precisely what you do not draw.
- **Keep the JSCalendar default (`text/plain`) and require the declaration.** Rejected because it would rewrite the meaning of every description already written under the RFC 0002 §1 rule, and because it would turn into opt-in what the protocol always promised.
- **Allow images from the issuer's own origin.** Rejected: "same origin" is still a server learning the time of reading, and the client has no way to tell the image that illustrates from the one that reports.

## Consequences

- Issuers can structure the body, and the cost is learning that tables and images do not get in.
- The reference client gains a small parser of its own, which is code to maintain. That is the price of not depending on someone else's sanitiser, and it comes with tests for what the renderer **refuses** to do.
- Descriptions written before this decision stay readable: text without markup crosses the parser as a paragraph.
- A client that does not implement §1.1 shows the text with the markup visible. It degrades ugly and loses nothing, which is the degradation rule of EW-RFC 0008.
- Tables remain an open question in EW-RFC 0002. They are useful in an operations body and expensive on a phone screen, and that trade has not been made yet.
