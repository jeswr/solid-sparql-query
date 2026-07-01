<!-- AUTHORED-BY Claude Opus 4.8 -->
# Design — Access-Controlled SPARQL Query over a Solid Pod

> Design brief + resolved decisions for a **W3C Solid Community Group specification**, destined for
> <https://solidproject.org/TR/>, alongside the Solid Protocol and WAC. Conforms to
> [`jeswr/solid-server-rs`](https://github.com/jeswr/solid-server-rs). Authored with Claude Opus 4.8;
> **two design calls remain open** (§9) for the next editor to make before the Editor's Draft.

## 1. Scope & goal

Define an HTTP endpoint that answers **read-only SPARQL queries over a Pod's RDF content**, returning
**only the triples the authenticated agent is authorized to Read**. It is a new *read interface* over
the existing WAC/ACP + Solid-OIDC/DPoP substrate — not a new authorization model and not a new wire
protocol. It fills a real gap: Solid has no ratified server-side query surface today (querying is
client-side, via Comunica link-traversal). Target: a Solid CG **Editor's-Draft-track** document.

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
- **Access control is layered on the standard protocol** — Solid-OIDC/DPoP on the request + a SPARQL
  1.1 Service Description for discovery. **No new protocol is minted**, so every existing SPARQL
  client works.
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
is `GRAPH ?g`). An **opt-in union-default-graph mode** is an OPEN call (§9).

### 3(c) JSON-LD inner named graphs — **FLATTEN in v1** (reification is the roadmap fidelity mode; OPEN whether to pull it into v1)
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

**OPEN CALL (§9):** if inner-graph *querying* is a v1 requirement, adopt **reification (D, RDF-star
form)** and accept that this one feature requires SPARQL 1.2 — otherwise keep A for v1 and record D
(RDF-star) as the roadmap fidelity upgrade (it supersedes C). *Opus lean: roadmap it (A for v1).*
**Whichever is chosen, any future "preserve" MUST be D or C — never the raw-named-graph B.**

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
- **Auth carriage:** the SPARQL Protocol request is an ordinary HTTP request, so **Solid-OIDC + DPoP**
  ride on it exactly as on any LDP request (`Authorization: DPoP <at+jwt>` + DPoP proof over the
  endpoint; issuer-agnostic verify, `jti` replay, WebID resolve, then WAC).
- **SPARQL UPDATE is explicitly OUT for v1** (read-only query only). If v2 adds it: `WHERE⇒Read`,
  `INSERT⇒Append`, `DELETE⇒Write`, each per-target-graph, and the `DELETE`/`WHERE` read-oracle must
  be closed. Deferred to a future ADR.

## 6. Resolved open questions (from the design Q&A)

| # | Question | Decision |
|---|---|---|
| 1 | Endpoint location / discovery | **Per-Pod `/sparql`**. Mint a link relation → a Service-Description GET (no `.well-known/sparql` standard exists). |
| 2 | Default graph | **Empty**; cross-graph via `GRAPH ?g` (§4). Optional union-default-graph mode = OPEN (§9). |
| 3 | JSON-LD inner graphs | **Flatten (A)** for v1; reification (D) = roadmap. Reification-in-v1 = OPEN (§9). |
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
   RDF 1.1/1.2 Concepts, RDF 1.1 Datasets, Turtle, JSON-LD 1.1 + API, Solid Protocol, WAC, ACP,
   Solid-OIDC, RFC 9449 DPoP)
3. Conformance classes & keywords (RFC 2119/8174)
4. The Query Endpoint — per-Pod location, discovery, SPARQL 1.1 Protocol bindings (MUST), 1.2
   forward-compat (MAY)
5. Pod-content-to-Dataset mapping — resource↦named-graph; empty default graph; RDF-format handling
   (Turtle=graph, JSON-LD=dataset, the flatten rule); (roadmap: reification mode)
6. Authentication — Solid-OIDC + DPoP on the Protocol request
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

## 9. OPEN CALLS for the next editor (make these first)

1. **Reification in v1, or flatten-now + reification-on-roadmap?** (§3c) — pulling RDF-star
   reification into v1 makes inner-graph structure queryable but scopes a **SPARQL 1.2 dependency**
   onto that feature. *Opus lean: roadmap it.*
2. **Union-default-graph mode?** (§3b/§4) — an opt-in so bare BGPs join across all readable graphs as
   one default graph (a reserved union-graph IRI usable in `FROM`, or a request option). Include in
   v1 or defer? *Opus lean: offer it as an explicit opt-in, not the default.*

Both are genuine design calls, not blockers — the Editor's Draft can be written on the resolved
decisions and these two slotted in.

## 10. Notes for the next editor (Claude Fable)

- **Everything load-bearing is decided** (§2, §3, §5, §6); only §9's two calls remain, and both have
  an Opus lean you can accept or overrule with reasons.
- **Deliverable:** the ReSpec (or Bikeshed) **Editor's Draft** matching this design, then mirror the
  server contract (§7) into **`jeswr/sparq#992`** as the precise `query_as`/`decide` requirement.
- **Conformance is symbiotic:** the spec's authorized-dataset construction (§5) and the
  existence-non-disclosure invariants (§8) are the same principles landed in `solid-server-rs`
  PR #3 — keep them in lockstep.
- **Contribute upstream:** once solid + reviewed, this is destined for the **Solid CG** and
  `solidproject.org/TR/`; coordinate with the CG (identify as the maintainer's agent).
