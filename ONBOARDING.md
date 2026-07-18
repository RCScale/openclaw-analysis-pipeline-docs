# Onboarding a new project

A new project goes live in five steps:

1. [Get the project's IDs](#1-get-the-projects-ids-2-min).
2. [Copy `.env.example` -> `.env`](#2-copy-envexample---env-2-min) (one-time per machine).
3. [Copy `queues/_TEMPLATE.yaml` -> `queues/<pid>.yaml`](#3-copy-the-queue-template-3-min).
4. [Drop the project's grading rubric into `analysis/grading_rubric.md`](#4-supply-the-grading-rubric-1-min).
5. [Run the orchestrator](#5-run-the-orchestrator).

The full sequence takes about 25 minutes wall-clock for a typical project (most of it is two LLM-bound steps, see [Step timing & cost](#step-timing--cost)).

---

## 1. Get the project's IDs (~2 min)

You need five things from the warehouse. Open a Redash query window
pointing at Snowflake (typically data source 22, but projects on
different Snowflake accounts may use a different id -- verify with
your platform contact) and run these:

```sql
-- a) Project ID + name
SELECT _id AS project_id, name
FROM projects
WHERE name ILIKE '%<your project name>%';

-- b) The courses gating the funnel
SELECT _id AS course_id, title
FROM coursev2
WHERE _id IN (
  SELECT DISTINCT course_id FROM courseprogressv2
  WHERE project = '<project_id from (a)>'
);

-- c) The "trusted reviewer" tag (the one ops applies to promoted L1s)
SELECT _id AS tag_id, name FROM tags
WHERE name ILIKE '%<project hint>%trusted%reviewer%'
   OR name ILIKE '%<project hint>%promoted%';

-- d) Any "known bad" tags (low_quality, spam, banned, etc.)
SELECT _id AS tag_id, name FROM tags
WHERE name ILIKE '%<project hint>%bad%'
   OR name ILIKE '%<project hint>%spam%'
   OR name ILIKE '%<project hint>%low_quality%';

-- e) Verify the review-level distribution (most projects use -1/0/1)
SELECT level, COUNT(*) FROM taskattempts
WHERE project = '<project_id from (a)>'
  AND created_at_pt >= CURRENT_DATE - 60
GROUP BY level ORDER BY level;
```

Save the results. You will paste them into the queue YAML in step 3.

---

## 2. Copy `.env.example` -> `.env` (~2 min)

Done once per machine, shared across projects.

```bash
cp .env.example .env
```

Fill in:

- `REDASH_API_KEY`: from `https://redash.scale.com/users/me` -> API Key tab.
- `SCALE_DASHBOARD_JWT`, `SCALE_DASHBOARD_CSRF`, `SCALE_DASHBOARD_CSRF_COOKIE`: from your browser dev tools after logging into `dashboard.scale.com`. See the comments in `.env.example` for which cookie/header to grab.
- `RL_ENVS_DIR` (optional): absolute path to your local `rl_envs/` directory if you have one. Only used for G2 phase-b audits + universe-coverage analysis.

Verify it loaded:

```bash
python3 -c "from dotenv import load_dotenv; import os; load_dotenv(); print('REDASH:', bool(os.getenv('REDASH_API_KEY')), 'JWT:', bool(os.getenv('SCALE_DASHBOARD_JWT')))"
# expected: REDASH: True JWT: True
```

---

## 3. Copy the queue template (~3 min)

```bash
cp queues/_TEMPLATE.yaml queues/<your_project_id>.yaml
$EDITOR queues/<your_project_id>.yaml
```

Fill in every field the template marks `REQUIRED`. There are nine top-level REQUIRED blocks (`project_id`, `analysis_window`, `team_email_patterns`, `courses`, `tags`, `review_levels`, `promotion`, `json_step_map`, `warehouse`), each with its own REQUIRED leaves annotated inline. Any missing REQUIRED field raises a clear `ValueError` from `Queue.load` — no silent defaults.

The most error-prone fields:

- **`courses.reviewer`** -- exactly one course. When multiple reviewer-gating courses exist, pick the one that ops uses to decide whether to apply the promoted-reviewer tag.
- **`tags.reviewer`** -- this is the "promoted, trusted" tag. Goal-4 PDR uses this as the "known-good" cohort for tag-separation analysis.
- **`tags.quality_negative[0]`** -- run_g4_pdr uses the FIRST tag in this list as the "known-bad" cohort. If you have multiple negative tags, put the canonical one first (every tag still gets included in the per-tag correlation run by run_g4_traits).
- **`review_levels`** -- the integer mapping for your project. Step 1 (e) above shows the distribution. Most OpenClaw projects: `-1/0/1`.
- **`json_step_map`** -- if your project is OpenClaw-flavored, copy the map verbatim from `queues/698318a45989d90bf44b9b53.yaml`. If it's a different module set, see `exploration/JSON_STEP_MAP.md` for the inventory of step types.

Validate the YAML:

```bash
python3 -c "
from pipeline.lib.config import Queue
q = Queue.load('queues/<your_project_id>.yaml')
print(f'OK: project_id={q.project_id}, display_name={q.display_name!r}')
print(f'  {len(q.courses_onboarding)} onboarding course(s)')
print(f'  reviewer course: {q.course_reviewer.title}')
print(f'  trusted-reviewer tag: {q.tag_reviewer.name}')
print(f'  {len(q.tags_quality_negative)} quality-negative tag(s)')
"
```

---

## 4. Supply the grading rubric (~1 min)

The LLM auditor (G2 phase-b, G4 mistakes) reads `analysis/grading_rubric.md` to know what "good work" looks like for this project. Rubrics are stored per-project at `analysis/rubrics/<project_id>_<slug>.md`; `analysis/grading_rubric.md` is a **symlink** that the orchestrator updates at run-start to point at the right one.

Drop your project's rubric at:

```bash
cp /path/to/your/rubric.md analysis/rubrics/<project_id>_<slug>.md
```

The slug can be anything (`openclaw_mm`, `screening_v2`, ...). The orchestrator (`analysis.run_all`) glob-matches `analysis/rubrics/<project_id>_*.md` and links the first match in at `analysis/grading_rubric.md` before any LLM step runs. If no matching file is found, the orchestrator prints a warning and leaves the existing symlink alone (which means the LLM auditor would silently use the wrong project's rubric \u2014 watch for the warning).

---

## 5. Run the orchestrator

```bash
python3 -m analysis.run_all --queue queues/<your_project_id>.yaml
```

This runs the canonical sequence (see [Step timing & cost](#step-timing--cost)) end-to-end. Output:

- All G1..G4 dashboards under `analysis/visualize/`.
- Recommendations JSON at `analysis/recommendations.json`.
- The redacted public-mirror tree at `pages_build/` (push to your own GitHub Pages repo via `analysis.build_for_pages` + `git push`).

If a step fails partway, resume from that step:

```bash
python3 -m analysis.run_all --queue queues/<pid>.yaml --from C6
```

Skip the LLM-bound steps (useful when iterating on presentation):

```bash
python3 -m analysis.run_all --queue queues/<pid>.yaml --skip-llm
```

`--skip-llm` drops every step flagged `llm=True` in the plan, so it never drifts if a new LLM step is added.

---

## Step timing & cost

| Code | Step | Wall time | LLM cost | Notes |
|------|------|-----------|----------|-------|
| B1 | `pipeline.diagnose` | 3-6 min | -- | Pulls Q-A..Q-E + CDS payloads. CDS pull dominates. |
| B2 | `pipeline.validate_feedback` | 1-2 min | -- | Computes review_validation + reviewer_scorecard. |
| B3 | `analysis.fetch_raw --phase pre-g4` | 1-2 min | -- | Q-F, Q-G1, Q-G2, tag-names. |
| C1 | `analysis.run_env_coverage` | 30 s | -- | Scans CDS payloads vs local rl_envs/. |
| C2 | `analysis.run_g1` | 30 s | -- | Funnel + gate enforcement + KM curves. |
| C3 | `analysis.run_g2 phase-a` | 10 s | -- | Outlier detection + sampling. |
| C4 | `analysis.run_g2 phase-b` | **15-25 min** | **~$300** | LLM audits 200 sampled reviews in parallel. |
| C5 | `analysis.run_g2 phase-c` | 10 s | -- | Aggregates audit verdicts. |
| C6 | `analysis.run_g3` | 30 s | -- | Per-course / per-question psychometrics. |
| C8 | `analysis.run_g4 step-1` | 20 s | -- | Trust filter + materialize trusted reviews. |
| C8b | `analysis.fetch_raw --phase post-trust` | 1 min | -- | G4 traits + baseline for the trusted cohort. |
| C9 | `analysis.run_g4 step-2` | 1-2 min | -- | Per-question PDR predictivity. |
| C10 | `analysis.run_g4_mistakes step-4a` | 30 s | -- | Extracts bad-review feedback. |
| C11 | `analysis.run_g4_mistakes step-4b` | **3-5 min** | **~$20** | LLM clusters bad-review feedback. |
| C12 | `analysis.run_g4_mistakes step-4c` | **2-3 min** | **~$15** | LLM scores cluster coverage per course. |
| C13 | `analysis.run_g4_mistakes step-5` | 10 s | -- | "Covered but workers still fail" pairs. |
| C15 | `analysis.run_g4_pdr` | 20 s | -- | Per-worker PDR + tag-separation. |
| C16 | `analysis.run_g4_traits` | 30 s | -- | Per-trait + per-tag correlations. |
| C17 | `analysis.run_g4_combos` | 30 s | -- | Question composite-score curves. |
| C18 | `analysis.run_g4_tagcombos` | 30 s | -- | Tag combination synergy search. |
| C19 | `analysis.run_g4_q_subsets` | 1 min | -- | Per-course subset search. |
| C20 | `analysis.run_g4_qdiagnose extract` | 20 s | -- | Per-question evidence collection. |
| C21 | `analysis.run_g4_qdiagnose stats` | 10 s | -- | Fisher z CIs back into the predictive CSV. |
| C21b | `analysis.run_g3_production_validity` | 10 s | -- | Joins G3 psychometrics with G4 predictive to render PDR-anchored discrimination scatter. Depends on C21's Fisher CIs. |
| C22 | `analysis.run_g4_qdiagnose diagnose` | **5-10 min** | **~$30** | LLM diagnoses each predictive/counterproductive question. |
| C23 | `analysis.build_question_index` | 30 s | -- | Final g4_questions_with_pdr.csv with deprecated tag. |
| D | `analysis.build_all_dashboards --skip-goals` | ~3 s | -- | Re-renders the top-level presentation (portal, themes, summary, audit viewer, G4 dashboard) using CSVs from the C-steps above. The per-goal HTMLs (g1/g2/g3/g4/env_coverage) were emitted directly by their C-step. |

**Total wall time:** ~25-40 minutes for a fresh project.

**Total LLM cost:** **~$365** per fresh run — ~$300 for the G2 phase-b reviewer audit, ~$35 for the G4 mistake-clustering pair (C11 + C12), and ~$30 for the per-question G4 qdiagnose (C22). The G2 audit and G4 qdiagnose caches survive across reruns (`analysis/audit_outputs/` and `analysis/g4_question_diagnoses.json`); the mistake-clustering steps have no cache today, so a data refresh has a ~$35 floor.

---

## After onboarding: re-runs

A pure presentation re-run (after editing copy or design tokens) skips everything except the dashboard builders:

```bash
python3 -m analysis.build_all_dashboards --queue queues/<pid>.yaml --skip-goals
```

This finishes in ~3 seconds and re-uses every existing CSV.

A re-fetch of just the raw warehouse data (e.g. weekly refresh of last-week's funnel):

```bash
python3 -m pipeline.diagnose --queue queues/<pid>.yaml
python3 -m analysis.fetch_raw --queue queues/<pid>.yaml --phase pre-g4
python3 -m analysis.run_all --queue queues/<pid>.yaml --from C2  # skip diagnose
```

---

## Troubleshooting

### `--queue` errors out with "Missing key '<x>' in <y>"

Your YAML is missing a REQUIRED field. The error message names the field; check the template for what to put there.

### `pipeline.diagnose` finishes in 30 seconds and creates 0 rows

The `team_email_patterns` ILIKE filter is too narrow (excluded every attempter). Cross-check with:

```sql
SELECT DISTINCT u.email FROM taskattempts t JOIN users u ON u._id = t.user
WHERE t.project = '<pid>' LIMIT 20;
```

Then loosen the patterns in your queue YAML.

### `run_g4_pdr` warns "queue.tags.quality_negative is empty"

Add at least one known-bad tag to `tags.quality_negative` in your YAML, or accept that the tag-separation column in `g4_question_pdr.csv` will be null. The downstream dashboards still work; they just lose one of the two cross-validation signals.

### LLM steps (C4, C11, C12, C22) fail with `claude: command not found`

Install the Claude CLI with `npm install -g @anthropic-ai/claude-code`, authenticate with `claude login`, and confirm with `claude --version`. The pipeline requires the CLI on `PATH` — the Python `anthropic` SDK is not sufficient.

### Pages build (`analysis.build_for_pages`) has a 100 MB file

The audit evidence viewer is intentionally excluded from the Pages mirror; if it appears in `pages_build/`, you ran the builder with `--include-audit-viewer`. Drop the flag.
