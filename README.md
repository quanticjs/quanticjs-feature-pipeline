# quanticjs-feature-pipeline

A **reusable, domain-agnostic pipeline** that turns an uploaded BRD into working code on any QuanticJS
product repo. The engine lives here once; product repos reference it via a one-file GitHub Actions
caller. **Upload a BRD → the whole chain runs end-to-end.**

```
docs/brd/** uploaded ─► BRD→HLD ─► HLD→FSD index ─► FSD authoring ─► one "HLD + FSD" PR
   merge that PR (docs/fsd/**) ─► FSD→SPEC ─► specs PR
      merge specs PR ─► (manual) /implement-spec per spec, dependency-ordered ─► code
```

Stage 0 (BRD→HLD) is **grounded in the actual QuanticJS framework repos** (`quanticjs-backend`,
`quanticjs-ui`, `quanticflow`, `notification-engine`, file service) so the HLD reflects real platform
capabilities, not guesses.

## Repo layout

```
.github/workflows/feature-pipeline.driver.yml     on: workflow_call — SINGLE-INVOCATION driver (best for testing)
.github/workflows/feature-pipeline.reusable.yml   on: workflow_call — multi-job matrix engine (scaled)
.github/actions/run-claude/                        headless runner (claude-code-cli implemented; API key or OAuth)
.github/actions/verify-fsd/                         structural gate (required /implement-spec sections)
prompts/   driver · 00-brd-to-hld · 10-index-and-template · 11-fsd-author · 20-fsd-to-spec   (all domain-agnostic)
templates/_TEMPLATE.fsd.md                          canonical FSD template seed
config/pipeline.defaults.yml                        defaults (decomposition, gates, framework, spec numbering/tracker, runner, paths)
examples/
  consumer-caller.example.yml                       the thin caller a product repo drops in
  trade-finance/                                     first real consumer (config + lossless reference run)
```

## Outputs: numbered specs + a tracker

Stage 1 assigns every spec a 3-digit **build-order number** (`spec.numbering: build-order`) and seeds
**`docs/specs/TRACKER.md`** (`spec.tracker: true`) — the status + dependency index. Stage 3 writes
`SPEC-<number>-<slug>.md` and fills the tracker rows (FSD, depends-on, sub-skills, Status ☐). Implement
in ascending number; flip Status as PRs merge. This mirrors how the Trade-Finance consumer's 31
foundation specs are tracked.

## Test it (driver path)

The `driver.yml` workflow runs the whole chain in ONE Claude Code invocation (sub-agent fan-out inside),
so there's no fragile cross-job plumbing to debug first. To test:
1. Set `ANTHROPIC_API_KEY` (recommended — no parallel/rate caps) **or** `CLAUDE_CODE_OAUTH_TOKEN`
   (Max subscription, shared limits) as a repo secret; set `FRAMEWORK_TOKEN` if stage 0 should clone
   the framework repos.
2. Point the consumer caller at `feature-pipeline.driver.yml`, replace `quanticjs`, tag this repo `v1`.
3. Push a BRD to the consumer's `docs/brd/` → one PR appears with HLD + FSDs + numbered specs + tracker.
Test on a repo that has **only a BRD** (no existing HLD/FSD) for a clean end-to-end.

## Using it from a product repo (the consumer contract)

A consuming repo needs exactly three things:

1. **A BRD** in `docs/brd/`.
2. **`docs/pipeline.config.yml`** — only keys that differ from `config/pipeline.defaults.yml`
   (see `examples/trade-finance/pipeline.config.yml`).
3. **`.github/workflows/feature-pipeline.yml`** — copy `examples/consumer-caller.example.yml`, set the
   owner, configure `ANTHROPIC_API_KEY` (or Bedrock/Vertex) and `FRAMEWORK_TOKEN` secrets.

Everything else — prompts, template, actions, defaults — is pulled from this repo at the pinned ref.
Per-product specifics are **derived by the prompts from that repo's BRD/HLD**, never hardcoded here.

## Configuration highlights (`config/pipeline.defaults.yml`)

- `gates.brd_upload: end-to-end` — a BRD upload runs BRD→HLD→FSD in one run and opens a single PR.
  Set `hld-only` to gate the HLD before FSD authoring.
- `gates.mode: pr-per-stage` — each stage opens a PR (the human gate). `autopilot` removes it (stage 1 only).
- `framework.repos` — the QuanticJS sources stage 0 grounds in; `framework.checkout: true` clones them
  read-only in CI (needs `FRAMEWORK_TOKEN`). Local runs can use `framework.local_paths` instead.
- `fsd.decomposition / template / numbering / authors_parallel`, `spec.granularity / validate`,
  `runner.engine / model` — the decomposition and runner knobs (were interactive decisions in the first run).

## How the interactive run maps to CI

- The three decision-gates → committed config (overridable per consumer).
- Agent fan-out → matrix jobs (`fsd.authors_parallel`), one per FSD batch from stage 1's `batches` output.
- Human review → a PR per gate.
- Each prompt declares a **JSON output contract** so stages pass structured data (the batch plan, the
  HLD manifest) without scraping prose.

LLM steps are not byte-deterministic. Repeatable: the stages, templates, the `/review-spec` gate,
dependency order, and banded numbering — for ANY BRD.

## Going live (remaining work — all marked in-file)

1. **Pick `runner.engine`** and implement that branch in `.github/actions/run-claude/action.yml`
   (Claude Code CLI `-p` / official `claude-code-action` / Agent SDK).
2. **Implement `setup`** in the reusable workflow: merge `config` over defaults (`yq`) and infer the
   stage from the changed `docs/` dir.
3. **Implement the framework checkout** step in `brd-to-hld` (clone `framework.repos` into `.framework/`).
4. **Architect-review `prompts/00-brd-to-hld.md`** before trusting stage 0 (designed, not recalled).
5. Set
   real secrets, and **tag `v1`**.

## Reference

`examples/trade-finance/reference-run.md` is the lossless record of the first run (a Trade-Finance BRD →
25 FSDs → 92 specs): the three decisions plus the verbatim per-FSD author assignments. Use it to audit
or re-derive the generic prompts.
