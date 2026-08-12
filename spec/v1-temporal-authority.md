# pq_key_binding — temporal authority (proposed v1)

**Status:** **DESIGN FROZEN 2026-08-12**, pending adversarial vectors. Extending the
design further is closed; the sequence from here is rewrite → attack with vectors →
implement against the frozen shape. Changes conformance, so it is a version bump, not
an erratum.
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

---

## 9. Agreed construction — admission-relative completeness

Converged with @pipavlo82 and @babyblueviper1. Replaces the direction sketched in
§8; R1–R3 above are superseded by ordering derived from the acceptance sequence.

```
acceptance record   { submission_seq, subject, binding_cc, predecessor }
                    append-only; head anchored INDEPENDENTLY of, and below, the batch

batch manifest      { construction_version, admission_head, covered_through_seq,
                      min_seq, max_seq, count, bindings_root, previous_manifest }
anchor              record( sha256(JCS(manifest)) )
```

Three issues collapse into one construction:

1. **ordering** — per-agent predecessor/successor derived from the committed
   acceptance sequence. `key_epoch` stops being an input; it is position.
2. **coverage** — a batch is checked against the anchored admission head, so an
   omitted admitted binding is a provable sequence gap.
3. **construction** — the rule that built a root is commitment-bearing, not metadata
   carried beside it.

`predecessor` is kept even though position implies ordering: position gives the
sequence, `predecessor` gives the intent, and a disagreement between them is itself
evidence. It fails loudly rather than silently.

### 9.1 What this proves, and what it does not — normative

The property is **admission-relative completeness**. Stated in full, because the
adjacent claim is the one that would be wrong:

| claim | provable to | mechanism |
|---|---|---|
| no **admitted** binding was omitted from a batch | any third party | seq contiguity of `covered_through_seq` against the anchored admission head |
| a given binding **was** admitted | the owner | acceptance record at its `submission_seq` under the anchored head |
| every **request** was admitted | **nobody** | — |

An implementation MUST NOT describe this as completeness over eligible requests. A
sequence records what was admitted, not what was asked; a producer that never issues
a `submission_seq` leaves no gap. That is a censorship property, and no internal
sequencing converts one into the other.

### 9.2 Acceptance acknowledgement (narrows, does not close)

Acceptance SHOULD return `{submission_seq, subject, binding_cc, predecessor}` signed
by the producer key.

An owner holding an ack whose `submission_seq` is absent from a later head covering
that range then has a **third-party-provable omission** — the producer attested to
admitting it. An owner holding no ack knows at request time that they were refused.

This makes refusal loud and contemporaneous rather than silent and retrospective. It
does not make refusal impossible, and MUST NOT be described as though it does.

### 9.3 Open

Anchor the admission head **per batch** (cheap, coarse — a gap surfaces only when
the next batch lands) or on its **own cadence** (bounded detection time, one extra
transaction)? The second makes "admitted but not yet covered" observable without
depending on when someone chooses to cut a batch.

---

## 10. Settled construction

Converged with @pipavlo82 and @babyblueviper1. **R1–R3 are superseded, not amended**
— this is a different construction, not a corrected one.

```
acceptance log   append-only, hash-chained
                 { submission_seq, binding_cc, subject, authorization_cc, prev_acceptance_cc }

authorization    owner-signed:  predecessor_binding_cc -> successor_binding_cc
                 referenced by cc from the acceptance record, never embedded

manifest         { schema, profile, construction_version,
                   acceptance_head_cc, covered_through_seq,
                   min_seq, max_seq, count, entries_root, prev_manifest_cc }
entries_root     Merkle over H(JCS({submission_seq, binding_cc, subject}))
anchor           record( sha256(JCS(manifest)) )
```

**Entries bind the sequence.** With bare `binding_cc` leaves, `min/max/count`
describe a sequence the root does not commit to — the range would sit *beside* the
root exactly as `construction_version` did. Caught by @pipavlo82.

**No epoch number in the signed authorization.** Once position derives the epoch, a
number inside a signed message is a trap: it will be read as authoritative, and if it
ever disagrees with position there are two sources of truth and a signature endorsing
one of them. Content addresses name the objects and cannot disagree.

