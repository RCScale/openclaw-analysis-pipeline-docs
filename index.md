---
layout: default
title: OpenClaw Analysis Pipeline
---

# OpenClaw Analysis Pipeline — Documentation

A repeatable pipeline that turns raw warehouse data about an OpenClaw
project into a public-shareable set of dashboards, answering four
operational questions:

- **Workers** — who's stuck, who's producing defects, what traits predict low defect rate?
- **Reviewers** — who can be trusted to grade work, who should be dropped?
- **Courses** — which questions predict downstream production quality, which are counterproductive?
- **Questions** — which specific course questions to rewrite, remove, or keep?

## Documentation

Four documents in reading order:

- **[README](README.html)** — one-page overview + directory layout.
- **[HANDOFF](HANDOFF.html)** — new maintainer, first-hour read. What
  the system produces, what makes it unique, what the known
  limitations are.
- **[ONBOARDING](ONBOARDING.html)** — 10-minute walkthrough for
  onboarding a new project. Queue YAML → `.env` → rubric → one command.
- **[HANDOFF_TECHNICAL](HANDOFF_TECHNICAL.html)** — deep architectural
  reference. Four layers, the 26-step canonical sequence, cost model,
  invariants, recommended next investments.

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
