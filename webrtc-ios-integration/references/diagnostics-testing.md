# Diagnostics, Testing, and Troubleshooting

## Logging

iOS native logs go to `stderr` by default. To capture logs:

- Use `RTCFileLogger` for file logs.
- Use `RTCCallbackLogger` to forward logs to app logging.
- Use `RTCSetMinDebugLogLevel` to set console minimum severity.
- Use `RTCLog`, `RTCLogInfo`, `RTCLogWarning`, and `RTCLogError` in Obj-C code.

`RTCFileLogger`:

- Not threadsafe.
- Rotates logs by max file size.
- Default severity is info.
- `logData` returns data only after logging has stopped.

`RTCCallbackLogger`:

- Not threadsafe.
- Default severity is info.
- Handler is called on the same thread that logs.
- Dispatch out of the callback if logging work can be slow.

Do not log TURN credentials, full SDP, user identifiers, or private network
details unless product privacy policy allows it.

## Stats

Use `RTCPeerConnection` stats APIs:

- `statisticsWithCompletionHandler:` for v2 stats.
- `statisticsForSender:completionHandler:`
- `statisticsForReceiver:completionHandler:`
- Legacy `statsForTrack:statsOutputLevel:completionHandler:` only when needed.

`RTCStatisticsReport` contains `timestamp_us` and `statistics`, a dictionary of
`RTCStatistics` by id.

AppRTCMobile polls stats with a timer and dispatches UI updates to main. In
production, keep stats polling bounded and disable it when not visible or
needed.

## RTC Event Logs, AEC Dump, and Tracing

PeerConnection diagnostics:

- `startRtcEventLogWithFilePath:maxSizeInBytes:`
- `stopRtcEventLog`

Factory audio diagnostics:

- `startAecDumpWithFilePath:maxSizeInBytes:`
- `stopAecDump`

Internal capture:

- `RTCStartInternalCapture(filePath)`
- `RTCStopInternalCapture()`

AppRTCMobile stores diagnostic files in the app documents directory and caps RTC
event logs and AEC dump at 5 MB in the sample.

Start diagnostics only when needed and stop them during cleanup.

## Metrics

Use native histograms when product needs WebRTC metrics:

- Call `RTCEnableMetrics()` before any other WebRTC call.
- Call `RTCGetAndResetMetrics()` to fetch and clear samples.

Centralize metric initialization.

## Useful Tests

Obj-C SDK tests:

- `sdk/objc/unittests/RTCPeerConnectionTest.mm`: configuration mapping, invalid SDP, invalid ICE candidates, bitrate validation.
- `sdk/objc/unittests/RTCPeerConnectionFactory_xctest.m`: factory behavior.
- `sdk/objc/unittests/RTCPeerConnectionFactoryBuilderTest.mm`: builder/default components/injected ADM builder.
- `sdk/objc/unittests/RTCAudioSessionTest.mm`: delegates, weak delegate cleanup, activation behavior.
- `sdk/objc/unittests/RTCCameraVideoCapturerTests.mm`: capture session setup, formats, invalid buffers.
- `sdk/objc/unittests/RTCDataChannelConfigurationTest.mm`: data channel configuration.
- `sdk/objc/unittests/RTCCertificateTest.mm`: certificate generation/handling.
- `sdk/objc/unittests/RTCMTLVideoView_xctest.m`: Metal renderer behavior.
- `sdk/objc/unittests/RTCCallbackLogger_xctest.m`: logger severity and callbacks.
- `sdk/objc/unittests/RTCPeerConnectionFactoryAudioEngine_xctest.mm`: AudioEngine ADM behavior.

Sample app tests:

- `examples/objc/AppRTCMobile/tests/ARDAppClient_xctest.mm`: in-process caller/answerer handshake with mocked room server, TURN client, and signaling.
- `examples/objc/AppRTCMobile/tests/ARDSettingsModel_xctest.mm`: settings behavior.
- `examples/objc/AppRTCMobile/tests/ARDFileCaptureController_xctest.mm`: file capture controller.

