---
layout: default
title: OpenClaw Analysis Pipeline — Reference
description: How we built an analysis pipeline for an evaluation platform. Design decisions, transferable patterns, and deferred work.
---

# Reference: How we built the OpenClaw analysis pipeline

**Audience.** You are designing an analysis pipeline for an evaluation
platform (rubric-driven graded tasks, reviewer feedback, worker
progression). You want to see how we approached a similar problem and
lift the pieces that transfer.

---

## 1. The problem shape

We were asked to answer four questions about an OpenClaw evaluation
project:

1. **Workers** — who is stuck in the funnel, who produces defects,
   which traits predict a low defect rate?
2. **Reviewers** — which reviewers can be trusted to grade work?
   Which should be dropped?
3. **Courses** — which onboarding courses are net-helpful vs
   net-harmful in aggregate? Where is the curriculum missing coverage
   of frequent worker mistakes?
4. **Questions** — which specific course questions to rewrite,
   remove, or keep, with the full question text and correct answer
   inline?

The primary outcome metric is **Performance Defect Rate (PDR)**: for
each worker, the fraction of their last three trusted reviews scoring
below the promotion gate (QM &lt; 4). We anchor every predictive
claim in the analysis to PDR because it is (a) computed from
production reviews and not course-quiz answers, (b) trust-filtered
before aggregation so untrusted reviewers are excluded, and (c)
directly interpretable as "fraction of shipped work that would be
rejected".

---

## 2. The trust chain

The single most important design decision in the pipeline is the
trust chain. Everything downstream depends on it being sound.

```
raw reviewer feedback  (Q-C from warehouse + CDS payload)
   |  LLM audits each claim in each feedback comment
   v  (Claude Opus, --effort max, --json-schema)
per-claim verdicts  (defensible / questionable / unjustified)
   |  majority-unjustified reviewers dropped from the trust pool
   v
trusted reviewer set
   |  filter raw reviews to those written by trusted reviewers
   v
trusted review corpus
   |  compute per-worker PDR from last 3 trusted reviews
   v
per-worker PDR
   |  point-biserial correlate PDR with per-question performance
   v  during onboarding
per-question predictivity  (r_pdr with Fisher-z CIs)
   |  cross-validate against platform's own worker tags
   v  (trusted_reviewer, bad_quality)
double-confirmed question flags
   |  LLM diagnoses each flagged question
   v  (root cause + recommended action)
per-question keep / rewrite / remove / investigate_grader
```

Every hop is drillable end-to-end from the dashboards: click a
question recommendation → source reviews for that question, click a
reviewer trust score → the audit evidence that dropped them. This
transparency is what makes operations-team stakeholders willing to
act on the LLM outputs.

---

## 3. What to steal

### 3.1. LLM-as-auditor on the reviewer's own feedback

