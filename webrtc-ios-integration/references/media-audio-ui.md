# Media, Audio, and UI

## Permissions and Capabilities

An iOS app that captures camera/microphone must define:

- `NSCameraUsageDescription`
- `NSMicrophoneUsageDescription`

`AppRTCMobile` also declares `UIBackgroundModes` entries for `audio` and `voip`.
Do not add background modes casually; they must match product behavior and App
Store policy.

## Camera Capture

Use `RTCCameraVideoCapturer` for camera input.

Important API points:

- `captureDevices`: available video capture devices.
- `supportedFormatsForDevice:`: supported formats.
- `preferredOutputPixelFormat`: most efficient output pixel format for the capturer.
- `startCaptureWithDevice:format:fps:completionHandler:`
- `stopCaptureWithCompletionHandler:`

`ARDCaptureController` shows the sample pattern:

- Choose front/back `AVCaptureDevice`.
- Choose the supported format closest to desired resolution.
- Prefer the capturer's preferred pixel format when resolution tie-breaks.
- Cap FPS to a product limit.
- Use the async start/stop methods when callers need completion.

Do not assume simulator camera support. AppRTCMobile uses file capture paths on
simulator.

## File Capture

Use `RTCFileVideoCapturer` for bundled video-file input, especially simulator or
tests.

API:

- `startCapturingFromFileNamed:onError:`
- `stopCapture`

Handle the error block; capture does not start if file reading fails.

## External and Broadcast Capture

For ReplayKit or external samples:

1. Validate the `CMSampleBuffer`.
2. Get the `CVPixelBuffer`.
3. Wrap it with `RTCCVPixelBuffer`.
4. Build an `RTCVideoFrame` with rotation and timestamp.
5. Deliver it through `capturer:didCaptureVideoFrame:`.

`ARDExternalSampleCapturer` ignores invalid sample buffers and only sends
`RPSampleBufferTypeVideo` in the broadcast sample handler.

## Local Preview

For camera preview, AppRTCMobile uses `RTCCameraPreviewView` and assigns the
capturer's `AVCaptureSession`.

On hangup:

- Set preview `captureSession` to nil.
- Stop capture.
- Release capture controller references.

## Remote Rendering

Use `RTCMTLVideoView` as the default renderer for iOS.

Important properties:

- `delegate`: receive video size changes.
- `videoContentMode`: iOS content mode.
- `enabled`: toggle rendering without detaching.
- `rotationOverride`: override frame rotation if needed.
- `isMetalAvailable`: gate Metal renderer use if supporting unusual devices.

Attach/detach renderer through the track:

- `[_remoteVideoTrack addRenderer:view]`
- `[_remoteVideoTrack removeRenderer:view]`
- clear with `[view renderFrame:nil]` when replacing tracks.

Respond to `RTCVideoViewDelegate` size changes by relaying layout to the main/UI
layer.

## Picture in Picture

`RTCPictureInPictureVideoRenderer` is a ready-to-use iOS renderer for WebRTC
tracks.

It conforms to `RTCVideoRenderer` and internally:

- Converts `RTCVideoFrame` to `CMSampleBuffer`.
- Enqueues into `AVSampleBufferDisplayLayer`.
- Exposes `videoGravity`.
- Can resize frames to renderer bounds.

Use it when product UI needs PiP-compatible sample-buffer rendering rather than a
regular Metal view.

## Stream Rendering Backend

Some fork builds expose `RTCPeerConnectionFactory+Stream.h`.

Use `frameBufferPolicy` when the app needs control over Objective-C frame-buffer
bridging, especially NV12-oriented rendering paths.

Policies:

- `RTCFrameBufferPolicyNone`: default behavior.
- `RTCFrameBufferPolicyWrapOnlyExistingNV12`: wrap existing NV12 buffers only.
- `RTCFrameBufferPolicyCopyToNV12`: copy to NV12.
- `RTCFrameBufferPolicyConvertWithPoolToNV12`: convert using a pool.

Set the policy before starting a call when the renderer expects a consistent
frame format. Changing it mid-call can yield a mix of I420 and NV12 frames.

## Audio Session

Use `RTCAudioSession` instead of mutating `AVAudioSession` setters directly.

`RTCAudioSession` is a proxy around the shared `AVAudioSession` with locking,
activation reference counting, manual-audio gating, and weak delegates.

Rules:

- Access the underlying singleton through `[RTCAudioSession sharedInstance]`.
- Call `lockForConfiguration` before setters.
- Always call `unlockForConfiguration`.
- Use `RTCDispatcherTypeAudioSession` for audio-session work; in Swift this
  usually imports as `.typeAudioSession`.
- Add/remove `RTCAudioSessionDelegate` weakly for route/interruption/glitch events.
- Use `setConfiguration:error:` or `setConfiguration:active:error:` for WebRTC configuration.

`RTCAudioSession` coordinates activation counts, WebRTC session counts, and
configuration locking. Calling setters without the lock can fail with
`RTCAudioSessionErrorLockRequired`.

