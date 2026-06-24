# WebRTC iOS Usage Flows

## Minimal Call Flow

Use `examples/objc/AppRTCMobile/ARDAppClient.m` as the canonical flow example.

1. Create a `RTCPeerConnectionFactory`.
2. Fetch or receive ICE servers from the app backend.
3. Join/register with the app's signaling service.
4. Create an `RTCConfiguration`.
5. Set `config.iceServers`.
6. Set `config.sdpSemantics = RTCSdpSemanticsUnifiedPlan`.
7. Create `RTCPeerConnection` with configuration, constraints, and delegate.
8. Create local audio/video tracks and add them with `addTrack:streamIds:`.
9. If caller, create offer, set local description, then send SDP.
10. If callee, wait for remote offer, set remote description, create answer, set local description, then send SDP.
11. Send generated ICE candidates over signaling.
12. Apply remote candidates with `addIceCandidate:completionHandler:`.
13. On hangup, stop capture, detach renderers, stop diagnostics, close the peer connection, and release references.

`ARDAppClient` is a sample convenience layer whose methods should only be called
from the main queue. Do not generalize that to raw WebRTC APIs; lower-level
WebRTC callbacks still need explicit dispatch.

For production, serialize PeerConnection operations through an app-owned call
queue or actor. Let signaling/network callbacks enqueue ordered SDP and ICE work
instead of mutating PeerConnection state directly.

## Factory Setup

Default app integration can use:

- `[[RTCPeerConnectionFactory alloc] init]` for default H264 encoder/decoder factories and default audio device module.
- `initWithEncoderFactory:decoderFactory:` when choosing codec preferences or custom factories.
- `RTCPeerConnectionFactoryBuilder` when injecting native components, field trials, audio device module, or audio processing builders.

Prefer one shared factory for a call subsystem unless isolation is required.

## Configuration

Common `RTCConfiguration` fields for integration:

- `iceServers`: STUN/TURN list from the app backend.
- `iceTransportPolicy`: use relay-only when product policy requires TURN.
- `bundlePolicy`, `rtcpMuxPolicy`, `tcpCandidatePolicy`, `candidateNetworkPolicy`: network behavior.
- `continualGatheringPolicy`: whether to gather once or continually.
- `sdpSemantics`: use Unified Plan.
- `certificate`: optional `RTCCertificate` reuse.
- `cryptoOptions`: advanced SRTP/SFrame policy.
- `turnLoggingId`: correlate TURN allocation requests with backend logs.

Changing STUN/TURN servers or ICE policy via `setConfiguration:` affects the
next gathering phase and can cause the next offer to generate new ICE
credentials. BUNDLE and RTCP multiplexing policies cannot be changed this way.

## Signaling Contract

WebRTC gives values to send; the app owns transport and ordering.

Send over the app signaling channel:

- `RTCSessionDescription` for offer/answer SDP.
- `RTCIceCandidate` for local candidates.
- Candidate removals when needed.
- Hangup/bye message according to the app protocol.

Receive and apply:

- Call `setRemoteDescription:completionHandler:` for offers/answers.
- Call `addIceCandidate:completionHandler:` for candidates.
- Surface completion-handler errors to call state, logs, and UI.

In Swift, prefer the imported completion forms and verify exact names in the
linked WebRTC module:

```swift
peerConnection.setRemoteDescription(remoteSdp) { error in
    // Hop back to the call actor/queue before mutating product state.
}

peerConnection.add(remoteCandidate) { error in
    // Apply only after remote SDP has been accepted.
}
```

Process SDP before queued ICE candidates. `ARDAppClient` queues messages until a
peer connection exists and an offer/answer has been received.

## Offer/Answer

Use completion-handler APIs. In Swift, these usually import as methods with
trailing completion handlers:

- `offer(for:completionHandler:)`
- `answer(for:completionHandler:)`
- `setLocalDescription(_:completionHandler:)`
- `setRemoteDescription(_:completionHandler:)`
- `setLocalDescriptionWithCompletionHandler(_:)`

On create-SDP success:

1. Set the local description.
2. Send the SDP over signaling.
3. Apply sender parameters such as max bitrate if required.

On set-remote-description success as callee:

1. Create an answer.
2. Set local description.
3. Send answer.

Handle errors by closing or recovering according to product state; do not ignore
SDP/candidate completion errors.

## Media Senders

Audio:

- Create `RTCAudioSource` with audio constraints.
- Create `RTCAudioTrack`.
- Add it with `addTrack:streamIds:`.
- Coordinate audio-session policy separately through `RTCAudioSession`; creating
  an audio track is not the same as choosing route, category, CallKit activation,
  or manual-audio timing.

Swift shape:

```swift
let audioSource = factory.audioSource(with: RTCMediaConstraints(
    mandatoryConstraints: nil,
    optionalConstraints: nil
))
let audioTrack = factory.audioTrack(with: audioSource, trackId: "audio0")
peerConnection.add(audioTrack, streamIds: ["stream0"])
```

Video:

- Create `RTCVideoSource`.
- Create a capturer with the source as delegate.
- Create `RTCVideoTrack`.
- Add it with `addTrack:streamIds:`.

For camera:

- `RTCCameraVideoCapturer` delivers frames to `RTCVideoSource`.
- Pick device, format, and FPS based on app settings and supported formats.

For file/simulator:

- `RTCFileVideoCapturer` can feed video from a bundled file.

For broadcast/screen samples:

- Convert valid `CMSampleBuffer` video frames to `RTCCVPixelBuffer`.
- Wrap in `RTCVideoFrame`.
- Call `capturer:didCaptureVideoFrame:` on the delegate.

## Remote Media

With Unified Plan:

- Watch `didStartReceivingOnTransceiver` and `didAddReceiver`.
- Read `transceiver.receiver.track`.
- If it is a video track, attach a renderer.

`AppRTCMobile` also gets the receiver's video track from the video transceiver
immediately after adding local video; the track produces frames after RTP
arrives.

## Data Channels

Create with:

- `dataChannelForLabel:configuration:`
- `RTCDataChannelConfiguration` for ordering, reliability, negotiated mode, channel id, and protocol.

Use:

- `sendData:` with `RTCDataBuffer`.
- `RTCDataChannelDelegate` for state and received buffers.
- `bufferedAmount` / `didChangeBufferedAmount` to avoid unbounded app queues.

## Cleanup

On hangup or fatal error:

- Remove renderers from old tracks.
- Render nil/clear UI views if needed.
- Stop camera/file/broadcast capture.
- Disable manual audio or unwind audio-session activation if this call enabled it.
- Stop AEC dump and RTC event log.
- Close `RTCPeerConnection`.
- Nil strong references to tracks, peer connection, channel, and timers.
- Notify app UI on main.

Make cleanup idempotent. `ARDAppClient` returns early when already disconnected.