Reviewer trust is typically inferred from inter-reviewer agreement or
from noisy operator sampling. Neither scales. We had Claude Opus read
each sampled review's `(prompt, rubric, worker deliverable, reviewer
feedback)` and score every claim in the feedback as `defensible` /
`questionable` / `unjustified`, with `--json-schema` enforcing the
output format so aggregation is deterministic.

Implementation details worth copying:

- Sample ~200 reviews per project with outlier-weighted plus
  control-random sampling (not pure random — you want the tails).
- Materialize the "trinity bundle" (prompt, rubric, worker output,
  reviewer feedback) into a per-attempt directory before dispatching,
  so the LLM never sees other reviews and audit runs are cacheable
  per attempt.
- Cache per-attempt outputs on disk. Re-runs after a few new reviews
  cost ~$1.50 per new review, not $300 for the whole batch.
- Use the largest reasoning model available. Claim-by-claim verdicts
  on ambiguous rubric criteria need reasoning-heavy models — the
  cheaper models produce plausible-sounding but inconsistent verdicts.
- Prompt the model to distinguish "zero-claim feedback" (short
  reviews with no substantive claims) as its own case, so those
  reviews are not scored as `defensible` by default.

### 3.2. Trust-weighted, production-anchored outcome metric

Anchor every claim about workers, courses, and questions to a metric
that (a) is computed from shipped production reviews, and (b) filters
out untrusted reviewers before aggregation. We used PDR as defined in
§1. Course-quiz performance is used only as a predictor of PDR,
never as the outcome variable itself.

This blocks a specific failure mode: a course question appearing to
"predict" positive outcomes when the outcome is itself an artifact of
generous reviewers who happen to also give worse workers a pass on
that question. Filtering the outcome by trust breaks this correlation.

### 3.3. Two-axis cross-validation

Every question flag from the correlation analysis (predictive or
counterproductive against PDR) is cross-checked against a second,
independent axis: the platform's pre-existing `trusted_reviewer` and
`bad_quality` worker tags. Agreements are double-confirmed;
disagreements are flagged as candidates for manual re-audit.

Two-axis validation turned out to be the single largest source of
stakeholder trust in the recommendations. A statistical flag alone is
easy to dismiss; a statistical flag that also matches the platform's
own worker labels is not.

### 3.4. Recover deprecated content from version history

Course questions get removed from the live course all the time. Naive
analysis skips them and produces stale "these are your problem
questions" lists — half of which are already resolved. We pull
removed questions from
`PUBLIC_W_DELETED.COURSEV2VERSIONHISTORIES` (Scale's Snowflake
version-history table) and stamp them `deprecated=true` in the
output. Operators can then distinguish "this is already fixed" from
"this still needs action".

If your platform stores content in a versioned schema, the equivalent
step is worth building on day one. Doing it later means every
recommendation list you ship carries known-stale items.

### 3.5. Per-flagged-question LLM diagnosis

For every question the correlation analysis flags as predictive or
counterproductive, dispatch a second LLM pass that reads
`(question text, sample worker answers, grader hints)` and returns:

- a **root cause class** — one of `answer_key_broken`,
  `trait_misalignment`, `prompt_ambiguous`, `noise_or_low_signal`,
  `genuinely_predictive`;
- a **recommended action** — one of `keep`, `rewrite`, `remove`,
  `investigate_grader`.

This turns a raw statistical flag into an actionable operator
recommendation. Cache per question, so refreshes after re-running the
statistics only re-diagnose questions whose predictivity actually
changed.

### 3.6. Static-HTML dashboards, no build step

Every dashboard renders as a static HTML file from a set of vanilla
Python builders (`analysis/build_*.py`). The design system is one CSS
token file plus a vanilla-JS component library (sortable tables,
filterable rows, tooltip modals). No React, no npm, no bundler.

Consequences worth the trade-offs:

- Redistribution is `rsync` a folder and serve it — no runtime.
- Public redaction is a text-substitution pass (hash emails and Mongo
  IDs) before publishing to GitHub Pages.
- The whole output survives inside `archive/<project_id>/` — one
  snapshot preserves the CSV analytics and the rendered dashboards
  together, so past states of the analysis stay browsable.
- The dashboards are pinnable in Slack, printable, and openable on a
  phone without a login.

### 3.7. Per-project rubric handoff via symlink

The LLM auditor reads a fixed path `analysis/grading_rubric.md`.
Actual rubrics live per project at
`analysis/rubrics/<project_id>_*.md`. The orchestrator swaps the
symlink at run-start based on the queue YAML being processed. Two
projects, two rubrics, one code path.

If we did this again we would fail hard when no matching rubric
exists (today we warn and continue, which risks silently auditing
project A's reviews against project B's rubric).

### 3.8. Queue YAML as the single source of project-specific config

Every project-specific value — course IDs, tag IDs, promotion FSM
policy, JSON step-map, warehouse identifiers, review-level integer
mapping — lives in one annotated YAML per project
(`queues/<pid>.yaml`) loaded through a typed dataclass
(`pipeline.lib.config.Queue`). No hardcoded project IDs anywhere in
the code. New project = new YAML, no code edits.

The dataclass raises a clear `ValueError` on any missing `REQUIRED`
field. This is boring and pays off every single onboarding.

---

## 4. Data flow at a glance

Four layers, each with distinct dependencies and failure modes:

- **Layer 1 — Warehouse pull.** Redash → Snowflake queries
  (`Q-A`..`Q-G`) plus a dashboard CDS payload download for per-attempt
  JSON blobs. Writes `reports/<project_id>/_raw/*.csv`. Cost:
  negligible ( &lt; $1).
- **Layer 2 — LLM dispatch.** Three sub-layers, all pattern-matched
  the same way ("sample → materialize evidence → dispatch to Claude →
  cache per record → aggregate"): reviewer feedback audit, mistake
  clustering, per-question diagnosis. Cost: ~$365 on a fresh project.
- **Layer 3 — Analytics.** Pure Python, no pandas. Trust filter,
  per-question predictivity (point-biserial + Fisher-z CIs),
  per-worker PDR, trait correlations, question composite curves,
  tag-bundle synergy. Cost: seconds to minutes.
- **Layer 4 — Presentation.** ~5 KLOC of static-HTML generators
  produce 14 dashboards (a landing portal, 4 cross-cutting theme
  pages, 7 goal-specific deep-dives, an executive summary, an
  audit-evidence viewer). Cost: ~3 seconds.

The layer invariant we tried to hold (and mostly did): each layer has
one canonical output shape that downstream layers read from. The
important consequence is that a bug in one layer never corrupts the
inputs of another — it only silently produces bad outputs at its own
tier, which a manifest-per-artifact scheme (§5 item 2) would catch.

```
Layer 1: Warehouse       Layer 2: LLM              Layer 3: Analytics         Layer 4: Presentation
-------------------      -----------------------   ------------------------   ---------------------
pipeline.diagnose  --->  run_g2 phase-b            run_g4 step-1/2       ---> build_all_dashboards
analysis.fetch_raw ----> run_g4_mistakes 4b/4c --> run_g4_pdr / traits    -->    -> visualize/*.html
validate_feedback  ----> run_g4_qdiagnose ------>  run_g4_combos          -->
                                                   build_question_index  --->  build_for_pages
                                                                                -> pages_build/
```

---

## 5. What we would do differently

Ranked by leverage. Every item is a known limitation we chose to
defer during the first two-project rollout.

1. **Namespace derived artifacts per project.** Today
   `analysis/g*_*.csv`, `analysis/audit_outputs/`, and
   `analysis/visualize/` are shared globally across projects;
   switching queues overwrites the previous project's output. Today's
   workaround is an `archive_project.sh` script that snapshots to
   `archive/<project_id>/`. The right fix is
   `artifacts/<project_id>/` as the canonical write path — bake this
   in on day one.
2. **Manifest-per-artifact.** Each output CSV should have a sibling
   JSON containing `project_id`, queue YAML hash, rubric hash, input
   file hashes, schema version. Every reader verifies before
   consuming. This one change would have prevented all three
   "silent stale-data" bugs we shipped:
   - Supplementary courses filtered out of the SQL and never noticed
     until a stakeholder asked "why isn't course X on the dashboard?".
   - A body-cache JSON that went stale because a builder updated its
     CSV output without updating the sibling JSON that dashboards
     actually read from.
   - Hardcoded chatv2 defaults (report date, tag IDs, RL-envs path)
     that produced wrong-but-not-obviously-wrong output for the
     second project until we deparameterized them.
3. **Semantic cache invalidation.** Today the orchestrator's freshness
   check is mtime-only, so editing the queue YAML to add a course
   does not invalidate downstream outputs. Content-hashing the
   inputs and dependencies solves this.
4. **Cache the mistake-clustering LLM steps.** The reviewer audit
   and per-question diagnosis are cached per record; the two
   mistake-clustering steps re-cluster from scratch on every run, so
   every data refresh pays a ~$35 floor.
5. **Auto-refresh the dashboard cookies.** The `SCALE_DASHBOARD_*`
   JWT cookies rotate every ~24 hours; the pipeline degrades to
   HTTP 401 with no automatic re-auth. Better to fail-fast with a
   clear message.
6. **Fail hard on missing rubric.** As mentioned in §3.7, today the
   rubric symlink logs a warning and continues. A $300 audit run
   against the wrong rubric is a much more expensive failure than a
   startup-time hard-stop; the orchestrator should exit on any
   ambiguous rubric resolution.

---

## 6. Cost and time model

**Fresh-project run:**

- ~25–40 minutes wall clock; the four LLM steps dominate.
- ~$365 in Anthropic API calls: ~$300 for the 200-review G2 audit,
  ~$35 for mistake clustering, ~$30 for per-question diagnosis.
- ~$0 Snowflake compute; the queries hit `courseprogressv2` and
  `taskattempts` and finish in seconds.

**Data refresh (new week of reviews, existing G2 audit cache
preserved):**

- ~5 minutes wall clock.
- ~$35 floor from the uncached mistake-clustering steps, plus
  ~$1.50 per new sampled review and ~$3 per newly-flagged predictive
  or counterproductive question.

**Engineer time to onboard a new project** (given the pipeline is
already stood up): ~10 minutes to fill the queue YAML, plus one
fresh pipeline run in the background.

---

## 7. See it live

The chatv2 project's rendered dashboards live at
[rcscale.github.io/openclaw-dashboard](https://rcscale.github.io/openclaw-dashboard/)
as a redacted public mirror. The four cross-cutting theme pages
(Courses / Questions / Reviewers / Workers) are the most operationally
useful views; the per-goal deep-dives underneath (G1..G4 +
Universe Coverage) expose the underlying methodology and raw tables.

---

## 8. Where the code lives

For reference against your own layout — the pipeline's source tree is
partitioned by responsibility, one Python package per layer:

- `pipeline/` — warehouse-pull layer. `diagnose.py` (`Q-A`..`Q-E` +
  CDS payload downloads), `validate_feedback.py` (QM score extraction
  + reviewer scorecard), `lib/` (queue-config loader, SQL builders,
  Redash and CDS HTTP clients).
- `analysis/` — everything downstream of the warehouse. Fetchers
  (`fetch_raw.py`), per-goal runners (`run_g1.py` through
  `run_g4_qdiagnose.py`), the 27-step orchestrator (`run_all.py`),
  presentation builders (`build_*.py`), shared helpers (`lib/`),
  per-project rubrics (`rubrics/`), and the rendered HTML output
  (`visualize/`).
- `queues/` — one annotated YAML per project, plus the
  `_TEMPLATE.yaml` reference with `REQUIRED` / `OPTIONAL` field
  annotations.
- `exploration/` — phase-1 warehouse discovery scripts and schema
  notes; useful if you need to map an unfamiliar variant of the
  underlying platform.
