# When a fully-specified task can't beat a frontier model — a Terminal-Bench 3 study

## The finding

This repository asks one question: can a **fully-specified, honestly-graded**
data-engineering task defeat a frontier coding model? Across a multi-design
campaign, tested against **both Claude Opus 5 and gpt-5.6-sol**, the answer is a
characterized **no** — and the characterization is the contribution:

> **Stated difficulty is solved by a self-verifying frontier model.** Given a
> complete spec, the model reads it, implements it, and writes its own
> verification to check itself — however entangled the rules, however large the
> codebase. Difficulty survives only in **unstated irreducibility**: an invariant
> discoverable *solely* from irregular hand-authored code (as in the census
> winners `wal-recovery` / `mvcc-lsm-compaction`), or a **false-green
> concurrency window** that a self-test structurally cannot force.

The sharpest evidence for the law came from its apparent counterexamples. During
development, five separate trials looked like genuine model failures (Opus and
sol each scored 0 on a task they'd solved before). **All five traced to our own
spec ambiguity, not model limitation** — a dedup row-set convention, an
inclusive-range reading, an unmapped-key fallback, a `by_value` classification
label, a helper-module import path. Each time, the model had produced a correct
and internally-consistent artifact that our exact-match verifier rejected for
choosing a different *valid* form. Fixing the spec (never the key) made the
"failure" vanish. A model that fails only where our prose is ambiguous, and
succeeds everywhere it is precise, *is* the law stated as data.

## The validated artifacts

The campaign's product is five original TB3 tasks — **uncheatable, deterministic,
TB3-grade** — that survive the discipline the finding demands: honest grading,
no hidden facts, byte-stable answer keys, and a `/cheat` that scores 0. Both
frontier models solve them cleanly; that is the finding **confirmed**, not a
shortfall. The two flagships match the census hard-class shape — a
built/repaired artifact, a large verifier surface, and a second orthogonal gate —
and are the closest a fully-specified task gets to the wall.

| task | domain | shape |
|---|---|---|
| **protocol-repair** ⭐ | Operations/Finance | audit **and** ship data-deriving repair modules; 114 instances across 3 tiers + a perturbation anti-cheat gate |
| **ledger-repair** ⭐ | Operations/Finance | concurrent bitemporal ledger crash-recovery; 43-trace differential battery with genuine masking + a false-green race harness |
| protocol-audit | Operations/Finance | classify 15 validation-split specs (sound/contaminated + mechanism + evidence) |
| bitemporal-asof | Operations/Finance | as-of resolution over valid × knowledge axes (Q2 semantics) |
| margin-interval-cal | Operations/Finance | prediction intervals under a two-gate (coverage + width) hidden-holdout verifier |

## Results (post spec-hardening; harbor jobs on this machine)

| task | oracle | nop | cheat | Opus 5 | gpt-5.6-sol |
|---|---|---|---|---|---|
| protocol-repair | 1.0 | 0.0 (13-21-45) | 0.0 | **3/3** (13-28-11, 13-37-56, 13-49-53) | **3/3** (13-28-25, 13-35-39, 13-42-18) |
| ledger-repair | 1.0 | 0.0 | 0.0 | **3/3** (18-34-44, 01-32-32, 01-45-47)¹ | **3/3** (13-13-39, 13-17-01, 13-19-21) |
| protocol-audit | 1.0 (18-52-07) | 0.0 (23 19-09-47) | 0.0 | 1.0 (19-09-30) | 1.0 (19-09-19) |
| bitemporal-asof | 1.0 (18-52-39) | 0.0 (24 01-34-26) | 0.0 | 1.0 (19-16-54) | 1.0 (19-12-46) |
| margin-interval-cal | 1.0 (18-53-10) | 0.0 (23 14-47-40) | 0.0 | 1.0 (19-19-41) | 1.0 (19-14-06) |

¹ Opus cleared ledger-repair on four runs (adding 18-43-16); three are cited.

Every 0 in the cheat column is a scripted adversary: for protocol-repair a
correct audit plus **hardcoded** repair dumps (caught by the perturbation gate);
for ledger-repair a **select-only fix** that leaves the prefix bug intact
(caught by the out-of-order traces). Neither can reach a passing score without
the actual correct work.

Full reward index for all cited jobs: [results/rewards.md](results/rewards.md).

## Reproduction

These tasks were built and evaluated against `terminal-bench` at commit
`9ab711d42442170cf6ad28b02d63da717940854a`; to reproduce, place the task
folder(s) into that version and run the documented commands below.

    harbor run -p tasks/<name> --agent oracle --env docker   # expect 1.0
    harbor run -p tasks/<name> --agent nop    --env docker   # expect 0.0

    # Opus 5
    --agent claude-code -m anthropic/claude-opus-5   # reasoning_effort=max, CLAUDE_CODE_MAX_OUTPUT_TOKENS=128000
    # gpt-5.6-sol
    --agent codex -m openai/gpt-5.6-sol              # reasoning_effort=xhigh, --ak version=0.149.1

## Infrastructure appendix

- **`CLAUDE_CODE_MAX_OUTPUT_TOKENS=128000`** is required for claude-code; the
  default budget truncates long repair batches.
- **codex `@latest` routing bug / version pin.** The harbor codex adapter
  installs `@openai/codex@latest` unpinned; a newer CLI routed `gpt-5.6-terra`
  to the platform endpoint with a placeholder key and returned 401. Fix: pass
  the agent `version` kwarg → `npm install -g @openai/codex@0.149.1` (the host's
  proven `codex-cli 0.149.1`). The adapter reads `OPENAI_API_KEY`, not
  `CODEX_API_KEY`; `CODEX_FORCE_AUTH_JSON=1` mounts `~/.codex/auth.json`.
- **`gpt-5.6-sol` entitlement finding.** Under a ChatGPT-account Codex login,
  `gpt-5.6-sol` returns verbatim:
  `{"status":400,"error":{"type":"invalid_request_error","message":"The 'gpt-5.6-sol' model is not supported when using Codex with a ChatGPT account."}}`
  Workaround: authenticate with an `OPENAI_API_KEY` whose org is entitled to the
  model (unset `CODEX_FORCE_AUTH_JSON` so codex uses API-key auth), then confirm
  with `codex exec --model gpt-5.6-sol -- 'hi'` before running the battery.

## What would beat the law

The law leaves one seam open, and it is where this campaign's own artifacts point:

- **Real-system anchoring.** The census winners embed correctness in a real,
  irregular codebase whose invariants are found only by reading it — not
  reproducible by generation, which yields regular, decompressible code. A task
  anchored to a genuine system (not a synthesized one) is the honest path.
- **False-green concurrency at scale.** `ledger-repair` demonstrates the one
  property that resisted — a self-test cannot force the out-of-order window;
  scaled to a multi-writer system it is the strongest lever, at hand-authoring
  cost.

## Post-mortem

The full write-up — the stated-vs-unstated law, the five spec strikes as its
confirmation, and why programmatic generation cannot produce the census-winner
property — is in [./POSTMORTEM.md](./POSTMORTEM.md).
