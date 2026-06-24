---
name: triage-gh-issue
description: Triage Stream iOS customer issue reports from a GitHub issue URL. Validate gh CLI access and authentication, confirm whether the current checkout matches the issue repository or fall back to remote-repo inspection, read the full issue thread before assessing, classify configuration vs SDK code issues, inspect Stream iOS docs or repository evidence, and draft a ready-to-send customer reply.
---

# Triage GH Issue

## Overview

Use this skill to triage a customer-reported GitHub issue for the Stream iOS
Video SDK. The skill reads the entire issue thread before assessing anything,
decides whether the report is most likely a configuration problem, an SDK issue,
or still missing evidence, then returns an internal assessment plus a
customer-facing reply.

The issue URL is the primary input. Accept GitHub issue URLs and issue-comment
URLs only.

## Portability

Keep this skill portable across Codex, Cursor, and Claude:
- keep `SKILL.md` as the workflow contract
- treat `agents/openai.yaml` as optional Codex metadata
- use generic wording such as local repo mode and remote repo mode instead of
  host-specific features
- prefer live GitHub and live docs inspection instead of bundled snapshots

## Prerequisites

Expect these capabilities before starting:
- `gh` installed locally
- `gh` authenticated for the GitHub host that owns the issue
- access to the issue repository through `gh`
- ability to browse `https://getstream.io/video/docs/ios/`

If `gh` is unavailable or unauthenticated, stop immediately and return a blocker
message instead of attempting triage.

## Required Inputs

Collect these inputs up front:
- GitHub issue URL or issue-comment URL
- current working directory
- optional extra context from the user if they already know the SDK checkout or
  suspect a specific subsystem

Reject these as out of scope:
- pull request URLs
- discussion URLs
- non-GitHub URLs

## Workflow

### 1. Validate `gh`

Run these commands before any issue or repo analysis:
- `gh --version`
- `gh auth status`

If either command fails:
- do not continue
- return a short blocker message telling the user that `gh` must be installed
  and authenticated first

### 2. Normalize the URL

Accept:
- `https://github.com/<owner>/<repo>/issues/<number>`
- `https://github.com/<owner>/<repo>/issues/<number>#issuecomment-...`

Rules:
- strip any `#issuecomment-...` fragment and treat the parent issue as the unit
  of analysis
- reject `/pull/` and `/discussions/` URLs
- extract `<owner>`, `<repo>`, and `<number>` from the normalized issue URL

### 3. Resolve and validate repo context

Do this before reading the issue thread.

Determine the target repository from the issue URL.

Then determine whether the current working directory is already a checkout of
that same repository:
- if inside a git repo, use `gh repo view --json nameWithOwner` or equivalent
  git remote inspection to identify the current local repo
- compare the local repo to the issue repo

Choose exactly one mode:
- local repo mode: current checkout matches the issue repo
- remote repo mode: current checkout does not match, or there is no local
  checkout

Rules:
- local repo mode is preferred when it is available
- remote repo mode is a first-class fallback, not an error
- explicitly state which mode you used in the final `Assessment`

### 4. Read the full issue thread

Do not assess before you finish this step.

Read:
- the issue title and body
- all issue comments with pagination
- comments in chronological order

Use `gh issue view <url> --json title,body,labels,url,state,author` for issue
metadata and `gh api` with pagination for comments in the target repository.

When the URL includes an issue comment fragment, still read the entire issue
thread, not only the referenced comment.

Before classification, write a short thread summary that captures:
- what the customer reports
- what Stream or other participants already asked
- what new facts later comments add
- whether later comments contradict or refine earlier assumptions

### 5. Extract evidence from the whole thread

Extract and normalize:
- reproduction steps
- SDK version
- device model and iOS version
- feature area or call type
- auth and setup details
- logs, stack traces, and exact error strings
- what the customer already tried
- any contradictions across comments
- evidence gaps

If a later comment corrects earlier information, trust the newer evidence and
note the change.

### 6. Classify after the full-thread read

Classify the issue as one of:
- `configuration`
- `code issue`
- `insufficient evidence`

Use this rubric:

`configuration`
- setup or authentication mistakes
- permissions problems
- incorrect call type or feature wiring
- networking or firewall issues
- ringing, push, or CallKit setup gaps
- behavior already covered by Stream docs

`code issue`
- SDK internals appear to be failing
- logs implicate SDK code paths
- likely regression or crash in the SDK
- behavior appears inconsistent with the documented setup

`insufficient evidence`
- the thread lacks enough facts to support either of the other outcomes after
  reading the full thread

If the evidence is mixed or weak, choose `insufficient evidence` instead of
guessing.

### 7. Investigate by branch

#### Configuration branch

Browse `https://getstream.io/video/docs/ios/` and find the most relevant guide
for the reported setup gap.

Start with these guide families when they fit:
- Client & Authentication
- Joining & Creating Calls
- Troubleshooting
- Networking and Firewall
- Connection Test

Requirements:
- include exact documentation links in the customer reply
- only recommend guides that directly match the issue evidence
- if multiple docs are relevant, keep the reply focused and link only the
  highest-value pages

#### Code issue branch

Investigate using the repo mode selected earlier.

Local repo mode:
- search the local checkout with `rg`
- inspect sources, tests, demo apps, and changelog when relevant
- anchor the investigation on log strings, symbols, feature names, and version
  clues from the thread

Remote repo mode:
- use `gh search code` against the target repo for symbols, error strings, and
  likely files
- use `gh api` against the target repo for issue metadata, comments, repo
  metadata, and targeted file-content lookups once relevant files are found
- treat remote investigation as lower-confidence than a matching local checkout

Rules:
- do not claim an SDK root cause unless the thread evidence plus repo evidence
  support it
- if remote repo evidence is too weak to defend a code-cause explanation, fall
  back to the `insufficient evidence` response

#### Insufficient evidence branch

Ask the customer for the missing facts.

When SDK logs are needed, explicitly request:
- `LogConfig.level = .debug`
- `LogConfig.webRTCLogsEnabled = true` when media or WebRTC behavior may be
  involved
- unredacted logs attached as a `.txt` file

Also request any missing details that matter, such as:
- exact reproduction steps
- SDK version
- iOS version and device
- call type or feature area
- screenshots or stack traces when relevant

## Output Contract

Return exactly these sections.

### Assessment

Include:
- classification
- repo mode used
- concise evidence summary
- confidence level
- next action

Keep this factual and short.

### Reply to customer

Return one ready-to-send message only.

Allowed outcomes:
- point the customer to a specific Stream iOS docs guide
- ask for additional context and logs
- explain the likely SDK issue and how it causes the reported behavior

## Reporting Standards

- Read the whole issue before assessing.
- Later comments can overturn an early assumption.
- Do not speculate when the evidence is weak.
- Prefer exact file paths, symbols, and log strings when they strengthen a code
  issue explanation.
- Prefer exact docs links when recommending documentation.
- If the issue appears to span both setup and SDK behavior, explain the setup
  risk first and only escalate to SDK-cause language when the repository
  evidence supports it.
