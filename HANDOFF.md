# Handoff: OpenClaw Analysis Pipeline

## What this is

A repeatable pipeline that turns raw warehouse data about an OpenClaw
project into a public-shareable set of dashboards that answer four
operational questions:

1. **Workers** — who is stuck in the funnel, who is producing defects,
   which traits predict a low defect rate?
2. **Reviewers** — which reviewers can be trusted to grade work, which
   should be dropped from the trust pool?
3. **Courses** — which courses are net-helpful vs net-harmful in
   aggregate, and which frequent worker mistakes are missing from the
   curriculum entirely?
4. **Questions** — which specific course questions to rewrite, remove,
   or keep, with the full question text and correct answer inline.

The pipeline has run end-to-end for one project (`OpenClaw chatv2`,
`698318a45989d90bf44b9b53`) and been configured for a second
(`OpenClaw MM`, `69f95a0f0992772af7907a03`). The chatv2 output is
published at [rcscale.github.io/openclaw-dashboard](https://rcscale.github.io/openclaw-dashboard/)
as a redacted public mirror.

## What it produces

For each project, running the pipeline end-to-end produces:

- **14 HTML dashboards** under `analysis/visualize/`:
  - Landing portal (`index.html`), printable executive summary
    (`summary.html`), and audit-evidence viewer
    (`g2_audit_review.html`) — 3 entry points.
  - Four cross-cutting theme pages
    (`themes/courses.html`, `themes/questions.html`,
    `themes/reviewers.html`, `themes/workers.html`) that operators
    spend most time in.
  - Seven per-goal deep-dives (`g1.html`, `g2.html`,
    `g2_distributions.html`, `g3.html`, `g3_production_validity.html`,
    `g4.html`, `g_universe_coverage.html`).
- **~30 CSV / JSON analysis artifacts** under `analysis/g*_*` covering
  every intermediate output (trust scores, per-question predictivity,
  mistake clusters, per-worker outcomes, tag correlations).
- **A redacted static-site build** under `pages_build/` that hashes out
  worker emails + Mongo IDs so the output is safe to publish externally.
- **A per-project archive path** (`archive/<project_id>/`) so switching
  between projects is a snapshot-and-restore operation with no
  re-run of the LLM audit steps.

## What is unique about this pipeline

- **LLM auditor on reviewer feedback itself.** For every sampled
  review, an LLM sub-agent (Claude Opus, `--effort max`) reads the
  prompt, rubric, worker deliverable, and reviewer feedback, then
  judges every claim in the feedback as `defensible` / `questionable`
  / `unjustified`. Reviewers whose majority of audited feedback comes
  back unjustified are dropped from the trust pool.
- **Trust-weighted PDR.** The per-worker outcome metric is
  Performance Defect Rate — the fraction of a worker's last 3 trusted
  reviews scoring below the promotion gate (QM &lt; 4). Reviews from
  dropped reviewers are excluded before this metric is computed.
- **Two-axis cross-validation.** Every predictive-or-counterproductive
  question flag is cross-checked against a second axis: the platform's
  `trusted_reviewer` and `bad_quality` worker tags. Agreements
  double-confirm the flag; disagreements surface as manual re-audit
  candidates.
- **Version-history recovery.** Questions removed from the live course
  definition are recovered from
  `PUBLIC_W_DELETED.COURSEV2VERSIONHISTORIES` and stamped
  `deprecated=true` in the output, so operators can distinguish
  resolved problems from open ones.
- **LLM per-question diagnosis.** For every question flagged as
  predictive or counterproductive, a second LLM pass reads the
  question text, sample worker answers, and grader hints, then
  classifies the root cause (`answer_key_broken` / `trait_misalignment`
  / `prompt_ambiguous` / `noise_or_low_signal` / `genuinely_predictive`)
  and recommends `keep` / `rewrite` / `remove` / `investigate_grader`.

## What is in place vs known limitations

### In place

- Two-project reference implementation with a documented queue-YAML
  schema (`queues/_TEMPLATE.yaml`), an env-var reference
  (`.env.example`), and a 10-minute new-project walkthrough
  (`ONBOARDING.md`).
- Committed fetch scripts for every raw warehouse pull; no ad-hoc
  Redash queries required to onboard.
- Orchestrator (`analysis.run_all`) that runs the canonical 27-step
  sequence with `--from`, `--only`, `--skip`, `--skip-llm`,
  `--force`, `--dry-run` flags. Mtime-based checkpointing to skip
  already-fresh steps.
- Per-project grading rubrics stored at
  `analysis/rubrics/<project_id>_*.md` with an orchestrator-managed
  symlink so LLM steps automatically read the correct rubric for the
  current run.
- Git repository at the workspace root; every source edit committed;
  outputs gitignored so a clean checkout is code-only.
- Design-system-driven presentation layer (single CSS token file +
  vanilla-JS component set; no build step, no npm).

### Known limitations (in priority order)

- **Derived artifacts share one global filesystem namespace.** Files
  under `analysis/g*_*.csv`, `analysis/audit_outputs/`, and
  `analysis/visualize/` are shared across projects; switching queues
  overwrites the previous project's output. Today's workaround: the
  `archive_project.sh` script snapshots to `archive/<project_id>/`
  before switching. Long-term fix: move every derived artifact to
  `artifacts/<project_id>/` (~2 days of work; see
  `HANDOFF_TECHNICAL.md` §5).
- **Cache invalidation is mtime-only.** Reruns compare output mtime
  against input mtimes. This misses semantic changes — editing the
  queue YAML to add a course without touching a CSV leaves downstream
  outputs stale. Three fix commits (`a1c566c`, `db7224a`, `e9f8e0d`)
  patched specific symptoms of this pattern.
- **The `SCALE_DASHBOARD_*` cookies rotate every ~24 hours.** CDS
  payload downloads (used by G2 phase-b audits and env-coverage)
  return HTTP 401 when the JWT expires. Refresh is manual; there is
  no automatic re-auth path.
- **LLM caches are keyed only by attempt ID.** Per-attempt audit
  outputs at `analysis/audit_outputs/<attempt_id>.json` carry no
  project ID prefix. If two projects share an attempt ID (rare in
  practice), whichever project wrote the cached audit first wins
  and the other project silently reuses it.
- **Presentation builders do not accept `--queue`.**
  `build_themes.py`, `build_summary.py`, and `build_g4_dashboard.py`
  read whatever `analysis/g4_*.csv` is on disk. Combined with the
  global-derived-artifacts pattern above, this is why the
  archive-and-restore workflow exists.
- **Cost surface is high on a fresh run.** ~$365 in Claude API calls
  per fresh project (see cost breakdown in `HANDOFF_TECHNICAL.md`
  §4). Two of the four LLM steps (mistake clustering step-4b and 4c)
  re-cluster from scratch every run, so data refreshes have a ~$35
  floor. The other two steps (G2 phase-b reviewer audit, G4
  qdiagnose) cache per-attempt / per-question and add cost only for
  new records.

## Where to go from here

- **To onboard a third project**: read `ONBOARDING.md`. It's a
  10-minute walkthrough (queue YAML + `.env` + rubric + one command).
- **To understand what the pipeline computes and how**: read
  `HANDOFF_TECHNICAL.md`. Covers architecture, data flow, cost model,
  every entrypoint, and a section on invariants + failure modes.
- **To reproduce the chatv2 run**: everything is in
  `archive/698318a45989d90bf44b9b53/`. Restore via
  `rsync -av archive/698318a45989d90bf44b9b53/ ./` and re-run
  `analysis.run_all` with the chatv2 queue.
- **To operate the pipeline as a scheduled refresh**: today
  `analysis.run_all` is invoked manually. A weekly loop needs a cron
  or GitHub Action wrapper around `analysis.run_all` +
  `analysis.build_for_pages` + `git push` from the `pages_build`
  submodule.

## Directory reference

```
/                              workspace root (git repo)
  .env                         local secrets (gitignored)
  .env.example                 template
  ONBOARDING.md                new-project walkthrough
  HANDOFF.md                   this file
  HANDOFF_TECHNICAL.md         deep architectural reference
  archive_project.sh           snapshot script (per-project archive)

  queues/                      one YAML per project
    _TEMPLATE.yaml             annotated template
    <project_id>.yaml          per-project config

  pipeline/                    warehouse-pull layer
    diagnose.py                Q-A..Q-E + CDS pulls, worker-state FSM
    validate_feedback.py       QM extraction + reviewer scorecard
    lib/                       config loader, SQL builders, Redash + CDS clients

  analysis/                    everything downstream of the warehouse
    fetch_raw.py               Q-F, Q-G1, Q-G2, G4 traits + tag pulls
    run_all.py                 top-level orchestrator (27-step sequence)
    run_g1.py                  funnel + gates + KM survival curves
    run_g2.py                  reviewer sampling + LLM audit + trust
    run_g3.py                  course psychometrics (CTT)
    run_g3_production_validity.py  Phase D: PDR-anchored discrimination
    run_g4.py                  trust filter + per-question predictivity
    run_g4_pdr.py              per-worker PDR + tag-based cross-validation
    run_g4_traits.py           per-trait correlations
    run_g4_combos.py           question composite-score curves
    run_g4_q_subsets.py        per-course subset search
    run_g4_tagcombos.py        tag combination synergy
    run_g4_mistakes.py         mistake clustering + course-coverage LLM
    run_g4_qdiagnose.py        per-question LLM diagnoses
    run_env_coverage.py        RL env acquisition priority
    build_question_index.py    joined per-question CSV + bodies JSON
    build_recommendations.py   deterministic "what to do this week"
    build_portal.py            landing page
    build_summary.py           printable executive summary
    build_themes.py            four theme pages
    build_g4_dashboard.py      deep-dive walkthrough
    build_audit_viewer.py      audit-evidence card viewer
    build_all_dashboards.py    orchestrator (presentation only)
    build_for_pages.py         redact + copy to pages_build/
    lib/                       shared helpers (audit prompt, trust rules,
                               psychometrics, KM, outliers, UI components,
                               Plotly layout)
    rubrics/                   per-project grading rubric
    visualize/                 rendered HTML output
      _assets/                 shared design system (CSS + JS)
    audit_inputs/              trinity bundles per attempt (gitignored)
    audit_outputs/             LLM verdicts per attempt (gitignored)
    audit_logs/                per-audit CLI logs (gitignored)
    g4_mistakes/               mistake-clustering intermediates
    g4_qdiagnose/              per-question evidence bundles

  reports/                     per-project warehouse data (gitignored)
    <project_id>/
      _raw/                    Q-A.csv, Q-B.csv, ..., cds/<attempt>.json
      <YYYY-MM-DD>/            dated diagnose outputs

  pages_build/                 redacted public-mirror tree (separate git repo)

  archive/                     per-project snapshots (gitignored)
    <project_id>/              full snapshot of all outputs for one project

  exploration/                 phase-1 warehouse discovery + schema notes
  queries/                     top-level SQL (sanity_g4.sql + one archive/)
  seed_from_v1/                legacy reference (unused today)
```