### 10.1 Layers

| layer | question it answers |
|---|---|
| position in the acceptance sequence | what order |
| owner signature | was it authorized |
| anchored acceptance head | which admissions exist (enumerable domain) |
| batch manifest | coverage and construction |
| on-chain anchor | external time |
| `legacy_bindings_root` | what predates all of it |

Each answers exactly one question, and none is a number a verifier is asked to
trust. Order and authorization are kept apart for the same reason the companion
endpoint reports `signature_valid` and `pubkey_bound` separately: a valid signature
under a key that was never bound is a forgery with a valid signature.

### 10.2 Activation — the genesis manifest IS the cutover

No separate activation record: the boundary and the first object governed by the new
construction are one immutable fact, rather than two whose ordering needs specifying.

```
construction_version      = 1
prev_manifest_cc          = null
legacy_predecessor_anchor = <last bare-root anchor>
legacy_bindings_root      = <Merkle over inherited {subject, binding_cc, key_epoch}>
```

**`legacy_bindings_root` is required**, because bindings that predate the acceptance
log have no position and are therefore unorderable under this construction. Measured
on a live deployment at the time of writing: 29 agents with an implicit epoch 0 plus
2 explicit rows, none with a `submission_seq`.

Backfilling acceptance records for them is **forbidden**: assigning a
`submission_seq` retroactively to a binding never admitted through the sequence
manufactures evidence that did not exist — the backdating this construction exists to
prevent, committed by the mechanism preventing it.

Freezing claims something weaker and true: *this is the state we inherited, committed
once, at a named moment*. Those epoch numbers are mutable local state today and become
tamper-evident from the genesis manifest onward.

A binding is governed by this construction if it has an acceptance record, and by the
frozen legacy set otherwise. Decidable from committed data, with no third state.

### 10.3 Admission-head cadence — a deadline inherited from a witnessed head

Per-batch-only couples observability to batching: an admitted binding can stay
globally invisible for an unbounded period if no batch is cut.

`observed_head_cc` — the head most recently anchored when the record was accepted —
goes in the acceptance record. Its job is narrow and it must not be widened:

> **It proves the admission happened AFTER that head. It is not an acceptance
> timestamp.** An externally witnessed causal lower bound, nothing more.

From that, the deadline is derived rather than asserted (@pipavlo82):

```
ack_deadline = anchor_time(observed_head_cc) + Δ
```

Fully recomputable by a third party. A producer may issue an ack late within the
interval, but then inherits a correspondingly short remaining window; if the latest
head is too stale for that to be practical, it anchors a fresh head first and acks
against that.

This is why the bound is phrased as **closure from the referenced anchored head**,
not "Δ from acceptance". A bound from acceptance needs an independently witnessed
acceptance time, which does not exist. It also avoids requiring empty heartbeat
transactions while idle: a fresh head is needed only before accepting against a
stale observation point, not on a timer.

```
acceptance_record = { schema, submission_seq, subject, binding_cc,
                      predecessor_binding_cc, authorization_cc,
                      observed_head_cc, prev_acceptance_cc }
acceptance_cc     = sha256(JCS(acceptance_record))
ack               = producer_signature(acceptance_cc)
```

The ack signs the **whole** record, so the holder has evidence of the exact object
that must appear under a later head, including its authorization link and chain
position.

### 10.3.1 The admission head needs indexed sequence semantics

A bag root proves membership; it cannot prove **what occupies position s**. Since
the failure states below turn on "the entry at `submission_seq = s` is not the
acknowledged `acceptance_cc`", the head MUST commit to an ordered construction —
Merkle over `H(JCS({submission_seq, acceptance_cc}))` — plus the committed
contiguous range. Without position, an omission and a reordering are
indistinguishable, and neither is provable.

### 10.4 State progression — semantic state, with liveness kept separate

