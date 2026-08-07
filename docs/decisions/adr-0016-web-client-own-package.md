# ADR-0016: The web client as its own package, with the bundle embedded in the host binary

**Status:** accepted (2026-08-05)

## Context

The host has served a web interface since the MVP, and it lives in `impl/ework-host/src/web.rs`: 1,305 lines of HTML, CSS and JavaScript inside a Rust constant. That was the right decision while the goal was to prove the protocol works from a browser, with keys being born there and the host never seeing a private key. A constant needs no build, needs no Node installed, and travels inside the binary without ceremony.

The design handoff in `design/` changes the scale of the problem. There are nine screens, overlays, a seven-step onboarding with three paths, a complete dictionary in two languages including long copy, three responsive breakpoints, and a design system with shape rules that have to be applied consistently. It is ten to twenty times the current size.

A Rust string has no syntax highlighting, no lint, no formatter, no reusable component and no test. Each of those absences is tolerable on its own; together, in a file of that order, they produce code nobody reviews because nobody can read it.

The real tension is with RNF-13, which asks for a host a hobbyist can operate: **one binary, one domain**. A modern web client normally becomes a second artefact to host, with its own origin, its own deploy and its own CORS. That would break the promise that makes someone agree to run this at home.

## Decision

1. **The web client becomes `impl/ework-web/`**, its own package in the repository, alongside `ework-core`, `ework-host`, `ework-cli` and `ework-py`.
2. **React with Vite.** The choice is about ecosystem: routing, i18n and drag and drop (which the board and the calendar ask for) have mature libraries, and it is the stack with the most people able to contribute.
3. **The compiled bundle is embedded in the host binary at build time**, and the host serves the files from inside itself. It remains one binary and one domain, and there is no second origin and no CORS.
4. **The web package is a separate build target.** `cargo build` alone does not require Node: whoever only touches the host does not need the frontend toolchain. The binary embeds whatever bundle has been built, and its absence degrades into a page that says what to do, never into a compile error. This requires the `ework-web/dist/` **directory** to exist even when empty: `rust-embed` refuses to compile pointing at a missing folder, with the message "folder does not exist". A versioned `.gitkeep` solves it, and CI proved the promise did not hold without one.
5. **The current client is not deleted now.** It keeps answering at `/` until the new one reaches parity, and the new one grows at `/app/`. The switch is the last step, not the first.

## Alternatives considered

- **Staying inside `web.rs`, growing the constant:** rejected. It is the lowest-effort alternative today and the costliest one in three months. With no components, the nine screens repeat structure; with no lint and no tests, regressions only show up in the browser; and the file passes ten thousand lines, which is territory nobody edits with confidence.
- **Svelte instead of React:** it was the initial recommendation, for a smaller bundle (which matters, because it goes inside the binary) and less ceremony for a one-person project. Rejected on the criterion of ecosystem and of who can contribute, which weighs more in a project that wants third parties implementing it. The size difference is measurable but not decisive at a bundle of this order.
- **Vanilla with Web Components, no build:** rejected. It preserves today's simplicity and transfers to the project the cost of writing routing, i18n and drag and drop by hand. Those are three solved problems not worth solving again.
- **The web client as a separate application, hosted apart:** rejected. It is the normal design in the market and it breaks RNF-13. It would require two deploys, CORS configuration and an origin decision for the keys in the browser, which is exactly where complexity is unwelcome.
- **Serving the bundle from disk, next to the binary:** rejected. Less tightly bound than embedding, and it reintroduces "the binary plus a folder that has to be beside it", which is halfway to the second artefact.

## Consequences

- **We gain** the ability to build the product in the handoff. Without this, the design stays a prototype.
- **We gain** a place where third parties who want to write another client can look for a reference on how to use the API.
- **We pay** one more toolchain in the repository, with what that brings in dependency updates and supply surface. Mitigation: the target is separate, and whoever touches the host is not required to install it.
- **We pay** one more step in the image build: the Dockerfile gains a Node stage before the Rust stage. It still produces **one** image.
- **The transition has a period with two live clients**, the old one at `/` and the new one at `/app/`. It is uncomfortable and it is the price of not breaking what works. The end of that period is an explicit issue, not a "someday".
- The binary grows with the bundle. Worth measuring and recording: if it passes a few megabytes, compressing the embedded assets comes up for discussion.
