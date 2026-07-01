<!-- AUTHORED-BY Claude Fable 5 -->
<!-- Server-side contract mirror for jeswr/sparq#992 — written to be pasted as a GitHub comment
     on that issue (do not edit the sign-off). Source of truth: the Editor's Draft at
     https://github.com/jeswr/solid-sparql-query (index.html) + design/DESIGN.md §9. -->

## Server-side contract: Access-Controlled SPARQL Query over a Solid Pod

The Editor's Draft of the **Access-Controlled SPARQL Query over a Solid Pod** spec is now written
([jeswr/solid-sparql-query](https://github.com/jeswr/solid-sparql-query) — ReSpec CG-DRAFT). This
comment is the concise, self-contained statement of what that spec requires of the **server side**
— the `query_as` / `decide` seam split between `jeswr/sparq` (the engine) and
`jeswr/solid-server-rs` (the Solid HTTP layer). It supersedes the earlier informal sketch.

### The dataset model

- One SPARQL **named graph per RDF resource**, graph name = the resource IRI, exactly.
- The **standing default graph is EMPTY**. A bare `{ ?s ?p ?o }` matches nothing unless the query
  explicitly opts in (below). All resource content is reachable only through named graphs.
- JSON-LD resources that parse to a dataset are **flattened** into their single resource graph
  (inner named-graph names dropped as graph names). An inner graph name MUST NEVER surface as a
  first-class SPARQL graph name (cross-resource graph-name collision/injection; it would break the
  one-authorization-verdict-per-resource invariant). `solid-server-rs` already does this at parse.

### What the engine seam must provide

1. **Per-request construction of the WAC-restricted graph set.** For each query request, the Solid
   layer resolves the requester (WebID or public) and computes the set of graph names (= resource
   IRIs) the requester may Read (WAC `acl:Read` via the `accessTo`/`default` walk, fail-closed;
   ACL auxiliary graphs only under `acl:Control`; ACP Read where ACP governs). The engine is handed
   this set — the **authorized dataset**: empty default graph + exactly those named graphs.
2. **Evaluation over exactly that dataset.** The engine MUST evaluate the query against the
   authorized dataset and nothing larger. Evaluating over the full store and post-filtering
   solutions is NOT acceptable, even when the final bindings coincide: intermediate cardinalities,
   aggregate inputs, resource-limit behaviour, and evaluation time all become oracles over
   unreadable data. Restriction happens at the **graph-name layer, before evaluation**.
3. **`GRAPH ?g` ranges over only the readable set.** No binding, enumeration, or result content may
   name or be derived from a graph outside the authorized dataset.
4. **Empty-default + explicit-union semantics.**
   - Default graph: empty, always, unless the query explicitly requests otherwise.
   - The spec mints the reserved IRI `http://www.w3.org/ns/solid/sparql#union-default-graph`.
     When it appears among the query's default-graph IRIs (via the `default-graph-uri` protocol
     parameter or a `FROM` clause), the default graph *for that query* = the RDF merge of all
     named graphs of the authorized dataset. The mode is per-query opt-in ONLY — never a standing
     or configurable default (`sd:UnionDefaultGraph` MUST NOT be advertised; the opt-in is
     advertised as the spec-minted `sd:Feature`
     `http://www.w3.org/ns/solid/sparql#UnionDefaultGraphOptIn`).
   - In `FROM NAMED`/`named-graph-uri` position the reserved IRI names nothing (treated as
     absent); `GRAPH ?g` never binds to it. When only the reserved IRI is given (no explicit
     named-graph IRIs), the query's named-graph set remains the authorized set, so `GRAPH`
     patterns stay usable alongside the union default graph.
5. **Non-disclosure as ENGINE-level requirements.** Unreadable and non-existent graphs MUST be
   observationally equivalent:
   - `FROM` / `FROM NAMED` / `default-graph-uri` / `named-graph-uri` naming an
     unreadable-or-missing graph resolve to **absent** — no triples contributed, no named graph
     added, and critically **NO error**: the engine must not raise "no such graph" or any
     distinguishable status/message for a dataset reference it cannot honour. Response status,
     shape, and errors must be identical to the same query referencing a graph that does not
     exist.
   - No oracle via bindings, `ASK`, aggregates/counts, `GRAPH ?g` enumeration, `DESCRIBE`
     content, resource-limit behaviour, or (SHOULD-level) timing. Items 1–2 make most of this
     structural: an engine that never touches unreadable graphs cannot leak them.
6. **Real named-graph isolation in the query backend.** The contract presumes genuine per-graph
   isolation (the `embedded` backend's `GRAPH` semantics). The live HTTP `sparq-server` folding
   named graphs into one default graph (DEVIATION-1 / FR-4) is incompatible with this contract and
   with the empty-default rule — the client-facing endpoint must not ship on that path.

### Protocol wrapper (the `solid-server-rs` half)

The endpoint is a standard **SPARQL 1.1 Protocol query service** (all three bindings; protocol
dataset parameters take precedence over `FROM`/`FROM NAMED`; SPARQL 1.2 optional, never required),
authenticated exactly like any other Solid resource request (Solid-OIDC + DPoP), returning the
full SPARQL 1.1 result-format set, with a capabilities-only Service Description that MUST NOT
enumerate graph names (graph names are resource IRIs — an inventory is a private-resource
listing). SPARQL Update MUST NOT be executed (read-only v1). Full normative detail, worked
examples, and 16 conformance test scenarios are in the Editor's Draft.

🤖 PSS agent — @jeswr's agent for `prod-solid-server` / the Solid app+Pod-Manager suite
