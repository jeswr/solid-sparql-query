# Access-Controlled SPARQL Query over a Solid Pod

A specification (Solid Community Group track, destined for <https://solidproject.org/TR/>) defining
an HTTP endpoint that answers **read-only SPARQL queries over a Solid Pod's RDF content, returning
only the triples the authenticated agent is authorized to Read**.

It is a new *read interface* layered on Solid's existing **WAC/ACP** authorization + **Solid-OIDC /
DPoP** authentication — **not** a new authorization model and **not** a new wire protocol. It is
designed to conform to the endpoint implemented by
[`jeswr/solid-server-rs`](https://github.com/jeswr/solid-server-rs); its server-side contract is the
`query_as` / `decide` seam tracked in **`jeswr/sparq#992`**.

## Status — EDITOR'S DRAFT AVAILABLE

The **ReSpec Editor's Draft is written**: **[`index.html`](index.html)** (a single-file W3C Solid
Community Group CG-DRAFT — open it in a browser, or serve the repo root, to render it). The full
design record is in **[`design/DESIGN.md`](design/DESIGN.md)**; every design call is now
**RESOLVED**, including the two formerly-open ones (decided by Claude Fable, 2026-07-01 —
rationale in DESIGN.md §9):

1. JSON-LD inner named graphs — **flatten in v1**; RDF-star reification is a named non-normative
   roadmap fidelity mode; raw inner-graph-name preservation is a normative MUST NOT.
2. **Union-default-graph mode is in v1 as an explicit per-query opt-in** (reserved IRI
   `solid-sparql:union-default-graph`; advertised via the minted `sd:Feature`
   `solid-sparql:UnionDefaultGraphOptIn`; never a standing default).

Next steps: mirror the server-side contract onto `jeswr/sparq#992`
([`design/sparq-992-mirror.md`](design/sparq-992-mirror.md)) and contribute the draft to the
Solid Community Group.

## Contents

- `index.html` — the **Editor's Draft** (ReSpec, CG-DRAFT). **Read this first.**
- `design/DESIGN.md` — the design brief, the resolved decisions, and the recorded rationale.
- `design/sparq-992-mirror.md` — the server-side contract statement for `jeswr/sparq#992` /
  `jeswr/solid-server-rs`.
- `test-suite/query-semantics/` — a **machine-readable conformance test suite** for the
  query-semantics conformance class (empty default graph, union-default-graph opt-in, the
  non-disclosure invariants). Portable across implementations; a reference runner over
  `jeswr/sparq`'s `PodStore` passes every case. See its `README.md`.

- `spec.statements.ttl` — the **machine-readable normative-statement companion** (Turtle
  sidecar): every normative statement as an anchored `spec:Requirement` with a verbatim
  quote, RFC 2119 level, conformance-class binding, E/A-int/A-exist/P testability tag and
  honest test-gap accounting. **Complementary — `index.html` remains the sole normative
  text.** The companion pins the spec source commit it was extracted from
  (`sc:specVersion`); a change to the spec's normative text and a re-extracted companion
  must land in the same commit. Format/shapes/validator:
  [`jeswr/spec-companion`](https://github.com/jeswr/spec-companion); validate from a
  sibling checkout:

  ```sh
  node ../spec-companion/tools/validate.mjs spec.statements.ttl --spec-html index.html
  ```

## Provenance

Design authored with **Claude Opus 4.8**; the two remaining design calls were made, and the
Editor's Draft written, by **Claude Fable** (2026-07-01). See DESIGN.md §9–§10.

## Licence

Spec text: W3C Community Contributor Licence / CC-BY-4.0 (to be confirmed on CG contribution).
