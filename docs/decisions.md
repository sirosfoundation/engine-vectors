# The recorded engine decisions

The places where a published specification allowed the engine a choice,
or where the two engines disagree and the disagreement has been decided
rather than copied. Every vector that tests one of these cites the
decision's `d.*` identifier in its `spec_refs`; the provenance is `spec`
when a clause demands the behaviour, `policy` when it is a recorded
SIROS decision.

A decision here is normative for both engines. The Go engine's current
behaviour where it differs is a `known_divergence` on the vector, not a
second valid answer.

---

## d.vci.plain-response-after-key — spec

**The decision.** When the wallet offered a
`credential_response_encryption` key in its Credential Request, a plain
(unencrypted) 200 answer from the credential endpoint is **refused**.

**The clause.** OpenID4VCI §7.3 (Security considerations): when the
Wallet provides encryption parameters, the Issuer MUST encrypt the
credential response.

**The divergence.** The Go engine accepts the plain response (P0.6
verification, `go-wallet-backend` `requestCredential`: the encrypted
branch is only entered on the JWE content type, and a plain 200 passes
through). The Rust core follows the spec; vectors for this carry a
`known_divergence` until the Go engine is fixed.

**Affected paths.** The credential request (pre-authorized and
authorization-code grants alike), the deferred poll (an encrypted
deferred answer goes through the same decrypt).

## d.vci.encrypted-without-key — spec

**The decision.** A JWE-encrypted credential response when the wallet
offered no key is **refused** — there is nothing to decrypt it with,
and accepting it would mean the engine invented a key.

**The clause.** OpenID4VCI §7.3: the response is encrypted to the key
the *Wallet* offered; no key, no encrypted response.

**The divergence.** The Go engine fails this path too, but late and
opaquely (the JWE branch cannot decrypt). The Rust core refuses at the
contract level. No divergence in outcome.

## d.vci.metadata-endpoints — spec

**The decision.** The credential, notification, token and nonce
endpoints come from the issuer's metadata members
(`credential_endpoint`, `notification_endpoint`, `token_endpoint`,
`nonce_endpoint`). A constructed path (`{issuer}/credential`, …) is
only a fallback when the metadata did not name the member — never the
primary.

**The clause.** OpenID4VCI §11.2.2: the Credential Issuer Metadata
defines these members; the wallet resolves endpoints from metadata, not
from URL construction.

**The divergence.** None in the Go engine (it is metadata-driven); this
decision records that the Rust core had to be brought to it, and pins
the vectors that keep it there.

## d.vci.nonce-retry-once — spec

**The decision.** On `invalid_nonce` (or any credential-endpoint error
body carrying a refreshed `c_nonce`), the fresh nonce comes from the
issuer's nonce endpoint (§7) first, then the error body's `c_nonce`
(the legacy fallback); the proofs are re-requested and the request
retried **exactly once**; a second `invalid_nonce` is terminal with the
issuer's own error.

**The clause.** OpenID4VCI §7.5 (the nonce is single-use per request)
and the draft's c_nonce semantics; the retry-once bound is the
P0.6-verified Go behaviour.

**The divergence.** None; the Go engine behaves identically.

## d.vp.jarm-enc-default — policy

**The decision.** When the verifier's client metadata declares no
`authorization_encrypted_response_enc`, the default is **A128GCM** —
not A128CBC-HS256, RFC 7518's first mandatory-to-implement `enc`.

**Why.** Confirmed live against verifier.multipaz.org (the Go engine's
own finding): its JsonWebEncryption decrypter implements only the GCM
family and rejects CBC-HS256 outright, without declaring
`encrypted_response_enc_values_supported` to signal the restriction.
GCM is equally spec-valid as a default and has the broader real-world
interop.

**The divergence.** None; the Go engine made the same choice. Recorded
as policy because the spec itself is silent on the default.

## d.vp.jarm-rsa-inference — policy

**The decision.** When `authorization_encrypted_response_alg` is absent
and the verifier's key is RSA, the inferred algorithm is
**RSA-OAEP-256** — not RSA-OAEP (SHA-1).

**Why.** SHA-1 OAEP is out of this stack: jose-rs 0.10 ships it only
behind the `deprecated` feature, which the engine core pins off (the
DEPENDENCIES.md posture — the same pin siros-verifier-core ships with).
RSA-OAEP with SHA-1 is deprecated by the current JOSE security
considerations.

**The divergence.** The Go engine infers RSA-OAEP (SHA-1). Vectors
covering an RSA verifier without a declared alg carry a
`known_divergence` for the Go engine.

## d.vp.jarm-refusal-not-sent — spec

**The decision.** When a `direct_post.jwt` request cannot be answered
encrypted (no usable key, no algorithm, an encryption failure), the
refusal is **not sent to the verifier** — the verifier times out; the
flow error goes to the wallet host.

**The clause.** OpenID4VP §6.2 with JARM: the response to a
direct_post.jwt request is by construction an encrypted JWT; there is no
defined plain error form for that channel, so a client that cannot
encrypt must not fabricate one.

**The divergence.** None in outcome; this pins the P0.6 correction that
the *decline* (a plain form POST) applies only to direct_post, never to
direct_post.jwt.

## d.vci.wia-via-wallet-provider — policy

**The decision.** The Wallet Instance Attestation used for
OAuth-Client-Attestation comes from the **existing go-wallet-backend
wallet provider** (`POST /wallet-provider/wia/challenge`, then
`POST /wallet-provider/wia/generate`), obtained by the wallet host
(the SDK), not minted or fetched by the engine. The engine asks the
host for the (WIA, PoP) pair — the Go engine's
`SignActionRequestAttestation` contract — because the instance key that
signs both PoPs (the WIA-PoP for the wallet provider, the per-flow
OAuth-Client-Attestation-PoP for the AS) lives on the device.

**Why.** The wallet provider is the WIA authority — its challenge
store, instance registry, revocation gate and signing key are the
trust anchor. A second WIA source would fork that registry.

**The retry.** On `use_attestation_challenge` (a 400/401 with that
error code), the engine resolves the fresh challenge from the
`OAuth-Client-Attestation-Challenge` response header, else the AS's
`challenge_endpoint` (POST; the header on the answer, else the
`attestation_challenge` body member), re-asks the host for a PoP bound
to it, and retries the request once.

**The divergence.** None; this is the Go engine's own contract, carried
into the Rust core's `Host::request_attestation` port.

**Affected paths.** The PAR push and the code-for-token exchange of the
authorization-code grant.
