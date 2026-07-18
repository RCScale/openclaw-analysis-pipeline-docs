# OpenClaw Analysis Pipeline

A repeatable pipeline that turns raw warehouse data about an OpenClaw
project into a public-shareable set of dashboards, answering four
operational questions:

- **Workers** — who's stuck, who's producing defects, what traits predict low defect rate?
- **Reviewers** — who can be trusted to grade work, who should be dropped?
- **Courses** — which questions predict downstream production quality, which are counterproductive?
- **Questions** — which specific course questions to rewrite, remove, or keep?

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
# 1. Restore the archived outputs for chatv2 (this workspace was cleaned
#    to code-only; chatv2 outputs sit in archive/).
rsync -av archive/698318a45989d90bf44b9b53/ ./

# 2. Point the rubric symlink at the chatv2 rubric and preview the site.
python3 -m analysis.run_all --queue queues/698318a45989d90bf44b9b53.yaml --dry-run
python3 -m http.server --directory analysis/visualize 8000
```

To onboard a new project: see [`ONBOARDING.md`](./ONBOARDING.md).

## Layout

```
queues/          per-project YAML config
pipeline/        warehouse-pull layer (SQL, Redash + CDS clients, diagnose FSM)
analysis/        everything downstream: fetch, LLM audits, math, presentation
  run_all.py     orchestrator (26-step canonical sequence)
  rubrics/       per-project grading rubric
  visualize/     rendered HTML output
    _assets/     shared design system (CSS + JS)
reports/         per-project warehouse data (gitignored)
archive/         per-project snapshots (gitignored)
pages_build/     redacted public mirror (separate git repo)
```

## What makes this pipeline unique

- **LLM auditor on reviewer feedback itself** — every claim in a
  reviewer's free-text feedback gets a `defensible` / `questionable`
  / `unjustified` verdict before their reviews are trusted downstream.
- **Trust-weighted PDR** — worker outcome is the fraction of last 3
  trusted reviews scoring below the promotion gate. Untrusted
  reviewers filtered out before the metric is computed.
- **Two-axis cross-validation** — every question flag double-checked
  against the platform's pre-existing worker tags. Disagreements
  surface as re-audit candidates.
- **Deprecated-question recovery** — questions removed from the live
  course are still recovered from `PUBLIC_W_DELETED.COURSEV2VERSIONHISTORIES`
  so operators know which "problems" are already resolved.
- **LLM per-question diagnosis** — for every counterproductive
  question, a second LLM pass classifies the root cause and
  recommends `keep` / `rewrite` / `remove` / `investigate_grader`.

## Fresh-project cost

- ~25-40 minutes wall clock
- ~$365 in Claude API calls (mostly the 200-review audit; per-attempt
  cached for re-runs)
- ~$0 Snowflake compute
