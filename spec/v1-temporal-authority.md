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

**R3 — fail closed.** If `n > 0` and no anchored batch contains `cc`, `B` has no
`governs_from` and is **never in force**. A binding that cannot be proven cannot
govern.

**R4 — resolution is otherwise unchanged from v0.** Among bindings with
`governs_from ≤ anchor_time(artifact)` and not revoked at or before it, the one
with the greatest `governs_from` governs. No eligible binding → no key in force →
the artifact is rejected.

**R5 — independence.** A conforming verifier MUST compute R1–R4 from published
leaves and anchor timestamps alone. No implementation-held value may appear in the
verdict path.

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

## 7. Open question

R2 uses the **earliest** containing anchor. If a leaf is re-anchored in a later
batch — routine, since batches include every current binding — the boundary must
not move forward. Stated explicitly because "the anchor containing it" is ambiguous
once a leaf appears in several, and the ambiguity is exactly the kind that reads as
settled until two implementations disagree.
