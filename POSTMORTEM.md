# Post-mortem: characterizing the wall between a fully-specified task and a frontier coding model

## The law, and why it is the result

A Terminal-Bench task is *fair* when everything the agent needs is either stated
in the instruction or present in the provided code — no withheld fact, no
answer-key in the container. The campaign set out to build a fair task that a
frontier coding model (Claude Opus 5 and gpt-5.6-sol) nonetheless **fails**. Nine designs, plus three further approaches evaluated and
refuted before build, tested against both models, yield not a single task but a
characterization of where such a task can and cannot exist:

> **A self-verifying frontier model solves any difficulty that is stated.** Given
> a complete specification it reads the rules, implements them, and — crucially —
> writes its own verification to check its work, however entangled the rules or
> large the codebase. The only difficulty that survives is **unstated
> irreducibility**: a correctness condition that is *hard to test even when known*
> (a concurrency window a self-test cannot force) or *hard to locate even when
> the principle is known* (an invariant discoverable only by reading irregular,
> hand-authored code).

Every design below is a controlled experiment against that law: a hypothesis
about which seam might hold, a method that isolates it, a result, and a diagnosed
cause. Read together they do not describe a search for a task that worked — they
describe the *measurement* of a boundary, and the boundary is sharp.

---

## Method: the blind kill-test

The campaign's core instrument is a **blind run**. Before committing to build a
task, the candidate's agent-visible material (spec + inputs only — no reference,
no harness, no author notes) is handed to a fresh frontier-model session with no
knowledge of the campaign. It implements the task at full effort, writing its own
tests. Its output is then differenced against the author's reference over an
adversarial corpus. This replaces the usual "imagine a careful agent" thought
experiment with the actual adversary: a strawman always loses to its author;
only the real model predicts the real model. Every go/no-go decision below rests
on a blind run, not an intuition.

---

## The experiments

### Class 1 — the hidden-fact traps
*Hypothesis: hide the discriminating fact; the agent cannot act on what it cannot
see.* Method: several designs (data-absence/survivorship among them) placed the
key fact off the stated surface. Result: **rejected before build.** To grade such
a task fairly, the author must *state the grading rule*, and a faithful grading
rule re-exposes the hidden fact — or the fact is recoverable by a check the agent
naturally runs. The survivorship design is the clean instance: the "dropped
entities" were reconstructable from a referential-integrity check any careful
agent performs before a join, and the naive statistic (a 0% default rate) was
self-evidently wrong. Discoverability and self-verifiability turned out to be the
same property. Cause: **a stated grading rule cannot both be honest and conceal
its own discriminator.**

### Class 2 — the exact-answer audits
*Hypothesis: make the answer a large, exact, unguessable object; brute force and
lookup both fail.* Method: `protocol-audit` asks the model to classify fifteen
validation-split specifications (sound vs. contaminated, by mechanism, with exact
evidence sets) against a byte-frozen key. Result: **solved** — the model reads
each spec, executes the split against the panel, and derives the classification
in about fifteen minutes. The exact-match key raises the *cost of being wrong*,
not the *difficulty of being right*: correctness is a deterministic function of
(panel, spec), and the model computes it. Cause: **exactness constrains the
output form, not the reasoning.**

### Class 3 — the compressible-spec conformance task
*Hypothesis: a specification with ~fifteen interacting rules is too entangled to
implement correctly without a reference.* Method: `filing-feed-normalizer` — a
flat-affect spec (every rule stated once, neutrally, no emphasis on the tricky
interactions) plus a blind run differenced over a hand-authored adversarial
corpus that stacked framing × escaping × fixed-width × four-way cell semantics ×
typed coercion × amendment overlay. Result: **zero divergences.** The blind model
reproduced the reference exactly on the first attempt, and its own report
narrated every cross-term correctly. Cause: **a complete spec is decompressible;
entanglement is volume, not irreducibility.** The verifier's own teeth were
confirmed by an injected-divergence control, so the zero is real.

