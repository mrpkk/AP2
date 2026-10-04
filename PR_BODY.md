## Summary

Signed AP2 mandate fixtures under `contrib/mandate-corpus/`, plus a second verifier in a different language. No SDK source file is modified.

A corpus that claims other languages can use it is only a claim until an independent implementation reads it. This supplies that implementation.

## The gap

There is no `fixture`, `corpus`, `vector` or `conformance` directory in this repository:

```
find . -type d \( -name "*fixture*" -o -name "*corpus*" -o -name "*vector*" -o -name "*conformance*" \)
→ (empty)
```

The Python SDK can sign mandates (`sd_jwt.create`, `kb_sd_jwt.create`), but nobody publishes signed fixtures. Consequently the positive test in `Universal-Commerce-Protocol/conformance` is skipped; PR #115 there states:

> "A real positive needs a signed SD-JWT+kb mandate fixture and a signing model, which this PR does not attempt."

## Why this blocks the ecosystem, measured 2026-10-04

| language | lines | signing crypto | mandate support |
|---|---|---|---|
| **Python** (this repo) | 8 863 | 20 modules | works |
| **Go** (`code/samples/go`) | 2 258 | **0** | none — `FakeJWT` |
| **PHP** (`ucp-php-sdk`) | 39 194 | present | PR #90, **unmerged since 2026-07-15** |

The Go sample hardcodes a token and contains zero files performing real signing or verification:

```
code/samples/go/pkg/roles/merchant_agent/tools.go:31
FakeJWT = "eyJhbGciOiJSUz1NiIsImtpZIwMjQwOTA..."
```

**Python was the only language able to verify an AP2 mandate**, so no other implementation could be tested. That is why the corpus alone was not enough, and why this adds a Go verifier.

## Contents

| Path | Purpose |
|---|---|
| `tools/generate_fixtures.py` | reproducible signed fixture generator |
| `tools/verify_fixtures.py` | reference verifier, Python |
| `tools/negative_vectors.py` | negative vectors plus a positive control |
| `tools/poc_aud_not_enforced.py` | reproducer for the defect below |
| `go/sdjwt/`, `go/cmd/verify/` | **second verifier, Go standard library only** |
| `crosscheck.py` | Python versus Go, must agree |
| `vectors/` | three signed vectors plus `index.json` |

Vectors: `card-basic` (100 USD), `debit-large` (50 000 minor units), `nonusd` (BRL) — the last catches implementations that silently normalise every currency to dollars.

## Cross-language agreement

```
$ python contrib/mandate-corpus/crosscheck.py

vector                                      python   go       agree
────────────────────────────────────────────────────────────────────────
open-payment-mandate-card-basic             ACCEPT   ACCEPT   yes
open-payment-mandate-debit-large            ACCEPT   ACCEPT   yes
open-payment-mandate-nonusd                 ACCEPT   ACCEPT   yes
────────────────────────────────────────────────────────────────────────
  agreements: 3/3, divergences: 0
```

`crosscheck.py` runs in CI, so agreement is **enforced rather than observed**. A change that breaks the chain in one language but not the other fails the build — the failure mode a conformance corpus exists to prevent.

The Go verifier uses the standard library only, deliberately: the claim "Go cannot do this" should not be answerable by wiring in a dependency.

### Wire format

Taken from the reference SDK rather than assumed, which mattered in three places:

```
compact token     header.payload.signature~disc~disc~...~
issuer_jwt        parts[0]      disclosures  parts[1:-1]
disclosure digest base64url(sha256(raw disclosure string))
sd_jwt            issuer_jwt + '~' + join(disclosures, '~') + '~'
sd_hash           base64url(sha256(sd_jwt))
placeholder form  {"...": digest} is replaced by the decoded object
ES256 input       header.payload, without the signature
```

## 2. Defect found: `expected_aud` is not enforced

**Impact:** audience check bypass. **Severity:** medium. Reported rather than patched, because changing verification semantics is a maintainer decision.

`verify_chain` documents:

> "index i (KB-SD-JWT): verified with the previous hop's `cnf.jwk` via `kb_sd_jwt.verify`, **enforcing `expected_aud` / `expected_nonce` on the terminal hop when provided**"

But `kb_sd_jwt.verify` only calls `common.verify_expected_claims` when `typ in TYP_TERMINAL`:

```python
TYP_TERMINAL     = ['kb+sd-jwt',    'kb-sd-jwt']
TYP_INTERMEDIATE = ['kb+sd-jwt+kb', 'kb-sd-jwt+kb']
```

A token with `typ='kb+sd-jwt+kb'` — which the SDK itself produces whenever the payload carries a `cnf` — falls into `TYP_INTERMEDIATE`. The `aud`/`nonce` check therefore never runs, even when the caller passed `expected_aud`. No warning is emitted.

```
  hop type      : kb+sd-jwt+kb (per TYP_INTERMEDIATE)
  aud in token  : https://attacker.example/ap2
  expected_aud  : https://honest-merchant.example/ap2
  audiences do NOT match — a correct verifier must reject

  ACCEPTED a token with the wrong audience.

  VERDICT: DEFECT CONFIRMED
```

Possible fixes: evaluate `expected_aud`/`expected_nonce` against the **last** token regardless of `typ`; or widen `TYP_TERMINAL`; or raise when `expected_aud` is supplied but the hop type is not terminal.

The Go verifier **deliberately omits the same check**, mirroring upstream behaviour, so that the two agree. When the SDK is fixed the identical check belongs there.

## 3. A subtlety worth recording

The root SD-JWT must carry a `cnf` naming the agent key **before** the human signs. Otherwise the human authorises a spending authority nobody can use. The same defect is described independently in `openmobilehub/credentagent` PR #189:

> "the human signed over a key that was then discarded and replaced at authorization: they authorized a spending authority nobody would ever use"

Both verifiers check this and reject a root without `cnf.jwk`.

## 4. Verification

```bash
python3 -m venv .venv && .venv/bin/pip install -e .
.venv/bin/python contrib/mandate-corpus/tools/verify_fixtures.py
.venv/bin/python contrib/mandate-corpus/tools/negative_vectors.py
.venv/bin/python contrib/mandate-corpus/crosscheck.py
```

Both verifiers report `3/3 passed verification`; cross-check reports `3/3 agreements, 0 divergences`.

The workflow declares least-privilege permissions: it only reads a checkout and runs local code.

## Scope and limits

- Vectors are validated by **two** implementations, but both by the same author. Independent authorship would be stronger.
- The Go verifier covers the happy path plus `cnf` and `sd_hash`. It does **not** cover the negative vectors yet.
- No `iat`/`exp` checks.
- Section 2 reflects AP2 HEAD as of 2026-10-04.

## Licence

Apache-2.0, matching this repository. Independent work; no affiliation with Google LLC.
