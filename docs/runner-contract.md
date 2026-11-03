# The runner contract

What every runner of these vectors must do — the Go runner in
go-wallet-backend (`test/vectorrun`) and the Rust runner in siros-wallet-core
(`engine-conformance`) implement exactly this, so a vector means the same
thing to both.

## Loading and validation

1. Locate the vectors: the `ENGINE_VECTORS_DIR` override first, then a
   sibling checkout of this repository, then the CI-pinned revision.
2. Validate every file against `vector-schema.json`. A schema violation
   fails the run; no vector runs unvalidated.
3. Reject a go-observed vector whose `review_by` has passed, unless it
   carries a `review_reason` — and fail the run entirely once the hard
   sunset date passes with go-observed vectors still gating.

## Execution

A vector is a script of steps, executed in order against one engine instance
built from `config`, with the world built from `setup`.

- **`wmp_in`** — deliver the message to the engine, exactly as the wallet
  host would. On the Rust engine this is `handle_request_sync` with the
  principal from `setup`; on the Go engine, the in-process bridge.
- **`http_out`** — the engine's next HTTP request must match `expect`:
  method, exact URL (with `*` wildcards, or a `re:` regular expression),
  the selected headers, form and JSON members, the **purpose**, and the size
  and redirect limits the engine declared. The world answers with `reply`.
  With `must_refuse: true`, the engine must make no further request at all.
  The reply's `body` has three forms: a string is raw bytes, a plain JSON
  value is its serialization, and `{"encrypt_to_offer": true, "plaintext": …}`
  builds a JWE at run time from the very `credential_response_encryption`
  JWK the engine's request offered (the declared alg and enc) — an encrypted
  answer cannot be pre-baked, and this form asserts the engine can decrypt
  what it asked for.
- **`host_ask`** — the engine's next sub-flow request must match `expect`:
  the kind and the selected inputs (the to-be-signed bytes, the claim paths,
  the transaction entries). Answer with `answer`. With `no_ask: true`, the
  engine must not ask.
- **`wmp_out`** — the engine's next message out: the method and the selected
  members, with `*` wildcards for engine-generated values. With
  `must_refuse: true`, the engine answers the `error_code` instead.

The step kinds interleave: an `http_out` or `host_ask` step may be satisfied
between two `wmp_in`/`wmp_out` steps, in arrival order. A step that expects
an event and does not see it before the next `wmp_in` fails the vector; a
step whose expectations are met is consumed and removed from the set the
next matching step may satisfy.

## Comparison

- JSON members compare by presence and value, not by order or by set
  equality: an object may carry more members than the expectation names,
  unless the expectation carries `"*"` in a member whose absence is then an
  error. A member value of the object `{"$jwe_header": {…}}` matches a
  JWE compact-serialization string by decoding its first part and matching
  the protected header — the only window into what an engine encrypted,
  since the ciphertext cannot be predicted and only the recipient can read
  the payload.
- A `*` wildcard matches any single value of the named type; `ses-*`
  matches any string with that prefix. A `re:` URL expectation is a
  regular expression over the whole URL.
- Numbers compare by value; `1` and `1.0` are equal.
- Engine-generated values a wildcard matched are **bound by name** for
  later steps: a later step's expectation may name the binding to assert
  the same value recurred (a session id, a nonce the engine must reuse).
  A binding's name is its JSON path within the expectation that captured
  it (`.state`, `.payload.code_verifier`), and a `wmp_in` step's params
  may substitute a binding by that reference — `"state": "$.payload.state"`
  delivers the very value the engine generated, so a two-call transcript
  (the authorization-code grant's redirect answer) checks the CSRF state
  without the vector predicting the seeded entropy.
- Secrets never appear in a vector in the clear; keys live in `setup` and
  answers assert decryptions, not ciphertext bytes.

## Reporting

A run reports, per vector: pass, fail (with the step that failed and the
difference), or skipped (with the reason — a vector may declare a runner
capability it needs in `config`). A run reports counts by provenance, so a
suite that is drifting back toward go-observed scaffolding is visible in the
summary line. A failing vector is triaged, never retried into green.

## What a runner must not do

- **Must not** regenerate vectors, or write to this repository.
- **Must not** special-case an engine: a vector passes for one engine and
  fails for the other only through a `known_divergence`.
- **Must not** log tokens, nonce text, openings or credential data from a
  vector run; the vector's own `notes` are the only text carried forward.