### Class 4 — the meta-condition / emergent-invariant tasks
*Hypothesis: state only the correctness meta-rule ("recovered state must equal a
crash-free run"), never the bug; the rule localizes nothing, and reproducing the
failure requires infrastructure the model won't build in-budget.* Method:
`crash-recovery-pipeline` — a functional-test-passing bitemporal pipeline with a
delivery-id dedup defect that violates equivalence only under at-least-once
redelivery-under-new-ids. Result: **solved by principle in ~5 minutes.** The
blind model read the honest at-least-once semantics, recognized delivery-scoped
dedup as unsound, and applied a full-ledger rebuild in logical order — dissolving
the entire defect class without ever reproducing the fault schedule. Cause: **the
load-bearing semantics, stated honestly, are exactly what a frontier model needs
to fix by principle.** The meta-condition felt non-localizing, but combined with
the stated semantics it localized the fix.

### The seam that held — unstated concurrency, and why it is fragile
*Hypothesis: a false-green concurrency window — where the agent's own self-test
structurally cannot reproduce the failing interleaving — resists the self-verify
loop.* Method: `ledger-repair`, a concurrent bitemporal ledger whose
durable-prefix invariant is violated only under out-of-order commit. A dedicated
kill-test measured two properties: an in-order self-test (the kind an agent
naturally writes) never constructs the pre-publish window (false green, 0/20
detection), while the out-of-order harness detects it deterministically (20/20);
and a coupled defect pair produces genuine masking (only the joint fix clears the
suite; each single fix flips which trace fails). Result: the kill-test **passed** —
this is the one property that resisted. But the discipline that makes the task
*fair* (state every invariant precisely) is in direct tension with the property
that makes it *hard* (leave the invariant unstated). When the invariant was
stated to remove ambiguity, the model solved it in six minutes. Cause: **the
resisting property is real but fragile — it lives precisely in what a fair task
must not state.**

---

## Verification discipline: catching our own ambiguities

Five times during development, a model scored zero on a task it had solved
before. Under a weaker process each would have been logged as a model failure.
The campaign's verification discipline instead diagnosed each one, and **all five
were our own specification ambiguities, not model limitations**:

- a dedup step's row-set convention (are removed rows absent from the output frame?),
- an `in [a, b]` reading (inclusive range vs. two-element set),
- an unmapped-key fallback (what unit does a company absent from the map take?),
- a `by_value` classification label (a split with no group key is a grouping-stage split),
- a helper-module import path (a package-organized submission the loader must import).

In every case the model had produced a **correct, internally-consistent artifact**
that eliminated the contamination and survived the perturbation gate; the
exact-match verifier rejected it only for choosing a different *valid* form. Each
was resolved by making the specification explicit and re-freezing — the answer
key never changed, because the fix clarified prose to match the existing answer,
not the answer to match a model. That a model fails *only* where the prose admits
two readings, and succeeds everywhere it is precise, is the sharpest confirmation
of the law: the boundary between solved and unsolved tracks the boundary between
specified and ambiguous, exactly.

The instruments themselves were held to the same standard: byte-identical answer
keys across independent environments (a determinism audit on every task), a
subprocess isolation layer that executes untrusted agent code as an unprivileged
user with no path to the reward channel, a perturbation gate that fails any
hardcoded output, and a scripted `/cheat` adversary per task confirmed to score
zero. A finding is only as trustworthy as the harness that produced it.

---

## The anchor: where the wall genuinely holds

The Terminal-Bench tasks that *do* defeat frontier models (the census hard-class —
`wal-recovery-ordering`, `mvcc-lsm-compaction`) were studied directly. Their
codebases are small (≈700 and ≈550 lines); their difficulty is not size but
**unstated irreducibility embedded in real, irregular code**: an omission that
compiles clean and sits two files from its symptom, fenced on both sides so
neither over- nor under-correction passes; or ~nineteen simultaneous invariants
coupled so that fixing one masks another, discoverable only by reading the whole.
That irregularity is authored, not generated — programmatic generation yields
regular, decompressible code that a model reads as fast as it reads a spec. This
is why the wall holds there and not in a synthesized task, however entangled.

---

## Two honest directions

The law leaves exactly one seam, and the campaign's own artifacts point at it:

1. **Real-system anchoring.** Difficulty that cannot be written down lives in
   genuine systems whose invariants are found only by reading them. A task
   anchored to a real codebase — not a synthesized one — is the honest path to
   the census hard-class, because the irreducibility is inherited rather than
   manufactured.

2. **False-green concurrency at scale.** `ledger-repair` demonstrates the one
   property that resisted the self-verify loop: a correctness condition the
   agent's own testing cannot force. Scaled from a single ledger to a
   multi-writer system, this is the strongest available lever — bought with
   hand-authoring cost, which is precisely the cost the finding predicts is
   irreducible.

---

## What the campaign delivered

Five original Terminal-Bench 3 tasks — uncheatable, deterministic, and fair —
that survive the discipline the finding demands, with `protocol-repair` and
`ledger-repair` as flagships in the census hard-class shape (a built/repaired
artifact, a large verifier surface, a second orthogonal gate). Both frontier
models solve them cleanly, and that is the finding confirmed rather than a
shortfall: the tasks mark exactly how close a fully-specified problem can come to
the wall, and the characterization above marks why it cannot cross without
leaving the specified regime. Measuring a boundary precisely is a result; these
tasks are where the measurement was taken.