| state | what is true | who can establish it |
|---|---|---|
| `REQUESTED` | **outside the normative machine.** Requester-local; no public admission fact exists | requester only |
| `ACKNOWLEDGED` | holder has `producer_signature(acceptance_cc)`. Cryptographically attributable, not yet under an external head. Closure deadline is mechanically `anchor_time(observed_head_cc) + Δ` | holder |
| `ADMISSION_ANCHORED` | the exact `acceptance_cc` appears at its committed `submission_seq` under an anchored head. Ordering is third-party derivable | anyone |
| `COVERED` | the binding entry appears in an anchored batch manifest; `governs_from` derivable | anyone |

Separately, and deliberately not mixed into the above — three positive-failure
conditions, distinguished by **what evidence establishes them**:

| condition | what it is | establishable by |
|---|---|---|
| `ACK_EQUIVOCATION` | two valid producer-signed acks make incompatible claims about the same `submission_seq` (different `acceptance_cc`). The producer contradicted itself **before any external closure** | anyone shown both acks — **no anchor required** |
| `ANCHOR_OVERDUE` | the ack deadline passed without a successor head closing the range. A timing fact; **the signed acceptance remains valid evidence** | holder; any third party shown the ack |
| `ADMISSION_CONFLICT` | a later anchored head closes position `s` to something incompatible with the acknowledged `acceptance_cc`. Absence has become a positive contradiction | anyone shown the ack, plus the chain |

`ACK_EQUIVOCATION` is the strongest of the three and needs the least: two signatures
and no chain at all. Raised by @pipavlo82.

**Reporting rule — within a class, not across classes.**

> Within the same semantic failure class, report the most specific provable
> contradiction. Orthogonal liveness facts remain independently reportable.

`ACK_EQUIVOCATION` and `ADMISSION_CONFLICT` are attributable semantic
contradictions; `ANCHOR_OVERDUE` is a liveness fact and is orthogonal to both. One
acknowledged admission may therefore correctly report **both** `ACK_EQUIVOCATION`
(semantic) and `ANCHOR_OVERDUE` (liveness) at once.

So: an equivocation MUST prevent that same semantic failure being softened to
omission because a head later chose a side — but it MUST NOT suppress an
independently true timing breach. An earlier draft of this rule totally ordered all
three, which would have silently hidden a real liveness failure behind a more severe
finding of a different kind. Caught by @pipavlo82.

### 10.4.1 The fork shape is constrained away, not named

Two signed acceptance records sharing a `prev_acceptance_cc` and both claiming to be
the next position is a sibling failure. Rather than give it a state:

> **`submission_seq` MUST equal `submission_seq(predecessor) + 1`, where the
> predecessor is FETCHED and RECOMPUTED, never resolved from implementation state.**

A verifier resolves `prev_acceptance_cc` to a published record, recomputes
`sha256(JCS(record))` and requires it to equal the referenced cc, reads the
`submission_seq` committed *inside that record*, and only then applies the
constraint.

If the predecessor cannot be fetched and recomputed, the chain relation is
**`UNRESOLVED`** — not satisfied, not violated, and above all not assumed. An
earlier draft wrote this as `seq(prev_acceptance_cc)`, a lookup that in any real
implementation resolves from local state: the precise defect this constraint exists
to remove, inside the constraint. Caught by @pipavlo82.

With the chain and the index required to agree by construction, a fork on
`prev_acceptance_cc` necessarily produces the same `submission_seq`, and therefore
collapses into `ACK_EQUIVOCATION`. A record violating the constraint is malformed
and rejectable on sight rather than being a fourth failure mode. Same instinct as
R6: remove the degree of freedom instead of naming what happens when it is abused.

### 10.4.2 Domain separation

The producer signature MUST NOT be over a naked `acceptance_cc`. Both halves:

```
acceptance_record = { schema: "kya.pq_acceptance.v1", profile, producer_key, chain_id,
                      submission_seq, subject, binding_cc, predecessor_binding_cc,
                      authorization_cc, observed_head_cc, prev_acceptance_cc }
acceptance_cc     = sha256(JCS(acceptance_record))
ack               = producer_signature( sha256(JCS({ type: "kya.acceptance_ack.v1",
                                                     acceptance_cc })) )
```

`profile`, `producer_key` and `chain_id` inside the record so **`acceptance_cc`
itself** cannot recur in another deployment — domain-separating only the signature
would still let the same content address appear in someone else's log. The typed
preimage on top so an ack signature cannot be replayed as a different object type
signed by the same producer key.

