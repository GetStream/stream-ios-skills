# Release Review Checklist

## Evidence Collection

Collect the smallest set of artifacts that explains the release delta:
- resolved SHAs for source and destination
- merge base
- diff stat and name-status output
- commit list unique to the destination
- changelog or release note candidates
- modified, added, or deleted tests
- package, dependency, or build-system changes

If the diff is large, bucket changed files by module or feature area before reading file-level details.

## Changelog Review Questions

Ask these questions for every changelog entry:
- Which files, commits, or tests support this claim?
- Does the wording overstate or understate the real behavior change?
- Is there a hidden migration, default-value shift, or compatibility caveat?
- Does the diff introduce a user-visible or integrator-visible change that is missing from the changelog?
- Does the changelog call something a fix when the diff actually changes semantics?

Treat missing changelog coverage as a release-process issue even when the code looks correct.

## Regression Hotspots For iOS SDK Releases

Check these areas whenever they are touched directly or indirectly:
- public API, symbol surface, initializers, and defaults
- networking, retries, reconnect, reachability, and offline recovery
- auth, token refresh, session state, and reconnect-after-auth flows
- background and foreground behavior, notifications, and permissions
- persistence, cache formats, migrations, and upgrade paths
- async ordering, cancellation, thread confinement, and main-thread assumptions
- memory ownership, observer lifetimes, delegates, and teardown paths
- package manifests, dependency versions, platform minimums, and build settings
- samples, integration docs, and migration snippets that reveal consumer impact
- tests that were deleted, narrowed, or no longer cover the old edge cases

## Manual QA Scenario Seeds

Generate scenarios from both claimed changes and adjacent regression surfaces:
- fresh integration build with the destination ref
- upgrade from the source release to the destination release
- happy path for every changelog item
- failure path for every risky area touched by the diff
- background and foreground transitions
- network interruption, reconnect, and retry behavior
- permission denied, revoked, or delayed-grant flows
- repeated actions, cancellation, rapid navigation, or teardown races
- cached-data or persisted-state behavior across upgrades
- compatibility with configuration values or defaults changed in the diff

## Scenario Shape

Use this shape for every QA item:
- Area
- Prerequisites
- Steps
- Expectations or pass criteria
- Failure signals

Prefer explicit, reproducible steps over vague advice like "test basic flows."

## Risk Language

Use consistent labels in the report:
- Confirmed mismatch: changelog statement conflicts with the diff or is unsupported.
- Confirmed regression risk: code or tests show an actual break or weakened behavior.
- Suspected regression risk: the diff strongly suggests a problem but evidence is incomplete.
- Manual verification required: the code path is high risk and the diff alone cannot prove safety.
