# pq-agent-binding

*Recomputable post-quantum key binding for on-chain AI agents — bind, anchor, attest, enforce a cutoff, rotate, revoke; every step re-derivable from public data.*

> **Don't trust. Recompute.** — including the migration off Shor-vulnerable signatures.

**Status:** reference spec extracted from a live, shipped implementation (2026-07/08). Intended home: the `trustless-ai` org, once polished. Companion to the recompute-verified `pq-key-binding-v0` shared spec.

---

## Why

Quantum computers break the **signatures** (ECDSA/RSA) under agent identity, finance, and public infrastructure — but leave hashes effectively intact (Grover, not Shor). So the migration problem is specifically the **signature layer**: how does an agent prove an action under a signature scheme that survives Shor — **without asking anyone to trust a provider**, and **without a flag day**?

This binds a post-quantum key to an agent identity such that every step is **independently recomputable from public data**. The only deliberately non-recomputable element is a single standing secret (the master seed), held offline / encrypted-at-rest.

## Design in one line

An agent's classical (ECDSA) identity gains a **PQ companion**: a post-quantum key, **owner-authorized**, **epoch-anchored**, carried on every attestation, and governed by a **load-bearing cutoff** — after which classical-only attestations are rejected — with **owner-driven rotation and revocation**. Dual-family (lattice **and** hash) so no single PQ assumption is load-bearing alone.

## Dual family (the hedge)

Two independent post-quantum families, both finalized NIST standards:
- **ML-DSA** (lattice, FIPS 204) — the primary per-agent signing key.
- **SLH-DSA** (hash, FIPS 205) — a hash-based companion for the attestor/KYA-L4 lane.

If one family is later weakened, the other still stands. Convergence surfaced at `/quantum`; primitives pinned (`@noble` 0.4.1).

## The binding lifecycle

Each step names what is **public + recomputable** vs what is **secret**.

**1. Keygen.** Per-agent ML-DSA keypair (+ SLH-DSA companion for the attestor lane). Public: the public keys. Secret: the private keys, derived from the master seed (the one standing secret; see below).

**2. Owner-authorized binding.** The agent's **owner** (proved via SIWE) authorizes binding a PQ public key to an **ERC-8004** agent id. The binding statement is a canonical record; anyone recomputes `binding_id` from it. No self-declaration — an agent cannot bind its own PQ key without the owner's authorization.

**3. Epoch anchor.** Agent bindings are batched into a **Merkle tree per epoch** and the root is anchored on-chain (amortized: one anchor covers many bindings; per-binding inclusion is a Merkle proof). Public + recomputable: the epoch root, each binding's inclusion proof.

**4. Per-attestation companion.** Every agent attestation carries a **PQ companion signature** over the same digest the classical signature covers. Recompute: verify the companion against the bound (and anchored) PQ key. The PQ signature rides *alongside* the classical one — no protocol fork.

**5. Cutoff enforcement (load-bearing).** A cutoff enforcer with three modes:
- `shadow` — record the PQ presence/validity, do not gate (migration warm-up).
- `amber` — surface "could not verify PQ" as a first-class state, do not reject.
- `reject` — after the cutoff, a classical-only (no valid PQ companion) attestation is **rejected**. Proven live with a real ML-DSA reject.
The mode + cutoff are themselves anchored statements; the decision is recomputable (was there a valid, in-force, anchored PQ companion as of the attestation's time?). Ships **inert** (`mode = shadow`) — nothing changes until an owner rotates.

**6. Rotation.** The owner (SIWE) rotates the PQ key: a **second binding** supersedes the first under an **in-force rule** (the latest owner-authorized, anchored binding as of `as_of` governs). Recomputable and append-only — the prior binding is preserved, not erased.

**7. Revocation.** An **anchored statement class** + a **temporal rule**: a revocation is its own anchored record; an attestation signed after the revocation's in-force time by the revoked key does not verify. The breach is **preserved, never rewritten** (a late/overturned outcome keeps its record — same discipline as captured-admission).

## The one deliberate secret

Everything above is recomputable from public data **except** the **master seed** — the root from which per-agent keys derive. That is the *point*: it is the single standing secret, held **offline / encrypted-at-rest** (no plaintext env, no HSM required for the free tier), with a public **fingerprint** so its identity is checkable without exposing it. Back it up — it is the only copy.

## Recompute checklist (what anyone can re-derive)

- `binding_id` from the owner-authorized binding record.
- Epoch Merkle root + each binding's inclusion proof.
- Each attestation's PQ companion against the bound, anchored, in-force key.
- The cutoff decision: valid PQ companion present ∧ in-force ∧ anchored, as of the attestation's time.
- Rotation: the in-force binding as of any `as_of`.
- Revocation: whether a signature's time postdates an in-force revocation.

## Relation to the stack

- **ERC-8004** — the agent identity a PQ key binds to.
- **KYA-L4 attestor signature** — the dual-family (ML-DSA + SLH-DSA) production lane.
- **OCP / anchoring** — epoch roots + statement classes anchored on-chain.
- **captured-admission** — the append-only / preserve-the-breach discipline the rotation & revocation rules inherit.
- **`pq-key-binding-v0`** — the recompute-verified shared spec this formalizes.

## Reference implementation

Live in the agent gateway (not vendored here): `pqAgent.ts` (keygen + owner-authorized binding), `pqEpoch.ts` (batch-Merkle epoch anchor), `pqCutoff.ts` (the enforcer), and self-test endpoints `/pq/agent/:r/:id/{enforce,rotation}/selftest`. This repo is the **process + design spec**; the reference impl stays in the live system until a sanitized extraction is ready.

## Acknowledgements

- The per-agent binding lifecycle (keygen · owner-authorized bind · epoch anchor · per-attestation companion · load-bearing cutoff · rotation · revocation) is **Vértice / trustless-ai** design + reference implementation.
- **Fede (`babyblueviper`)** independently **recompute-verified** the binding — cold, from public data — and the shared **`pq-key-binding-v0`** spec this formalizes landed with him. The recompute-verification is what lets this spec claim "checkable, not asserted" honestly: a second party re-derived it with no shared state.
- Dual-family and cutoff/rotation/revocation semantics build on the working group's OCP / captured-admission / KYA-L4 lines.

*This is a working-group artifact; contributions credited by the exact thing each party did.*

---
*CC0. Extracted 2026-08-04 from Vértice / trustless-ai post-quantum work. To be transferred to `trustless-ai` when the reference extraction lands.*