Swift spelling usually imports `RTCDispatcherTypeAudioSession` as
`.typeAudioSession`:

```swift
import AVFoundation
import WebRTC

RTCDispatcher.dispatchAsync(on: .typeAudioSession, block: {
    let session = RTCAudioSession.sharedInstance()
    session.lockForConfiguration()
    defer { session.unlockForConfiguration() }

    do {
        try session.setCategory(
            AVAudioSession.Category.playAndRecord.rawValue,
            mode: AVAudioSession.Mode.voiceChat.rawValue,
            options: [.allowBluetooth, .allowBluetoothA2DP]
        )
    } catch {
        // Surface this to call state/logs; audio configuration failures matter.
    }
})
```

Forks and binary builds can expose slightly different Swift names. Check the
generated WebRTC module interface or autocomplete before giving exact spelling.

## Audio Configuration

`RTCAudioSessionConfiguration.webRTCConfiguration` is a mutable global default.
It initializes category, category options, and mode from the current
`AVAudioSession`, then sets preferred WebRTC audio attributes:

- High-performance sample rate of 48 kHz where available.
- I/O buffer duration tuned for WebRTC's 10 ms audio cadence.
- Mono input/output preference to reduce resources and conversions.

Set the product's desired category, mode, and options explicitly before applying
the configuration. For example, a VoIP app usually decides whether it needs
`PlayAndRecord`, voice chat mode, Bluetooth options, default-to-speaker, mixing,
or ducking based on its own UX and CallKit policy.

Some hardware, such as Bluetooth or wired headsets, may not support preferred
sample rate or channels. Decide whether
`ignoresPreferredAttributeConfigurationErrors` is appropriate for product UX.

## Activation and CallKit

Activation is reference-counted. Pair each successful `setActive:YES` with a
matching `setActive:NO`; do not run generic cleanup deactivation if this
component did not activate the session. Unbalanced calls can leave audio active
or make later deactivation happen too early.

CallKit activation is app-owned:

1. In `provider:didActivateAudioSession:`, forward to
   `[RTCAudioSession.sharedInstance audioSessionDidActivate:session]`.
2. Enable WebRTC audio if manual audio is in use.
3. In `provider:didDeactivateAudioSession:`, forward to
   `[RTCAudioSession.sharedInstance audioSessionDidDeactivate:session]`.
4. Disable or tear down call audio according to product state.

These activation delegate methods update WebRTC's activation count and
interruption state. They do not replace the app's call-state machine.

Swift CallKit hook:

```swift
func provider(_ provider: CXProvider, didActivate audioSession: AVAudioSession) {
    let rtcSession = RTCAudioSession.sharedInstance()
    rtcSession.audioSessionDidActivate(audioSession)
    rtcSession.isAudioEnabled = true
}

func provider(_ provider: CXProvider, didDeactivate audioSession: AVAudioSession) {
    let rtcSession = RTCAudioSession.sharedInstance()
    rtcSession.isAudioEnabled = false
    rtcSession.audioSessionDidDeactivate(audioSession)
}
```

## Manual Audio

`RTCAudioSession` supports `useManualAudio` and `isAudioEnabled`.

Use manual audio when the app needs to delay VoIP audio-unit initialization, for
example to avoid interrupting an `AVPlayer` or to coordinate with CallKit.

Flow:

1. Before creating/connecting call audio, set `useManualAudio = YES`.
2. Keep `isAudioEnabled = NO` until the product is ready for WebRTC to start the
   audio unit.
3. On CallKit activation, user consent, or end of competing playback, set
   `isAudioEnabled = YES`.
4. On hold, pre-connect states, or cleanup, set `isAudioEnabled = NO`.

The effective gate is `canPlayOrRecord = !useManualAudio || isAudioEnabled`.
When it changes to false, WebRTC can stop and uninitialize the audio unit,
stopping both incoming and outgoing audio.

Swift setup:

```swift
let audioSession = RTCAudioSession.sharedInstance()
audioSession.useManualAudio = true
audioSession.isAudioEnabled = false
```

## AudioEngine ADM

Some fork builds expose `RTCAudioDeviceModuleTypeAudioEngine`.

Default factory paths use the platform default ADM, not AudioEngine. AudioEngine
requires explicitly creating the factory with
`RTCAudioDeviceModuleTypeAudioEngine`. Injected Obj-C audio devices are a
separate ADM path and do not become AudioEngine automatically.

Use AudioEngine ADM only when the app needs AVAudioEngine-level integration:

- Observe engine creation/start/stop/release.
- Configure input/output nodes.
- Enable manual rendering mode.
- Control advanced ducking and ducking level.
- Control mute mode, voice processing, AGC, and stereo playout.

Prefer the platform default ADM unless the product needs these controls.
AudioEngine-only Obj-C methods return defaults or no-op on non-AudioEngine
modules, so verify the construction path before relying on them.

AudioEngine observer callbacks run off-main on WebRTC worker context. Keep them
fast, dispatch app work elsewhere, and return `0` unless intentionally aborting
engine setup. A nonzero return can fail an engine transition.

