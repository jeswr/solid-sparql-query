<!-- AUTHORED-BY Claude Opus 4.8 -->
# Design — Access-Controlled SPARQL Query over a Solid Pod

> Design brief + resolved decisions for a **W3C Solid Community Group specification**, destined for
> <https://solidproject.org/TR/>, alongside the Solid Protocol and WAC. Conforms to
> [`jeswr/solid-server-rs`](https://github.com/jeswr/solid-server-rs). Authored with Claude Opus 4.8;
> the two remaining design calls were **resolved by Claude Fable** (§9, 2026-07-01) and the
> **Editor's Draft ([`../index.html`](../index.html)) is written to them**. This document is now the
> design record behind that draft; the draft is normative where they differ.

## 1. Scope & goal

Define an HTTP endpoint that answers **read-only SPARQL queries over a Pod's RDF content**, returning
**only the triples the requesting agent is authorized to Read**. It is a new *read interface* over
the Solid server's existing authentication layer and WAC/ACP authorization substrate — not a new
authentication mechanism, not a new authorization model, and not a new wire protocol. Requests
without an authenticated agent are evaluated only as the public agent and fail closed wherever
that agent lacks Read authorization. It fills a real gap: Solid has no ratified server-side query
surface today (querying is client-side, via Comunica link-traversal). Target: a Solid CG
**Editor's-Draft-track** document.

## 2. Decision 1 — SPARQL **1.1** Protocol (require 1.1; 1.2 optional superset)

**RESOLVED.** Normatively require the **SPARQL 1.1 Protocol** query operation as the floor; treat
**SPARQL 1.2 as an optional, non-breaking superset** — do **not** require it.

- SPARQL 1.1 Protocol is a stable **2013 W3C Recommendation**, universally supported (Jena, RDFLib/
  SPARQLWrapper, Comunica), with `application/sparql-results+json`. **SPARQL 1.2 Protocol is only a
  Working Draft** (Apr 2026; not even CR — the CR milestone belongs to *RDF 1.2 Concepts*), so
  requiring it would peg the spec to a moving pre-CR target.
- The 1.2 protocol layer is a **backward-compatible superset**: same GET / POST-form / POST-direct
  bindings, same core media types; it adds only an optional `version` media-type parameter and
  RDF-1.2-aware payloads. Forward-compat costs nothing.
- **Access control is layered on the standard protocol** — the ordinary HTTP request is
  authenticated through the Solid server's existing authentication layer, and a SPARQL 1.1
  Service Description provides discovery. **No new protocol is minted**, so existing SPARQL
  clients can use whichever authentication mechanism that Solid server supports.
- **Conformance keywords:** MUST implement the 1.1 query operation (all three bindings) +
  content-negotiate the result formats of §7; SHOULD serve a Service Description on a
  no-query-string GET (§8); MAY accept the 1.2 `version` parameter + emit RDF-1.2 triple terms.
- **Rust conformance:** `solid-server-rs`'s internal `HttpSparqClient` already speaks this exact 1.1
  wire form (`application/sparql-query` → `application/sparql-results+json` / `application/n-triples`);
  the *client-facing* endpoint is the not-yet-built piece (§8/`sparq#992`).

## 3. Decision 2 — Pod content → SPARQL dataset (incl. JSON-LD named graphs)

A SPARQL query executes against an **RDF Dataset** = one default graph + zero-or-more IRI-named
graphs (SPARQL 1.1 Query §13; RDF 1.1 Concepts §4). The default-graph contents are endpoint-defined
(§13.2) — that is the seam we design.

### 3(a) Resource → graph — **graph name = resource IRI** (RESOLVED)
Each Solid LDP RDF resource at IRI *X* contributes the named graph `<X>`. Standard quad-store/LDP
mapping; matches what `solid-server-rs` already models (`GRAPH <resource-iri> { … }`).

### 3(b) Default graph — **empty**, with cross-graph query via `GRAPH ?g` (RESOLVED)
The default graph is **empty** — a bare `{ ?s ?p ?o }` returns nothing. Readable resources are
exposed as **named graphs**, so all cross-graph querying is explicit and access-safe by
construction. See §4 for the join semantics (there is no `FROM *`/`FROM ?g` in SPARQL — the idiom
is `GRAPH ?g`). An **opt-in union-default-graph mode** is included in v1 (RESOLVED §9.2 —
explicit per-query opt-in via a reserved IRI; never a server's standing default).

### 3(c) JSON-LD inner named graphs — **FLATTEN in v1** (RESOLVED §9.1 — reification (D) is the named non-normative roadmap fidelity mode; raw-preserve (B) is a normative MUST NOT)
A Turtle document denotes a single RDF **graph**, but a **JSON-LD document denotes an RDF *dataset***
(JSON-LD 1.1 §1.4: a node object with `@id` + `@graph` yields an *inner named graph* named by the
`@id`). So one JSON-LD resource *X* can parse to default-graph triples **plus** inner named graphs —
breaking the clean `X ↦ <X>` bijection. Options:

- **A — FLATTEN:** collapse the resource's whole dataset (default + all inner graphs, inner names
  dropped) into the single graph `<X>`. Lossy on inner-graph names; **zero collisions; one clean
  WAC verdict per resource.**
- **B — PRESERVE raw** (inner `@id` ⇒ a first-class SPARQL named graph): **a security hole** — two
  resources declaring the same inner `@id` commingle under one graph name that WAC decides as a
  unit; an inner `@id` can even equal another resource's IRI `<Y>`, injecting triples into it.
  **Reject B.**
- **C — HYBRID** (namespace each inner graph under its owner + a provenance map so one WAC verdict
  per resource still holds): faithful + safe, but needs a name-rewrite + provenance table.
- **D — REIFICATION** (keep the inner-graph structure as *data* inside the single graph `<X>`):
  cleaner than C (no separate graphs, no collision, one WAC verdict). Two forms — **RDF-star
  annotation** `<< :s :p :o >> :inGraph <g>` (clean; but RDF-star = **RDF 1.2 / SPARQL 1.2**, which
  we made *optional*, so this scopes a 1.2 dependency onto the inner-graph-query feature), or
  **classic RDF reification** (works in 1.1 but verbose, non-asserting → duplication, pollutes the
  flat view).

**RESOLVED for v1: FLATTEN (A).** Rationale: (i) it **exactly matches** what `solid-server-rs` does
today (`src/ldp/content.rs` keeps only default-graph quads and **drops named-graph quads at parse** —
the deliberate "an RDF resource is one graph" LDP choice), so the spec conforms to the endpoint;
(ii) it is **collision-free** and keeps the resource↦graph↦WAC invariant airtight; (iii) JSON-LD
`@graph` nesting is rare in Pod data.

**Key reframe (shrinks the fidelity loss):** the **document round-trips byte-exact on GET regardless**
of the query mapping (byte-exact resource storage), so flatten does **not** lose the JSON-LD document
or its inner graphs — a plain GET returns the original untouched. Flatten only affects what a SPARQL
*query* sees / can `CONSTRUCT`. So the only thing at stake is whether inner-graph structure is
**queryable**.

**RESOLVED (§9.1, Claude Fable):** A (flatten) is the v1 normative rule; D (RDF-star reification)
is specified as a **named, non-normative roadmap fidelity mode** (an informative Editor's-Draft
section sketching the future opt-in — it supersedes C); and B (raw-preserve) is a **normative
MUST NOT** in the draft, with the recorded rationale that it enables cross-resource
graph-name collision/injection and breaks the one-WAC-verdict-per-resource invariant.

## 4. Cross-graph join semantics (the reminder)

Empty default graph; readable resources are named graphs. There is **no `FROM *` / `FROM ?g`** in
SPARQL — `FROM`/`FROM NAMED` take fixed IRIs. The "query over all graphs" capability is `GRAPH ?g`,
which ranges over exactly the caller's **readable** named graphs.

- `{ ?s ?p ?o }` → empty default graph → **0 results**.
- `{ GRAPH ?g { ?s ?p ?o } }` → every triple across all readable graphs (`?g` = each graph).
- **Intra-graph join** (both patterns in one block) — `{ GRAPH ?g { ?p foaf:knows ?f . ?f foaf:name ?n } }`
  → both triples must be in the **same** graph `?g`.
- **Cross-graph join** (separate blocks, distinct graph variables) —
  `{ GRAPH ?g1 { ?p foaf:knows ?f } GRAPH ?g2 { ?f foaf:name ?n } }`
  → the two triples may come from **different** graphs, joined on the shared `?f`.
- Or merge specific graphs into a per-query default graph: `SELECT … FROM <A> FROM <B> WHERE { … }`
  — `<A>`+`<B>` RDF-merged; an unreadable/absent `FROM` target is just **absent** (§5, §6).

**Rule:** a join inside a single `GRAPH` block is intra-graph; cross-graph joins need either separate
`GRAPH` blocks or a `FROM`-merged default graph. (A true "union of all readable graphs as the default
graph" so bare BGPs join across everything is the OPEN opt-in union-default-graph mode of §9.)

## 5. Access-control semantics (RESOLVED)

- **Results = union of the named graphs for resources on which the requester holds `acl:Read`**
  (WAC/ACP `accessTo`/`default` walk, fail-closed). Only the **Read** mode governs visibility.
- **Unauthorized *and* non-existent graphs behave as ABSENT** — the existence-non-disclosure
  principle. No oracle may leak an unreadable graph via **bindings / `ASK` / counts / `GRAPH ?g`
  enumeration**, via **status/error differences** (no 403-vs-404 inside the dataset), or via
  **timing**. Enforce at the **graph-name layer** — restrict the dataset to the readable set, *then*
  evaluate — never post-filter a full-store result (that reopens the count/timing oracle).
  `FROM`/`FROM NAMED` on an unreadable/missing graph → treated as an empty graph (no error).
- **Authentication carriage:** the SPARQL Protocol request is an ordinary HTTP request whose
  authentication is processed by the same Solid Protocol authentication layer as any LDP resource
  request. The specification deliberately leaves the mechanism and failure disposition to that
  layer; the authenticated agent it establishes is used for WAC/ACP authorization, while failed
  credentials establish no authenticated agent or permissions. As a non-normative example, a
  deployment can use Solid-OIDC with DPoP-bound access tokens, but neither Solid-OIDC nor DPoP is a
  conformance requirement of this specification.
- **SPARQL UPDATE is explicitly OUT for v1** (read-only query only). If v2 adds it: `WHERE⇒Read`,
  `INSERT⇒Append`, `DELETE⇒Write`, each per-target-graph, and the `DELETE`/`WHERE` read-oracle must
  be closed. Deferred to a future ADR.

## 6. Resolved open questions (from the design Q&A)

| # | Question | Decision |
|---|---|---|
| 1 | Endpoint location / discovery | **Per-Pod `/sparql`**. Mint a link relation → a Service-Description GET (no `.well-known/sparql` standard exists). |
| 2 | Default graph | **Empty**; cross-graph via `GRAPH ?g` (§4). Union-default-graph mode = **RESOLVED: in v1 as an explicit per-query opt-in** (§9.2 — reserved IRI `solid-sparql:union-default-graph`; MUST NOT be a standing default; advertised via the minted `sd:Feature` `solid-sparql:UnionDefaultGraphOptIn`). |
| 3 | JSON-LD inner graphs | **Flatten (A)** for v1 — **RESOLVED** (§9.1): reification (D) = named non-normative roadmap fidelity mode; raw-preserve (B) = normative MUST NOT. |
| 4 | Query resource limits | **Left to the server** — `MAY` limit; `SHOULD` respond `503` (or a defined error) when exceeded. |
| 5 | `FROM`/`FROM NAMED` on unreadable/missing graph | **Act as if the graph does not exist** (empty, no error, no leak). |
| 6 | Service-Description leakage | The SD **MUST NOT enumerate graphs the requester can't Read** (the graph names *are* the resource IRIs → enumerating them leaks the private resource list). Resolution: **omit the named-graph inventory** (advertise endpoint + capabilities + formats only); clients discover readable graphs via `GRAPH ?g` at query time. If ever listed, per-requester-filtered. |
| 7 | Result formats | **Full SPARQL 1.1 set** — `application/sparql-results+json`, `+xml`, `text/csv`, `text/tab-separated-values` (SELECT/ASK); RDF (Turtle, JSON-LD, …) for CONSTRUCT/DESCRIBE. Defer to the SPARQL spec. |

## 7. Conformance to `solid-server-rs`

Matches the server's **models** (one named graph per resource IRI; per-request per-resource
fail-closed WAC with 401/403 + existence-non-disclosure already at the LDP layer — see the
`phase-existence-non-disclosure` branch / PR #3; JSON-LD inner graphs flattened at parse = 3(c)/A),
but **exceeds its current surface**: there is **no client-facing `/sparql` route**, no `query`/
`query_as` on the `SparqClient` trait, no `PodStore::decide` seam — those are the **`sparq#992`**-gated
work. Realizing v1 requires: (i) an axum `/sparql` route running the 1.1 Protocol query operation;
(ii) the `query_as`/`decide` seam that hands the engine a **WAC-restricted graph set** for the
requester and evaluates over it; (iii) the `embedded` backend (real `GRAPH` isolation) as the
reference path (the live HTTP `sparq-server` still folds named graphs into one default graph —
DEVIATION-1 / FR-4). **This design doc doubles as the precise requirement for `sparq#992`.**

## 8. Proposed spec outline (the full document)

1. Introduction — motivation, relationship to Solid Protocol & WAC/ACP, non-goals
2. Terminology & normative references (SPARQL 1.1 Protocol/Query, SPARQL 1.1 Service Description,
   RDF 1.1/1.2 Concepts, RDF 1.1 Datasets, Turtle, JSON-LD 1.1 + API, Solid Protocol, WAC, ACP;
   Solid-OIDC and RFC 9449 DPoP appear only as a non-normative authentication example)
3. Conformance classes & keywords (RFC 2119/8174)
4. The Query Endpoint — per-Pod location, discovery, SPARQL 1.1 Protocol bindings (MUST), 1.2
   forward-compat (MAY)
5. Pod-content-to-Dataset mapping — resource↦named-graph; empty default graph; RDF-format handling
   (Turtle=graph, JSON-LD=dataset, the flatten rule); (roadmap: reification mode)
6. Authentication — mechanism-agnostic delegation to the Solid server's authentication layer,
   with Solid-OIDC + DPoP only as a non-normative example
7. Authorization & the authorized dataset — WAC/ACP Read at the graph-name layer; per-request dataset
   construction; `FROM`/`FROM NAMED` = absent-if-unreadable
8. Existence non-disclosure — results/error/timing invariants; `GRAPH ?g`, `ASK`, `FROM/FROM NAMED`,
   and Service-Description leak rules
9. Result formats & content negotiation (full SPARQL 1.1 set)
10. Service Description (capabilities only; no graph inventory)
11. Security & privacy considerations (query cost / DoS, the read-oracle class)
12. SPARQL UPDATE — out of scope for v1 (future-work note + the mode-mapping sketch)
13. Conformance test scenarios
- Appendix: worked examples (SELECT across readable graphs; a JSON-LD-with-inner-graphs resource
  under the flatten rule; an unauthorized-graph-is-absent test vector; intra- vs cross-graph joins)

## 9. RESOLVED CALLS (decided by Claude Fable, 2026-07-01; accepting both Opus leans)

Both formerly-open calls are decided and baked into the Editor's Draft (`../index.html`).

### 9.1 JSON-LD inner named graphs — FLATTEN (A) in v1; reification (D) = named non-normative roadmap mode; raw-preserve (B) = normative MUST NOT

- **FLATTEN (A) is the v1 normative rule** (draft §"Concrete syntaxes and the flattening rule"):
  when a resource's representation deserializes to an RDF dataset, the resource graph is the union
  of the default graph and all inner named graphs' triples, inner graph names discarded *as graph
  names* (triples mentioning them survive as ordinary triples).
- **RDF-star reification (D) is specified as a named, non-normative ROADMAP fidelity mode** — an
  informative draft section ("Roadmap: a named-graph fidelity mode") sketching the future opt-in
  (`<< s p o >> …:inGraph <g>`-style annotation inside the one resource graph). It supersedes C.
- **Raw-preserve (B) is a normative MUST NOT** in the draft, with the recorded rationale: promoting
  an inner `@id` to a first-class SPARQL named graph lets one resource's *content* choose arbitrary
  graph names — enabling **cross-resource graph-name collision/injection** (an inner `@id` equal to
  another resource's IRI injects triples under its graph name) — and **breaks the
  one-WAC-verdict-per-resource invariant** (two resources commingled under one name that WAC
  decides as a unit).
- **Recorded rationale for flatten-in-v1:** (i) v1 conforms to the reference implementation —
  `solid-server-rs` drops named-graph quads at parse (`src/ldp/content.rs`); (ii) consistency with
  Decision 1 (§2) — the spec refused a pre-CR SPARQL 1.2 dependency at the protocol layer, so the
  mapping layer must not smuggle an RDF 1.2 *normative* dependency in via reification; (iii)
  resources round-trip byte-exact on GET, so flatten loses no data — only SPARQL query-visibility
  of inner-graph names, a rare structure in Pod data; (iv) a fidelity mode can be added compatibly
  later as an additive opt-in (stored documents need no migration).

### 9.2 Union-default-graph mode — INCLUDED in v1 as an explicit per-query OPT-IN

- Servers **MAY** implement it; a requester **MUST explicitly request it per query**; it **MUST NOT
  be a server's standing default** (the standing default graph remains empty per §3b).
- **Mechanism:** the spec mints the reserved IRI
  **`http://www.w3.org/ns/solid/sparql#union-default-graph`**, usable (a) as a value of the SPARQL
  1.1 Protocol `default-graph-uri` parameter and (b) as a `FROM` target in the query text; either
  form requests that the default graph *for that query* be the RDF merge of all named graphs the
  requester is authorized to Read. In `FROM NAMED`/`named-graph-uri` position the reserved IRI
  names nothing (absent, like any unreadable/missing target); `GRAPH ?g` never binds to it.
- **Advertisement:** the spec-minted `sd:Feature`
  **`http://www.w3.org/ns/solid/sparql#UnionDefaultGraphOptIn`** in the Service Description —
  explicitly **NOT** the standing `sd:UnionDefaultGraph` dataset property, since that would falsely
  describe the endpoint's default dataset (which is empty-default).
- **Security analysis (recorded in the draft):** the union's content is exactly the information
  already exposed via `GRAPH ?g`, so the mode adds **no new disclosure oracle**; authorization is
  still enforced at the graph-name layer before evaluation, and the reserved IRI behaves like any
  other `FROM` target (absent-if-not-requested / absent-if-unsupported semantics never leak).

## 10. Status + next steps (post-draft)

- The **Editor's Draft is written** (`../index.html`, ReSpec CG-DRAFT) on the resolved decisions
  §2, §3, §5, §6, §9 — every load-bearing call in this document is now reflected in normative text.
- **Server contract mirror:** [`sparq-992-mirror.md`](sparq-992-mirror.md) states the server-side
  contract (`query_as`/`decide` seam) for **`jeswr/sparq#992`** + `jeswr/solid-server-rs`; it is
  written to be pasted onto that issue.
- **Conformance is symbiotic:** the spec's authorized-dataset construction (§5) and the
  existence-non-disclosure invariants (§8 of the draft) are the same principles landed in
  `solid-server-rs` PR #3 — keep them in lockstep.
- **Contribute upstream:** once solid + reviewed, this is destined for the **Solid CG** and
  `solidproject.org/TR/`; coordinate with the CG (identify as the maintainer's agent).
