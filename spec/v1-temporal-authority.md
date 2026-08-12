# pq_key_binding — temporal authority (proposed v1)

**Status:** proposed. Changes conformance, so it is a version bump, not an erratum.
**Supersedes:** the temporal rule of `pq_key_binding.v0` only. Everything else in
v0 — statement shape, canonicalisation, content addressing, recovery classes,
terminality — is unchanged.
**v0 is not withdrawn.** It remains the accurate description of what v0
implementations do, including the defect below.

**Raised by** [@pipavlo82](https://github.com/pipavlo82) from a cold read of the
pinned v0, during an independent verification round by
[@babyblueviper1](https://github.com/babyblueviper1).

---

## 1. The defect

v0 says an epoch-0 binding governs `[T0, Trot)`. **It never defines `T0`.**

Implementations supplied it from local state. That makes the temporal boundary
*verdict-bearing* — it decides which key governs an artifact — while not being
*commitment-bearing*: it appears nowhere in the JCS statement, so it never reaches
the content address, the leaf, or any anchored root.

The consequence is the part that matters:

> The same committed statement, the same leaf, and the same anchored root can
> produce a **different temporal verdict** if a local value changes.

An implementation can therefore alter which key governs an artifact, and **every
recompute check the profile recommends will still pass**. Root recomputation
cannot detect it, because nothing about the roots changed.

### 1.1 Demonstrated, not hypothesised

Against a live deployment, two bindings for one agent, altering only the local
epoch-0 boundary:

```
boundary = 0             -> verdict at 2026-07-25: epoch 0
boundary = 1785436450    -> verdict at 2026-07-25: NO KEY IN FORCE
content addresses in both cases: 856aeceffd…, 81aec99364…   (identical)
```

### 1.2 The roles had already diverged in production

In the same deployment the field carried the time a rotation was *submitted*, not
the time it became *anchored*:

```
key_epoch 0:  local value 0             first anchored in batch 1
key_epoch 1:  local value 1786548015    first anchored in batch 2 at 1786549450
                                        divergence: 1435 seconds
```

For 1435 seconds the successor key was verdict-bearing while no anchor existed
that could prove it. It governed artifacts that no third party could yet verify it
governed. The two roles below are not a theoretical refinement; they had already
come apart in deployed data.

## 2. Two roles, separated

| role | meaning | normative? |
|---|---|---|
| `submitted_at` | when an implementation accepted the binding | **No.** Local, informational, MUST NOT affect any verdict. |
| `governs_from` | the temporal authority boundary | **Yes.** Derived, never stored. |

`governs_from` is **derived, not asserted**. A stored `governs_from` would remain
mutable, remain verdict-bearing, and remain invisible to root recomputation — a
better convention, not a bound fact.

It is also **not** added to the statement. Doing so would change the JCS bytes and
therefore every existing epoch-0 content address. More importantly, a committed
`governs_from` would be a self-asserted timestamp; an anchor is externally
witnessed. **Deriving is strictly stronger than committing here.**

## 3. Normative rules

For a binding `B` of agent `A` at `key_epoch n`, with leaf
`cc = sha256(JCS(statement))`:

**R1 — baseline.** For `n = 0`, `governs_from(B) = 0`.

> The baseline key is the agent's key from creation. There is no honest moment
> before which it did not govern. Anchoring gives a binding a provable time; it
> does not give it a birthday.

**R2 — successors.** For `n > 0`, `governs_from(B)` is the anchor timestamp of the
**earliest** anchored batch whose root contains `cc`.

**R3 — no containing anchor, split into two states.** If `n > 0` and no anchored
batch contains `cc`, `B` has no `governs_from` and does not govern. But *why* it
has none matters, and the first draft of this rule collapsed two different
situations into one verdict:

- **`PENDING`** — no batch has yet included it. It may still resolve. Derived: no
  binding of the same agent at `key_epoch > n` has a containing anchor.
- **`UNRESOLVABLE`** — it was **skipped** and never can resolve. Derived: some
  binding of the same agent at `key_epoch > n` **does** have a containing anchor,
  so a batch ran after `B` existed and did not include it.

Both refuse to govern, so neither admits an artifact. They are distinguished
because they mean different things to a reader: one is "wait", the other is
"this can never be proven, and anything signed under it is permanently
unverifiable". Reporting a permanent loss as a pending state is the same defect
this document exists to remove, one layer up.

The distinction is derived from anchors and epoch ordering alone. It deliberately
does **not** consult `submitted_at`, which would violate R5.

**R6 — batch completeness (the actual fix).** An anchored batch MUST contain a leaf
for **every binding not already contained in some earlier anchor**, not merely each
agent's current binding.

> Found by [@babyblueviper1](https://github.com/babyblueviper1), who checked the
> live batches instead of assuming: the leaf set is one leaf per
> `(registry, agent_id)` — 29 leaves, 29 distinct pairs, no agent twice — i.e. the
> *current* key at anchor time, not a rotation history.
>
> So an agent that rotates **twice between batches** — `N` submitted, then `N+1`
> submitted before the next batch fires — leaves `N` in no anchor, ever. Any
> companion genuinely signed under `N` during its brief life has no provable
> governance window, structurally and permanently. And fast double rotation is not
> exotic: it is what you do when a replacement key is itself suspect.

R6 removes the failure mode rather than defining a state for it. Cost is a few
extra leaves in one Merkle root and the same single transaction.

`UNRESOLVABLE` under R3 therefore remains reachable only for **legacy data
anchored before R6** and for **non-conforming implementations**. A defined state
for a loss that should no longer occur is a diagnosis, not a design.

**Rejected alternative:** forbidding a rotation while an earlier one is unanchored
(queueing it until the pending batch fires). That couples an owner's ability to
recover to the operator's batching cadence, and blocks the emergency case —
replacing a key twice quickly — at exactly the moment it is needed. Recovery must
not wait on a schedule.

**R4 — resolution is otherwise unchanged from v0.** Among bindings with
`governs_from ≤ anchor_time(artifact)` and not revoked at or before it, the one
with the greatest `governs_from` governs. No eligible binding → no key in force →
the artifact is rejected.

**R5 — independence.** A conforming verifier MUST compute R1–R4 from published
leaves and anchor timestamps alone. No implementation-held value may appear in the
verdict path. (R6 binds the *producer* of a batch, not the verifier — a verifier
cannot enforce it, only observe that a skipped epoch is UNRESOLVABLE under R3.)

## 4. What changes in practice

Only the window between a rotation being accepted and being anchored. Under v0
that window resolved to the successor; under v1 it resolves to the predecessor.

That is the intended correction, not a side effect: **during that window the
successor is not provable, so the key that is provable governs.** It also removes
any incentive to backdate, since the boundary is no longer something an
implementation states.

Artifacts outside that window are unaffected. In the deployment measured above,
the entire divergence is 1435 seconds on one agent, and no artifact falls inside
it.

## 5. Conformance vectors required

To be added to `recompute-kit/conformance/pq-key-binding-v1/`:

| vector | asserts |
|---|---|
| `baseline-governs-from-zero` | R1 — an artifact predating every anchor still resolves to epoch 0 |
| `successor-governs-from-anchor` | R2 — boundary is the earliest containing anchor, not submission |
| `pre-anchor-window-resolves-to-predecessor` | §4 — the v0/v1 divergence, explicit |
| `unanchored-successor-never-in-force` | R3 — fails closed |
| `submitted-at-is-not-verdict-bearing` | R5 — mutating `submitted_at` changes no verdict |
| `earliest-not-latest-containing-anchor` | R2 — re-anchoring the same leaf later does not move the boundary |
| `skipped-epoch-is-unresolvable` | R3 — an epoch superseded before any batch is UNRESOLVABLE, not PENDING |
| `unanchored-latest-epoch-is-pending` | R3 — the newest epoch with no batch yet is PENDING, distinct from the above |
| `batch-includes-every-unanchored-binding` | R6 — a conforming batch contains skipped epochs, so the state cannot arise |

The last two are the ones that would have caught the defect. A vector suite that
only checks agreement between implementations cannot find a value that every
implementation reads the same wrong way; `submitted-at-is-not-verdict-bearing`
tests the *shape* of the dependency rather than its value.

## 6. Migration

1. Implementations SHOULD rename the stored field to `submitted_at` so nothing
   continues reading a submission time as an anchor fact.
2. Implementations MUST stop using any stored value in resolution and derive
   `governs_from` per R1–R3.
3. v0 verdicts and v1 verdicts differ only within §4's window. Implementations
   SHOULD report which version produced a verdict.

## 7. Open questions

**Answered.** R2 uses the **earliest** containing anchor: a leaf re-anchored in a
later batch must not move the boundary forward. Routine, since batches re-include
current bindings, and ambiguous the moment a leaf appears in several.

**Answered by R6.** The skipped-epoch case — an epoch superseded before any batch
ran — was raised by @babyblueviper1 against the first draft, where R3 silently
covered both "not yet anchored" and "can never be anchored". R6 makes it
structurally impossible; R3 now names the two states apart for data anchored
before R6.

**Still open.** R6 changes what a batch contains, so a verifier recomputing an
OLD root must use the pre-R6 construction (one leaf per agent) and a new root the
post-R6 one. Roots are not self-describing about which rule built them. Options:
carry a construction version alongside the root, or accept that pre-R6 roots are
only recomputable against the pre-R6 rule and say so normatively. Leaning to the
former — a root whose construction must be inferred is a root that will eventually
be recomputed wrong.

---

## 8. UNRESOLVED — three ways this is still implementation-state-dependent

Raised by @pipavlo82 after R6, and they invalidate parts of the rules above rather
than merely extending them. **R1–R3 should not be treated as settled.**

**8.1 `key_epoch` is itself a trusted input.** R1/R2/R3 branch on `n`, but the
statement commits to `schema, subject, pq_pubkey, algorithm, profile,
secp256k1_pubkey` — **no `key_epoch`**. A verifier cannot tell a baseline from a
successor. The original failure shape survives one level lower: the same leaf can
be classified either way and receive a different `governs_from`.

Worse, the evidence that would order the chain already exists and is destroyed.
`rotationMessage` is owner-signed over `registry, agent_id, next_epoch,
next_pubkey, issued_at`; `submitRotation` verifies it and inserts a row with no
signature column.

**8.2 R6 has no enumerable domain.** A root proves membership of leaves it
contains; it cannot prove that an omitted binding existed, if that binding is
visible only in implementation state. As written R6 is a producer requirement, not
a third-party-recomputable completeness property.

**8.3 `construction_version` must be bound, not carried.** Metadata beside a root
leaves the same root interpretable under either construction.

### Direction under discussion

@babyblueviper1's `enumerate-verify` shape from `trustless-agent-substrate` #1,
transplanted: a committed gateway-wide monotonic `submission_seq` per binding, with
each batch committing `{min_seq, max_seq, count}`. Contiguity of the union is then
checkable, and a gap is a *provable* omission.

It also answers 8.1, which is the part worth noting: with a committed sequence,
**`key_epoch` is derived from position** — the k-th binding of an agent in sequence
order is its epoch k. Ordering stops being a number to trust. That is stronger than
binding `key_epoch` into the statement, which only makes the number tamper-evident.

One gap in that scheme as stated: a withheld *manifest* is indistinguishable from a
gap, so enumerating bindings requires first enumerating batches. Proposed
resolution — anchor a manifest commitment rather than a bare root:

```
manifest = { construction_version, seq_range{min,max}, count, leaves_root, prev_manifest_cc }
anchor   = record( sha256(JCS(manifest)) )
```

which binds the construction (8.3), the domain (8.2), and chains batches so none can
be hidden. Pre-manifest anchors need an independently anchored activation boundary.

`recompute-kit` #8 is **on hold** until this settles: its vectors model the R1–R5
shape and exercise neither R6 nor the PENDING/UNRESOLVABLE split.