The AudioEngine delegate declares `NS_SWIFT_NAME` annotations, so Swift callbacks
use Swift labels:

```swift
final class AudioEngineObserver: NSObject, RTCAudioDeviceModuleDelegate {
    func audioDeviceModule(
        _ audioDeviceModule: RTCAudioDeviceModule,
        didCreateEngine engine: AVAudioEngine
    ) -> Int {
        // Configure quickly or dispatch elsewhere. Return nonzero only to fail.
        return 0
    }
}
```

AudioEngine timing and interaction notes:

- iOS route selection remains `AVAudioSession`-driven; do not rely on
  `outputDevices`, `inputDevices`, or `trySet*Device` as production route UI.
- Manual rendering uses fixed 10 ms chunks and 48 kHz format; treat it as a
  low-level mode that needs timing tests.
- `prefersStereoPlayout` disables voice processing while enabled.
- Stereo playout can collapse back to mono when input becomes active or muted
  state changes.
- Voice processing and AGC are linked; AGC requires voice processing.
- `voiceProcessingBypassed` is a runtime bypass, not the same as disabling voice
  processing.
- Ducking behavior depends on input being enabled and voice processing being
  active.
- Mute mode changes can affect whether input is muted through voice processing,
  mixer volume, or engine restart behavior.

## Route Changes

For speaker/receiver toggles:

- Dispatch to `RTCDispatcherTypeAudioSession` / Swift `.typeAudioSession`.
- Lock the session.
- Call `overrideOutputAudioPort:error:`.
- Unlock.
- Return completion to UI on the appropriate queue.

AppRTCMobile disables the route button while the route-change operation is in
progress. Treat that as sample UI only: production route UI should observe
actual route-change callbacks and recover if headphones, Bluetooth, Control
Center, or system policy changes the route.

Swift speaker override:

```swift
func setSpeakerEnabled(_ enabled: Bool, completion: @escaping (Result<Void, Error>) -> Void) {
    RTCDispatcher.dispatchAsync(on: .typeAudioSession, block: {
        let session = RTCAudioSession.sharedInstance()
        session.lockForConfiguration()
        defer { session.unlockForConfiguration() }

        do {
            try session.overrideOutputAudioPort(enabled ? .speaker : .none)
            DispatchQueue.main.async { completion(.success(())) }
        } catch {
            DispatchQueue.main.async { completion(.failure(error)) }
        }
    })
}
```

Timing risks:

- Route changes can arrive while an interruption or CallKit transition is in
  progress.
- Speaker override is not the same as selecting a Bluetooth or wired route.
- UI should model route-change errors instead of assuming a button tap changed
  the route.
- Avoid doing route work from WebRTC callbacks; hop to the audio-session queue.

## Interruptions and Glitches

Listen through `RTCAudioSessionDelegate`:

- `audioSessionDidBeginInterruption:`
- `audioSessionDidEndInterruption:shouldResumeSession:`
- `audioSessionDidChangeRoute:reason:previousRoute:`
- `audioSessionMediaServerTerminated:`
- `audioSessionMediaServerReset:`
- `didDetectPlayoutGlitch:`
- start/stop play-or-record events.

Delegate callbacks may arrive on system notification threads or WebRTC threads.
Dispatch app/UI state explicitly. Do not synchronously call locked
`RTCAudioSession` setters from inside a delegate that may have been invoked while
the audio-session lock is held.

Swift delegate methods are imported with Swift labels; keep them as handoff
points:

```swift
func audioSessionDidBeginInterruption(_ session: RTCAudioSession) {
    audioCoordinatorQueue.async { /* mark interrupted */ }
}

func audioSessionDidEndInterruption(
    _ session: RTCAudioSession,
    shouldResumeSession: Bool
) {
    audioCoordinatorQueue.async { /* resume only if call state allows */ }
}
```

Interruption flow:

1. On begin, mark call audio interrupted on the app audio coordinator.
2. Pause route overrides and avoid new audio-session configuration unless needed.
3. On end, use `shouldResumeSession` plus product call state to decide whether to
   reactivate or keep audio disabled.
4. Re-check current route, sample rate, channel count, and AudioEngine processing
   state before declaring audio recovered.

Media-server reset/termination should be treated as a stronger recovery path:
reapply configuration under lock, refresh route/UI state, and restart call audio
only if the call is still active.

## Audio Lifecycle Checklist

For a production call:

1. Configure product `AVAudioSession` policy through `RTCAudioSession` under
   lock before call media starts.
2. If using CallKit/manual audio, set `useManualAudio` and initial
   `isAudioEnabled` before creating active call media.
3. Create the peer connection and audio track.
4. On CallKit activation or app-ready audio state, enable audio and let WebRTC
   start play/record.
5. Handle route, interruption, media-server, and glitch delegates by hopping to
   the audio coordinator.
6. On hangup, disable manual audio if used, stop capture/rendering, close the
   peer connection, remove delegates, and balance activation.
