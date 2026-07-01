# Access-Controlled SPARQL Query over a Solid Pod

A specification (Solid Community Group track, destined for <https://solidproject.org/TR/>) defining
an HTTP endpoint that answers **read-only SPARQL queries over a Solid Pod's RDF content, returning
only the triples the authenticated agent is authorized to Read**.

It is a new *read interface* layered on Solid's existing **WAC/ACP** authorization + **Solid-OIDC /
DPoP** authentication — **not** a new authorization model and **not** a new wire protocol. It is
designed to conform to the endpoint implemented by
[`jeswr/solid-server-rs`](https://github.com/jeswr/solid-server-rs); its server-side contract is the
`query_as` / `decide` seam tracked in **`jeswr/sparq#992`**.

## Status — DESIGN PHASE

The full design is in **[`design/DESIGN.md`](design/DESIGN.md)**. The two load-bearing protocol +
semantics decisions are made; **two design calls remain open** (see the design doc):

1. JSON-LD inner named graphs — **reification in v1, or flatten-now + reification-on-roadmap?**
2. An optional **union-default-graph** query mode (opt-in) — include or not?

Next step after those are decided: the **ReSpec/Bikeshed Editor's Draft** (`index.html` / `spec.bs`).

## Contents

- `design/DESIGN.md` — the design brief, the resolved decisions, and the open calls. **Read this first.**
- (later) `index.html` / `spec.bs` — the Editor's Draft.

## Provenance

Design authored with **Claude Opus 4.8**; intended for completion + the final design calls by a
subsequent model (**Claude Fable**). See the "Notes for the next editor" section of `design/DESIGN.md`.

## Licence

Spec text: W3C Community Contributor Licence / CC-BY-4.0 (to be confirmed on CG contribution).
