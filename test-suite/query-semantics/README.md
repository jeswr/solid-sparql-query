# Query-semantics conformance suite

A machine-readable conformance test suite for the **query-semantics conformance class** of
the *Access-Controlled SPARQL Query over a Solid Pod* Editor's Draft ([`../../index.html`](../../index.html)).

The **query-semantics** class is the part of the specification that a **query engine**
implements once it has been handed a WAC/ACP-authorized dataset: the pod-content-to-dataset
mapping (§*Resource graphs*), the **empty standing default graph** (§*The default graph is
empty*), the per-query **union default graph mode** opt-in (§*Union default graph mode*), and
the query-observable **non-disclosure invariants** (§*Non-disclosure invariants* /
§*Read visibility*). It deliberately excludes the HTTP-**protocol** class (protocol bindings,
Service Description, SPARQL-Update refusal, caching) and the **content-mapping** class (JSON-LD
flattening / no-raw-preservation) — those are implemented by the Solid HTTP server and the RDF
parser respectively, and are listed under `outOfClassScenarios` in the manifest with the reason
each is out of this class.

## Files

- **`manifest.json`** — the cases. Each case names a spec section + the informative
  `#test-scenarios` scenario number(s) it restates, a requester, a SPARQL query, and an
  expected result. Portable across implementations.
- **`fixture.nq`** — the shared storage as N-Quads (one named graph per resource, WAC ACLs in
  the linked `<resource>.acl` graphs). Mirrors the draft's *Example storage*: requester
  **R** (`https://r.example/card#me`) can Read `/profile/card` and `/contacts/bob` but not
  `/private/journal`.

## Expectation vocabulary

| `expect.type`   | meaning |
|-----------------|---------|
| `rowCount`      | a SELECT/... returns exactly `value` solutions |
| `ask`           | an ASK returns the boolean `value` |
| `count`         | a `SELECT (COUNT(...) AS ?var)` binds `var` to `value` |
| `graphBindings` | the query's `?var` bindings **include** every IRI in `includes` and **exclude** every IRI in `excludes` |

A conforming query service MUST satisfy every case (`requirement: MUST`); `MAY` cases apply
only to a service that implements the optional union default graph mode (the reference
implementation does).

## Running it

The suite is data — any implementation can walk `manifest.json` against its own
authorized-dataset construction. A reference runner over the `sparq` `PodStore` read path
lives at `jeswr/sparq:crates/sparq-solid/tests/conformance_solid_sparql_query.rs`, which
vendors a pinned copy of this directory and asserts all cases pass.