## Running iOS Tests

The iOS docs note that WebRTC iOS tests must be deployed as `.app` bundles to a
device or simulator. For gtest-style filters in Xcode, edit the scheme and pass
`--gtest_filter`. With `ios-deploy`, pass launch arguments with `-a`.

For simulator-specific test targets, the docs show GN args using:

`enable_run_ios_unittests_with_xctest=true`

Do not invent a test command. Inspect current GN targets or existing CI workflows
before telling the user what to run.

## Troubleshooting Checklist

No connection:

- Confirm remote SDP is set before applying queued candidates.
- Confirm ICE servers are present and TURN credentials are valid.
- Check `didFailToGatherIceCandidate`.
- Watch `RTCIceConnectionState` and `RTCPeerConnectionState`.
- Inspect candidate-pair change callbacks if implemented.

No remote video:

- Confirm Unified Plan receiver/transceiver callbacks are used.
- Confirm remote `RTCVideoTrack` has a renderer attached.
- Confirm UI work happens on main.
- Confirm remote RTP arrives via stats.

No local video:

- Confirm camera permission and Info.plist string.
- Confirm capture device and format selection.
- Use `RTCFileVideoCapturer` on simulator if camera is unavailable.
- Check capturer start completion error.

No audio or wrong route:

- Confirm microphone permission and Info.plist string.
- Configure through `RTCAudioSession` under lock.
- Dispatch route changes to `RTCDispatcherTypeAudioSession`.
- Check interruptions, media server reset, and playout glitch callbacks.
- Check CallKit activation/deactivation balance.
- If using manual audio, check `useManualAudio`, `isAudioEnabled`, and the expected `canPlayOrRecord` transition.
- If using AudioEngine ADM, confirm the factory was created with `RTCAudioDeviceModuleTypeAudioEngine`.
- Do not rely on AudioEngine input/output device arrays as proof of iOS route selection; inspect the current `AVAudioSession` route.

SDP or ICE errors:

- Use completion-handler APIs and surface errors.
- Avoid deprecated `addIceCandidate:` without completion handler.
- Validate signaling message ordering.

Performance or quality:

- Use stats before changing RTP parameters.
- Tune sender encodings, not SDP strings.
- Prefer hardware-friendly capture sizes/FPS.
- Avoid heavy work in render, audio, or logging callbacks.

## Audio Manual QA

Run audio checks on real devices when product behavior depends on hardware route
or voice processing:

- Start and end calls with receiver, speaker, wired headset, Bluetooth HFP, and Bluetooth A2DP where relevant.
- Toggle speaker while ICE is connecting, while media is active, and immediately after CallKit activation.
- Receive a phone interruption, Siri interruption, and app foreground/background transition; verify recovery follows product state.
- Use CallKit activation/deactivation and confirm activation counts remain balanced across answer, hold, resume, and hangup.
- With manual audio, verify audio does not start before `isAudioEnabled = YES` and stops when set back to `NO`.
- With AudioEngine ADM, verify observer callbacks stay fast and return `0`, route changes recover, manual rendering keeps timing, and observer errors are surfaced.
- Test voice processing, AGC, `voiceProcessingBypassed`, mute modes, ducking, and stereo playout together; stereo preference can disable voice processing and active input can collapse playout to mono.

## Threading Troubleshooting

When callbacks race or state appears stale:

- Confirm all PeerConnection operations enter through the call queue or actor.
- Confirm WebRTC delegate/completion callbacks hop before calling WebRTC again.
- Confirm signaling preserves SDP-before-ICE ordering.
- Cancel timers, stats polling, and queued callbacks on hangup so closed calls are not resurrected.
- Move log upload, stats formatting, and UI work out of WebRTC callbacks.
- In Swift, check that non-`Sendable` WebRTC objects are not freely crossing
  actors/tasks; pass simple values or hop back to the owner actor/queue.
