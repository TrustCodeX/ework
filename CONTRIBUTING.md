# Contributing

This repository is the **published** form of the specification. It is generated
from a working repository where the drafts are written, so a pull request
changing a file here would be overwritten by the next publication.

That is not a way of saying contributions are unwelcome. It is a warning about
where they land.

## What is useful right now

The specification is in Draft status, which means the useful contribution is
finding the holes rather than fixing the prose. Concretely:

- **A scenario that does not work.** The six reference scenarios in
  [the vision](docs/vision.md) are the acceptance criteria. If you can walk one
  of them through the specification and hit a step that needs magic, that is
  the most valuable thing you can report.
- **An attack the threat model misses.** [The threat model](docs/threat-model.md)
  lists 29 adversaries with their mitigations and residual risks. A thirtieth,
  or a mitigation that does not hold, is worth more than any wording fix.
- **A place where the text says something different from what it means.**
  Normative words (MUST, SHOULD, MAY) carry weight; if one of them is used
  where the sentence is not normative, that is a real defect.
- **Prior art we missed.** [The landscape](docs/prior-art.md) is the most
  fragile document here, because it makes claims about how other systems work.
  A correction with a source is welcome; a claim without one is not usable.

## How to send it

Open an issue. Include the document and the section, and say what you expected
to be true. If you are reporting a scenario that breaks, walk through it step by
step: that is what makes it actionable rather than a matter of opinion.

For anything with security impact, the [security contact](docs/threat-model.md)
of the reference host is `security@imake.codes`.

## Implementations

There is a reference implementation running on two federating domains, but it
is not published here yet: its code carries comments and function names in
Portuguese, and half-translated code is worse than unpublished code. When the
translation happens, it arrives under Apache-2.0, separately from this
specification's licence.

If you want to implement independently, that is the point of the whole exercise.
An RFC only moves from Proposed to Stable once a second independent
implementation exists.
