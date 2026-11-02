# The Phase A generator design

How the first engine vectors are produced: from the Go engine, by a test-only
harness, with the sunset built in. The generator is implemented in
go-wallet-backend **after** the spike's start condition — go-wmp v0.13.0 and
the PR 163 rework landed on `main` — because vectors generated from a moving
reference go stale by construction. The design is written now so the
implementation is a mechanical step the day the gate opens.

## Where it lives

A test-only package in go-wallet-backend, `test/vectorgen`:

- `//go:build ignore`-free and importable only from tests; never imported by
  production code, with a lint or review check that keeps it that way.
- Nothing in `internal/` changes for this effort. If the Go engine needs a
  fix because a vector found a bug, that is an issue for the owner, not a
  patch in the generator's commit.

## What it drives

The generator does not speak a transport. It drives the engine through its
Go interface, in-process, exactly as the backend's own tests do:

- **`httptest` fakes** for the issuer, the authorization server, the
  verifier and the PDP: one handler per scenario, scripted by the same
  `expect`/`reply` steps the vector records, so what the fake enforces is
  literally what the vector says.
- **A recording `http.RoundTripper`** wrapping the engine's HTTP client:
  every outgoing request is captured — method, URL, headers, body, and the
  purpose and limits the engine's resolver pipeline declared, read from the
  request's context values where the engine sets them.
- **A scripted wallet host** implementing the engine's host interface: the
  sign, match, consent, tx_code and authorization steps, answering from the
  vector's `host_ask.answer` and asserting its `expect`.
- **A fixed clock and an injected entropy source**, so a scenario is
  reproducible: `setup.time` and `setup.entropy_seed` enter the engine
  through the same seams the backend's tests already use.

## Determinism: relations, not bytes

The Go engine reads `crypto/rand` and the wall clock; the generator does
**not** add injection points to production code for this. Instead it
asserts relations:

- the PKCE challenge equals S256 of the recorded verifier;
- the DPoP proof verifies under the presented JWK, with the right `htm`,
  `htu`, `ath` and nonce;
- the JWE decrypts under the test key and carries the right header members;
- random-looking values are matched by pattern in the vector and bound by
  name for later steps.

Where a scenario needs exact bytes — a signature's to-be-signed input — the
test supplies the key and the vector records that input, not the signature.
A generated vector is therefore reproducible across runs and engines without
freezing Go's randomness.

## Pinning and regeneration

- Every generation run names the go-wallet-backend commit it generated
  from; the commit goes into each vector's `generator` field.
- Regeneration is a deliberate, reviewed act: the generator prints a diff
  against the checked-in vectors, and a regeneration PR carries that diff
  with a written reason.
- The Go runner (`test/vectorrun`) runs the same files during the overlap,
  so a Go regression is visible while both engines exist.

## What is generated first

The scenario groups of the specification, in the order that de-risks the
spike:

1. **Offer** — by value, by reference, URI scheme form; duplicate and
   case-variant members refused; nothing usable fails at once.
2. **Issuer metadata** — literal path form with the trailing-slash cases;
   404 fallback; 5xx no-fallback; `credential_issuer` mismatch; signed vs
   unsigned per `metadata_type`; 406 fallback; oversize refusal.
3. **AS metadata** — RFC 8414 insertion; OIDC discovery only; 404 then 200;
   5xx and 401 no-fallback; `issuer` mismatch; all-404 endpoint guess;
   served document without a token endpoint.
4. **Pre-authorized issuance** — plain, `tx_code`, encrypted, deferred,
   notification.
5. **Nonce** — endpoint; `invalid_nonce` retry; twice-terminal; legacy
   `c_nonce`.
6. **Client auth** — DPoP with server nonce retry; client attestation.
7. **Auxiliary data** — usable; conflict relayed; plain-with-auxiliary
   refused; unrecognized type ignored.
8. **Trust and failure** — untrusted issuer fails closed; PDP unreachable
   fails closed; SSRF refusal surfaces as a flow error.
9. **Logging** — no token, nonce text, opening or raw body in the recorded
   telemetry of any run.

Each group lands as one generation run, reviewed against the vector schema
before it is committed here, with its `review_by` set at most sixty days out.

## The purpose-assertion problem

The Go engine has no `FetchPurpose` concept: its resolver pipeline fetches
by URL and configuration, and a recorded request's purpose is what the
*generator* infers from the URL and the scenario stage, not something the
engine declared. Generated vectors therefore carry `purpose` as an inferred
annotation, and the runner contract marks it: a go-observed vector's
`http_out.expect.purpose` asserts what a conforming engine *should* declare
for that URL, and the Rust core treats a purpose mismatch as a review
trigger, not an automatic failure. Vectors authored directly (spec, oidf,
policy) assert purposes normatively.
