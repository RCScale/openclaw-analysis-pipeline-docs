---
layout: default
title: OpenClaw Analysis Pipeline
---

# OpenClaw Analysis Pipeline — Documentation

A repeatable pipeline that turns raw warehouse data about an OpenClaw
project into a public-shareable set of dashboards, answering four
operational questions:

- **Workers** — who is stuck in the funnel, who is producing defects,
  which traits predict a low defect rate?
- **Reviewers** — which reviewers can be trusted to grade work, which
  should be dropped from the trust pool?
- **Courses** — which courses are net-helpful vs net-harmful, and
  which frequent worker mistakes are missing from the curriculum
  entirely?
- **Questions** — which specific course questions to rewrite, remove,
  or keep (with the full question text and correct answer inline)?

## Documentation

Four documents in reading order:

- **[README](README.html)** — one-page overview and directory layout.
- **[HANDOFF](HANDOFF.html)** — new-maintainer first-hour read.
  Covers what the system produces, what makes it unique, and the
  known limitations.
- **[ONBOARDING](ONBOARDING.html)** — 10-minute walkthrough for a new
  project: queue YAML → `.env` → rubric → one command.
- **[HANDOFF_TECHNICAL](HANDOFF_TECHNICAL.html)** — deep architectural
  reference. Four layers, the 26-step canonical sequence, cost model,
  invariants, and recommended next investments.

## Reference materials

- **[Queue YAML template](queue_template.yaml)** — annotated schema
  for the per-project config file (11 required fields + optional
  defaults).
- **[Environment template](.env.example)** — required secrets (Redash
  API key, dashboard cookies) with instructions on where to get each.

## Example output

The chatv2 project's rendered dashboards live at
[rcscale.github.io/openclaw-dashboard](https://rcscale.github.io/openclaw-dashboard/)
as a redacted public mirror.

## What ships with the pipeline

- End-to-end orchestrator (`analysis.run_all`) with resumable /
  skippable steps.
- Committed fetch scripts for every raw warehouse pull.
- Per-project grading rubrics with automatic symlink management.
- Design-system-driven dashboards (single CSS token file + vanilla-JS
  component set; no build step).
- ~$365 in Claude API calls per fresh project; re-runs cost far less
  because per-attempt and per-question caches survive.
