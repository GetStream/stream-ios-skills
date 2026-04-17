---
name: sdk-release
description: Analyze iOS SDK release diffs between a source and destination ref. Summarize changes, validate changelog coverage, flag regressions, and generate manual QA scenarios.
---

# SDK Release

## Overview

Use this skill to turn a release diff into a release-readiness report for the iOS team. Treat an explicit request to use this skill by name, through a slash command, or through the host's skill UI as permission to delegate to subagents or parallel reviewers when the host supports that capability, because deep parallel analysis is part of this skill's contract.

If the host does not support explicit subagents, keep the same decomposition and run the review passes sequentially. State that limitation in the report instead of silently collapsing the workflow.

## Portability

Keep this skill portable across Codex, Cursor, and Claude:
- keep `SKILL.md` as the source of truth for workflow instructions
- use generic terms such as `subagent`, `parallel reviewer`, and `delegated pass` instead of host-specific tool names
- treat literal invocation strings such as `$sdk-release` as host-specific examples rather than universal syntax
- treat `agents/openai.yaml` as optional Codex UI metadata, not as required instructions
- do not move core workflow guidance into `AGENTS.md`, Cursor rules, or `CLAUDE.md`; those are host-specific memory layers, not the skill contract

## Prerequisites

Expect these capabilities before starting:
- a local Git checkout that contains both refs
- ability to run `git` commands against that checkout
- ability to search the repository for changelogs, release notes, tests, and touched modules

If local Git access is unavailable, say that the full skill cannot run as designed. Either ask for an exported diff and changelog artifacts, or continue with a reduced-confidence review that explicitly calls out the missing evidence sources.

## Required Inputs

Collect these inputs up front:
- source ref: the last shipped branch or tag
- destination ref: the release candidate branch or tag
- repo root
- changelog or release note path if the user already knows it

If the changelog path is not provided, search common locations such as `CHANGELOG.md`, `CHANGELOG`, `ReleaseNotes*`, `docs/releases/`, and package-level changelogs.

Resolve both refs to commit SHAs before analyzing, and keep the human-friendly ref names in the final report.

## Workflow

### 1. Build the diff map locally

Do the first pass yourself before delegating.

Capture:
- `git rev-parse <source>`
- `git rev-parse <destination>`
- `git merge-base <source> <destination>`
- `git diff --stat <source>..<destination>`
- `git diff --name-status <source>..<destination>`
- `git log --oneline --no-merges <source>..<destination>`

Group changed files into:
- public API or module surface
- runtime behavior
- build, package, or dependency changes
- tests
- docs and changelog
- sample or demo app changes

Use this map to decide where delegation will pay off.

### 2. Spawn focused agents

Split work into disjoint questions. Keep the overall synthesis local.

Typical delegation layout:
- Agent 1: changelog or release note validation
- Agent 2 and beyond: subsystem reviewers, one per major module or feature area
- Final parallel agent: regression and test-coverage reviewer
- Optional final parallel agent: manual QA scenario drafter grouped by affected areas

Rules:
- give each agent the exact refs, paths, and ownership scope
- avoid duplicate review coverage unless a second pass is intentional
- do not hand off the critical-path synthesis
- wait only when blocked on a delegated result
- if the diff is small, use fewer agents; if it is broad, split aggressively by subsystem

If you must fall back to a sequential review:
- keep the same scopes as separate passes: changelog, subsystem review, regression and tests, manual QA
- finish one scoped pass at a time and record its evidence before moving to the next pass
- preserve the same final report structure and risk labels
- state that the review was sequential because the host could not run parallel agents

### 3. Validate the changelog

Check that the changelog is updated and accurate.

For each changelog item, answer:
- what code or tests in the diff support this claim
- whether the scope is understated or overstated
- whether the entry omits breaking behavior, migrations, or rollout caveats
- whether the diff contains user-visible or integrator-visible changes missing from the changelog

Treat any of these as findings:
- no changelog update despite meaningful release changes
- changelog entries with no supporting diff evidence
- important changes missing from the changelog
- breaking or risky changes described as routine fixes
- changed defaults, dependency shifts, or migration requirements not called out

### 4. Review for regressions from the source

Compare the destination against the source, not just against the changelog narrative.

Prioritize:
- behavior changes around modified code paths
- deleted or weakened tests
- public API changes or semantic shifts
- error handling, retries, reconnects, and fallback behavior
- threading, async flows, lifecycle, memory, and state retention
- config, entitlement, permissions, or dependency changes
- sample app or integration-code drift that hints at migration impact

For each risk, note whether it is:
- confirmed by code or tests
- strongly suspected from the diff
- uncertain and needs manual verification

Always propose the next fix, mitigation, or verification step.

### 5. Produce manual QA coverage

Turn both the changelog claims and the risky areas into manual QA scenarios.

Every scenario must include:
- feature or area under test
- prerequisites
- reproduction steps
- expected results or pass criteria
- failure signals

Cover:
- each changelog claim that affects runtime behavior or integrators
- each high-risk modified area
- nearby regression surfaces that often break when that area changes

Use [references/release-review-checklist.md](references/release-review-checklist.md) when you need deeper heuristics or a scenario seed list.

## Output Contract

Return exactly these sections.

### Summary of What Changed

Keep this concise. Group the change set by subsystem or release theme. Call out notable API, runtime, dependency, and test changes.

### Potential Regressions and Fixes Needed

List only substantive items. For each item include:
- severity
- area
- what changed
- why it might regress or misalign with the changelog
- evidence from files, commits, or tests
- recommended fix or next action

Separate confirmed mismatches from suspected risks when the evidence is weaker.

### Manual QA Test Scenarios

Group scenarios by feature area or release theme. For every scenario include:
- prerequisites
- steps
- expectations or pass criteria
- failure criteria or suspicious outcomes

Prefer scenarios that directly validate the changelog, then expand to adjacent regression checks around the touched code.

## Reporting Standards

- Use both ref names and resolved SHAs in the final report.
- Call out uncertainty explicitly instead of guessing.
- If the diff is too large, say which modules or files were sampled and why.
- If a changelog file cannot be found, say so and treat it as a release-process gap.
- Keep the report dense with evidence, not long for its own sake.
- Reference specific files, tests, commits, or directories whenever they materially strengthen a claim.
