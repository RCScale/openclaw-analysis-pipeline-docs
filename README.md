# OpenClaw Analysis Pipeline

A repeatable pipeline that turns raw warehouse data about an OpenClaw
project into a public-shareable set of dashboards, answering four
operational questions:

- **Workers** — who is stuck in the funnel, who is producing defects,
  which traits predict a low defect rate?
- **Reviewers** — which reviewers can be trusted to grade work, which
  should be dropped from the trust pool?
- **Courses** — which courses are net-helpful vs net-harmful, and
  which frequent mistakes are missing from the curriculum entirely?
- **Questions** — which specific course questions to rewrite, remove,
  or keep (with the full question text and correct answer inline)?

## Start here

- **New to the pipeline?** Read [`HANDOFF.md`](./HANDOFF.md). One page,
  answers "what is this / what does it produce / what are the
  limitations".
- **Onboarding a new project?** Read [`ONBOARDING.md`](./ONBOARDING.md).
  10-minute walkthrough: queue YAML → `.env` → rubric → one command.
- **Need architectural depth?** Read
  [`HANDOFF_TECHNICAL.md`](./HANDOFF_TECHNICAL.md). Every layer, every
  entrypoint, the cost model, invariants, and recommended next
  investments.

## Try it

The chatv2 project is fully onboarded. Its rendered dashboards live at
[rcscale.github.io/openclaw-dashboard](https://rcscale.github.io/openclaw-dashboard/)
(redacted public mirror).

To reproduce locally:

```bash
# 1. Restore the archived outputs for chatv2 (this workspace ships
#    code-only; chatv2 outputs live in archive/).
rsync -av archive/698318a45989d90bf44b9b53/ ./

# 2. --dry-run runs the "point the rubric symlink" side effect and
#    prints the plan without executing steps. The rendered HTML is
#    already inside the restored archive, so no rebuild is needed.
python3 -m analysis.run_all --queue queues/698318a45989d90bf44b9b53.yaml --dry-run
python3 -m http.server --directory analysis/visualize 8000
```

To onboard a new project: see [`ONBOARDING.md`](./ONBOARDING.md).

## Layout

```
queues/          per-project YAML config
pipeline/        warehouse-pull layer (SQL, Redash + CDS clients, diagnose FSM)
analysis/        everything downstream: fetch, LLM audits, math, presentation
  run_all.py     orchestrator (27-step canonical sequence)
  rubrics/       per-project grading rubric
  visualize/     rendered HTML output (14 dashboards total)
    _assets/     shared design system (CSS + JS)
reports/         per-project warehouse data (gitignored)
archive/         per-project snapshots (gitignored)
pages_build/     redacted public mirror (separate git repo)
```

## What makes this pipeline unique

- **LLM auditor on reviewer feedback itself.** Every claim in a
  reviewer's free-text feedback gets a `defensible` / `questionable`
  / `unjustified` verdict before their reviews are trusted downstream.
- **Trust-weighted PDR.** Worker outcome is the fraction of a
  worker's last three trusted reviews scoring below the promotion
  gate. Reviews from dropped reviewers are excluded before the metric
  is computed.
- **Two-axis cross-validation.** Every question flag is checked
  against a second axis — the platform's pre-existing `trusted_reviewer`
  and `bad_quality` worker tags. Agreements are double-confirmed;
  disagreements surface as re-audit candidates.
- **Deprecated-question recovery.** Questions removed from the live
  course are recovered from `PUBLIC_W_DELETED.COURSEV2VERSIONHISTORIES`
  and stamped `deprecated=true` so operators know which flagged
  problems are already resolved.
- **LLM per-question diagnosis.** For every question flagged as
  predictive or counterproductive, a second LLM pass classifies the
  root cause (`answer_key_broken` / `trait_misalignment` /
  `prompt_ambiguous` / `noise_or_low_signal` / `genuinely_predictive`)
  and recommends `keep` / `rewrite` / `remove` /
  `investigate_grader`.

## Fresh-project cost

- ~25-40 minutes wall clock.
- ~$365 in Claude API calls. The 200-review G2 audit is the largest
  slice at ~$300 and is cached per-attempt (re-runs after new data
  land pay only for new attempts).
- ~$0 Snowflake compute.
- Data refreshes (fresh warehouse pull, existing G2 audit cache
  reused) still pay a ~$35 floor for the two G4 mistake-clustering
  LLM steps, which re-cluster from scratch every run.
