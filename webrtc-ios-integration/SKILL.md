---
name: webrtc-ios-integration
description: Provides repo-grounded architecture, usage flows, and best practices for integrating WebRTC into iOS apps. Use when working on iOS WebRTC, RTCPeerConnection, RTCVideoTrack, RTCDataChannel, RTCAudioSession, camera capture, rendering, signaling, stats, diagnostics, or Swift/Objective-C WebRTC integration.
---

# WebRTC iOS Integration

## Overview

Use this skill for iOS app integration guidance from the WebRTC checkout at
`~/workspace/webrtc`. Consult repo sources first; do not answer from generic
WebRTC memory when the exact Objective-C SDK or sample app behavior matters.
Assume integrator app code is Swift unless the user says otherwise; map advice
back to Obj-C SDK headers only to verify exported API behavior.

Build, packaging, distribution, and release artifact guidance is intentionally
out of scope unless the user asks for it.

## Portability

Keep this skill portable across Codex, Cursor, and Claude:
- keep `SKILL.md` as the workflow contract
- use `references/` for detailed guidance
- avoid host-specific assumptions
- prefer live repo inspection over stale bundled snapshots

## Workflow

1. Identify the user's integration area: PeerConnection, signaling, media, audio, rendering, threading, data channels, security, diagnostics, or tests.
2. Read only the narrowest matching reference file below before advising or editing; add more references only when the task crosses areas.
3. Prefer public Obj-C SDK headers in `sdk/objc/api`, `sdk/objc/base`, and `sdk/objc/components`, then present app-facing examples in Swift.
4. Use `examples/objc/AppRTCMobile` as the practical sample, not as production architecture to copy wholesale.
5. Keep changes minimal and app-owned: WebRTC handles media/transport primitives; the app owns signaling, permissions, lifecycle, threading, and UI state.
6. In answers, name the repo files inspected and label advice as public API, sample-app pattern, fork-specific surface, or WebRTC internal.

## References

- For repo orientation, public API boundaries, and PeerConnection architecture, read `references/architecture.md`.
- For setup, signaling, negotiation, media sender/receiver flows, and cleanup, read `references/usage-flows.md`.
- For camera/file/broadcast capture, renderers, audio session, routing, interruptions, and CallKit/manual audio, read `references/media-audio-ui.md`.
- For threading rules, Swift bridging basics, codecs, RTP tuning, data channels, security, and privacy, read `references/best-practices.md`.
- For detailed Swift concurrency, actors/queues, publisher/subscriber ownership, and callback race guidance, read `references/threading-swift.md`.
- For logs, stats, RTC event logs, AEC dumps, tests, CI validation, and troubleshooting, read `references/diagnostics-testing.md`.
- For exact source files to inspect by task, read `references/repo-guide.md`.

## Hard Rules

- Do not depend on WebRTC internals unless the user is modifying WebRTC itself. Public app integration should use exported WebRTC SDK APIs from Swift.
- Prefer Unified Plan: `RTCConfiguration.sdpSemantics = RTCSdpSemanticsUnifiedPlan`.
- Treat WebRTC callbacks as non-main-thread unless proven otherwise. Dispatch UI and app state updates explicitly.
- Configure `AVAudioSession` through `RTCAudioSession`, using `lockForConfiguration` and `unlockForConfiguration`.
- Keep `RTCPeerConnection` operations serialized on an app-owned call queue or actor; do not call WebRTC APIs directly from delegate callbacks.
- When referencing `RTCDispatcher` from Swift, prefer the imported form, for example `RTCDispatcher.dispatchAsync(on: .typeAudioSession, block: { ... })`; verify exact names in the linked WebRTC module when forks differ.
- Attach video capturers to `RTCVideoSource`; attach renderers to `RTCVideoTrack`.
- Do not call deprecated APIs in new code when a completion-handler or Unified Plan API exists.
- Initialize field trials or metrics before any other WebRTC call if they are needed.
- Do not turn AppRTCMobile's room server, WebSocket channel, or settings model into a production dependency; use it as a flow example.
- Treat fork-specific APIs such as AudioEngine ADM and Stream rendering backend as optional surfaces; verify their headers exist in the built SDK before recommending them.

## Default Advice

For a new iOS integration, guide agents toward this shape:

1. Create one long-lived `RTCPeerConnectionFactory` per call subsystem unless there is a clear reason for more.
2. Create `RTCConfiguration` with ICE servers, Unified Plan, and any crypto/network policy required by the app.
3. Create `RTCPeerConnection` with a delegate, then add tracks or transceivers.
4. Exchange offer/answer SDP and ICE candidates over the app's signaling channel.
5. Feed camera, file, or external frames into an `RTCVideoSource`.
6. Attach remote `RTCVideoTrack` objects to Metal or PiP renderers on the UI side.
7. Route all audio-session changes through `RTCAudioSession`.
8. On hangup, stop capture, detach renderers, stop diagnostics, close the peer connection, and release strong references.
