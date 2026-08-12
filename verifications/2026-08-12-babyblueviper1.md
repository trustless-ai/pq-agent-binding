# Independent verification — 2026-08-12

**Verifier:** babyblueviper1 (Fede), invinoveritas
**Subject:** `kya-l4-genesis`, the in-force binding served at
`https://gateway.ensub.org/.well-known/pq-key-binding.json`
**Recompute target:** `trustless-ai/recompute-kit` @ tag `pq-key-binding-v0`
**Result:** matches on both lanes, cold.

This is a record of a method, not a claim of correctness. Everything below is
re-derivable by a third party from public data; nothing here asks for the
verifier — or us — to be believed.

---

## What "cold" means here

No shared state with the implementation. The verifier pulled the served artifact
and the on-chain data independently, and used a public RPC rather than this
project's gateway or any summary it produces. Independence is a property of the
method, which is why the method is what this file records.

## Lane 1 — content address

Recomputed from the raw fields in the conformance vector — `schema`,
`secp256k1_pubkey`, `pq_pubkey`, `algorithm`, `bound_at`, `profile` — not from
any pre-computed value:

```
canonical_content        = JCS(statement)          # RFC-8785 / receiptos-c14n
canonical_content_sha256 = sha256(canonical_content)
                         = b26a01590215926373544dc82d22fadc8d97c98debaae5d6ce8899b83ffd05da
```

Byte-exact against **both** the pinned vector and what is served live at the
well-known path.

## Lane 2 — anchor

Queried directly against a public RPC, independent of the gateway:

| | |
|---|---|
| tx | `0x469655a08accf0300def211bf0c9ebd463e65b89f4ede1ac372ed2796e7ba916` |
| status | success |
| from | `0xFf9a176577Fb42b6bc9c19fd05a241e8fCd0ca14` |
| to | `0x1e2A118a2bf1C240aE6fDe187c07f905D360f094` (TruthAnchor, ERC-8281) |
| block | 25646404 |
| log topic | carries `b26a0159…ffd05da` — the content address derived above |

The log also carries the committer as an indexed topic, equal to the sender.

**The load-bearing consequence:** the statement *names* a classical key, and the
anchoring transaction was *sent by* that key. Sender-of-record is therefore the
classical proof-of-possession, established from chain data rather than asserted
by anyone. That is the property the dual-family design depends on, and it is now
checked by a second party.

## Re-deriving this yourself

1. Fetch `https://gateway.ensub.org/.well-known/pq-key-binding.json`
2. Take the statement fields, canonicalize with JCS, sha256 → compare to the
   content address above
3. Query the tx against any Ethereum RPC → check `from`, `to`, status, and that
   a log topic equals the content address
4. Cross-check against `recompute-kit` @ `pq-key-binding-v0`

## What this does NOT establish

One binding, at one time. It says nothing about rotation or revocation behaviour,
nothing about the cutoff enforcement path, and nothing about future bindings. It
is also not a review of the implementation — no code was read, deliberately.

## Prior context

This is a second independent read. The first was the original cold
recompute-verification that the shared `pq-key-binding-v0` profile is built on.
The verifier recomputed across a production migration without being told it had
happened — the gateway moved to its own repository hours earlier — and matched
byte-exactly on both sides.