### 10.4.3 Required negative vectors

All three positive-failure conditions get explicit vectors, and the equivocation
case must assert the **stronger** label:

| vector | asserts |
|---|---|
| `ack-equivocation-two-signed-acks` | same `submission_seq`, different `acceptance_cc`, no anchor needed |
| `ack-equivocation-survives-head-choice` | after a head picks a side, the loser reports equivocation, NOT omission |
| `anchor-overdue-does-not-invalidate-ack` | deadline passed; the acceptance is still valid evidence |
| `admission-conflict-committed-position` | head closes `s` to a different `acceptance_cc` |
| `fork-on-prev-collapses-to-equivocation` | the seq constraint forces the two shapes together |
| `unresolvable-predecessor-is-not-satisfied` | chain relation `UNRESOLVED` when the predecessor cannot be recomputed |
| `equivocation-and-overdue-report-together` | orthogonal classes both reported; neither suppresses the other |
| `naked-cc-signature-rejected` | an ack over an untyped preimage does not verify |

An earlier draft of this table got that wrong — it named an `OMITTED` state on the
weaker condition, one section after §9.1 states that "we cannot see it" and "it
provably is not there" are different claims and the first must never be reported as
the second. Recorded rather than silently corrected, because writing the principle
down plainly did not prevent violating it in the next table.

---

## 11. This is a PROFILE, not a new mechanism

Diffed against the frozen `captured-admission.v0` core (verified on `main`:
**63/63**, checker `sha256 7a2369ae…`, matching the freeze at `a724afc`).

Every structural property §10.4 converged on already exists there, blind-diffed by
@pipavlo82 and @babyblueviper1:

| §10 property | frozen case → verdict |
|---|---|
| claim before the boundary has no authority | `authority_claim_before_activation_out_of_authority` → `out_of_authority` |
| rotation / revocation non-retroactive | `..._after_supersede_nonretroactive`, `NC2b_..._after_revoke_nonretroactive` → `attributed` |
| a successor must be bound | `AUTH_supersede_without_bound_successor_invalid` → `invalid_transition` |
| PENDING | `AUTH_as_of_before_activation_epoch_not_yet_active` → `epoch_not_yet_active` |
| not-yet-visible ≠ omitted | `AUTH_future_claim_not_visible_at_earlier_asof` → `claim_not_yet_visible` |
| `ADMISSION_CONFLICT` / `ACK_EQUIVOCATION` | `ENUM_conflicting_commitments_same_index` → `conflicting_index` |
| gaps / duplicates / ordering / head≠anchor | `ENUM_gap_missing_index`, `ENUM_duplicate_index_same_commitment`, `ENUM_out_of_order_entries`, `ENUM_commitment_mismatch_head_ne_anchor` |
| `ANCHOR_OVERDUE` is liveness, not invalidity | `NC1_obligation_overdue_is_liveness_not_rejected` → `unresolved\|liveness_failure` |
| breach survives late closure | `NC5_late_resolution_preserves_deadline_breach` → `late` |

**The core takes an activation boundary as INPUT and proves the state machine over
it. It never says where that boundary comes from — which is the question this
document was actually about.** The machinery was always correct; our input was
implementation state.

So `pq_key_binding.v1` is a **profile** over the frozen core, reusing
`admission_check.py` unmodified (the pattern @babyblueviper1 established in
recompute-kit PR #6). The profile owns only:

1. **derivation of activation** — `governs_from` from the earliest containing anchor;
   `0` for a baseline
2. **the owner-signed transition** — `predecessor_binding_cc → successor_binding_cc`,
   domain-separated, persisted
3. **`seed_epoch`** — the master-seed axis the core has no notion of, because all
   agent keys derive from one seed
4. **`legacy_bindings_root`** — freezing bindings that predate any sequence

**Limit of this claim:** the table above is a semantic mapping by case name and
verdict, not a mechanical proof. The proof is to express these vectors in
`captured-admission.v0`'s shape and run them through the unmodified frozen checker.
Until that is green, "this is a profile" is a proposal, not a result.
