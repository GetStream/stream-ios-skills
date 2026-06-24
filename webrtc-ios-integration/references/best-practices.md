# Best Practices

## Threading

WebRTC public API calls are designed around a single client thread, but callbacks
and observer events are not guaranteed to arrive on that thread.

Default stance:

- Treat every WebRTC delegate/completion callback as non-main-thread.
- Dispatch UI changes to the main queue.
- Avoid calling WebRTC APIs directly from callbacks when reentrancy or thread ownership is unclear; schedule work onto the app's client queue.
- Use `RTCDispatcher` queues for shared platform resources: main, capture session, audio session, and network monitor.

`ARDAppClient` comments explicitly note that `RTCPeerConnectionDelegate` and SDP
callbacks occur on non-main threads and dispatch to main as needed.

## Threading Architecture

Recommended app shape:

- Use one call coordinator plus one serialized Swift actor or queue per
  `RTCPeerConnection`.
- Keep publisher and subscriber PeerConnection state separate unless a call-level
  transition requires coordination.
- Preserve SDP-before-ICE ordering at the app boundary.
- Route shared platform work to the right owner: audio through
  `RTCDispatcher.dispatchAsync(on: .typeAudioSession, block:)`, capture through
  `.typeCaptureSession`, render/view-model updates through `MainActor`, and
  stats/log work through a diagnostics queue.
- Keep WebRTC, AudioEngine, audio, render, and logging callbacks fast; hop before
  touching product state or calling WebRTC again.

Do not expose WebRTC's internal signaling/worker/network threads as app
architecture. The app owns ordering at its API boundary; WebRTC handles its
internal dispatch after calls enter the Obj-C API.

For Swift, prefer an actor for product state and a small queue wrapper when an
SDK call must happen from a specific GCD queue. WebRTC Obj-C objects are not
documented as `Sendable`; keep ownership on the relevant call or PeerConnection
actor and pass simple values across concurrency boundaries.

For detailed actor/queue layout, publisher/subscriber ownership, callback
handoffs, and race examples, read `threading-swift.md`.

## Swift Bridging

When advising Swift callers:

- Wrap callback APIs with `async` functions at app boundaries, not inside WebRTC internals.
- Resume continuations exactly once from completion handlers.
- Use `@MainActor` only for UI state, renderer ownership, and view-model updates.
- Keep PeerConnection operations serialized through a call/session actor or queue.
- Do not assume WebRTC objects are `Sendable`.
- Avoid holding `RTCAudioBuffer` beyond the audio processing callback; its valid scope is only inside the callback.
- Prefer Swift snippets in user-facing answers, with Obj-C selector names only when explaining the underlying SDK header.

For delegate methods, bridge into Swift state like:

- Receive callback.
- Hop to the session actor/queue for call state.
- Hop to `MainActor` for UI only.

## Lifecycle and Ownership

Prefer explicit ownership:

- Call/session object owns `RTCPeerConnection`, local tracks, capturers, diagnostics, signaling channel, and timers.
- View/controller owns render views and attaches/detaches renderers.
- Audio coordinator owns `RTCAudioSession` policy and CallKit interactions.
- Signaling adapter owns backend message transport and hands ordered SDP/ICE work to the call queue.

Avoid retain cycles:

- Use weak self in timers and async completion handlers.
- Invalidate timers before releasing.
- Remove renderers before replacing tracks.

Cleanup should be idempotent.

## Unified Plan

Use Unified Plan for new work:

- `RTCConfiguration.sdpSemantics = RTCSdpSemanticsUnifiedPlan`
- `addTrack:streamIds:` or transceiver APIs.
- `transceivers`, `senders`, and `receivers` for media.
- `didAddReceiver` and `didStartReceivingOnTransceiver` for remote media.

Avoid deprecated Plan B paths in new integrations:

- `addStream`
- `removeStream`
- `senderWithKind:streamId:`
- stream-only remote media callbacks as the primary design.

## Codec Selection

Default factories:

- `RTCDefaultVideoEncoderFactory`
- `RTCDefaultVideoDecoderFactory`

`RTCDefaultVideoEncoderFactory` exposes `supportedCodecs` and `preferredCodec`.
AppRTCMobile sets `preferredCodec` from settings before creating the factory.

If using custom codecs, implement custom `RTCVideoEncoderFactory` and
`RTCVideoDecoderFactory`.

Do not force codecs by SDP string surgery unless there is no supported API for
the required behavior and the tradeoff is explicit.

## Fork-Specific Rendering Tuning

If `RTCPeerConnectionFactory+Stream.h` is available, `frameBufferPolicy` can tune
decoded frame buffer bridging.

Best practice:

- Set it before call start when possible.
- Keep one owner responsible for changing it.
- Expect mixed frame formats if it changes mid-call.
- Treat NV12 conversion/copy policies as performance-sensitive; profile before assuming they help.
- Fall back to default behavior if renderer or conversion paths fail.

## RTP and Bitrate Tuning

Use sender parameters for send-side tuning:

- Read `sender.parameters`.
- Modify `RTCRtpEncodingParameters` entries.
- Set `maxBitrateBps`, `minBitrateBps`, `maxFramerate`, `numTemporalLayers`, `scaleResolutionDownBy`, `scalabilityMode`, or `isActive` as needed.
- Assign back through `setParameters:`.

`ARDAppClient` sets max video bitrate by iterating senders, finding the video
sender, and updating each encoding's `maxBitrateBps`.

Use `setBweMinBitrateBps:currentBitrateBps:maxBitrateBps:` only when global BWE
control is truly needed and values are sane; tests assert invalid ranges fail.

## Field Trials and Metrics

Field trials and metrics are process-level concerns:

- `RTCInitFieldTrialDictionary` must be called before any other WebRTC call and is deprecated in favor of passing field trials when building `RTCPeerConnectionFactory`.
- `RTCEnableMetrics` must be called before any other WebRTC call.
- `RTCGetAndResetMetrics` gets and clears native histograms.

Do not scatter these calls through feature code. Centralize initialization.

## Fork-Specific Audio Controls

If `RTCAudioDeviceModuleTypeAudioEngine` is available, use it only for products
that need AVAudioEngine-level behavior.

Best practice:

- Keep device selection and engine policy in one audio coordinator.
- Test voice processing, AGC, mute mode, stereo playout, ducking, interruptions, and route changes on hardware.
- Remember that `prefersStereoPlayout` disables voice processing while enabled.
- Prefer observer callbacks for engine lifecycle integration instead of reaching into internal audio objects.
- Do not rely on AudioEngine device lists for production iOS route selection; use `RTCAudioSession`/`AVAudioSession` route policy.

## Data Channels

Use `RTCDataChannelConfiguration` deliberately:

- `isOrdered`: ordered delivery.
- `maxPacketLifeTime`: time-limited unreliable delivery.
- `maxRetransmits`: retransmission-limited unreliable delivery.
- `isNegotiated` and `channelId`: externally negotiated channels.
- `protocol`: app-defined sub-protocol.

Send with `RTCDataBuffer` and observe `readyState` and `bufferedAmount`.

Backpressure matters: do not enqueue unbounded app data when `bufferedAmount`
grows.

## Security and Privacy

Use WebRTC defaults unless product security requirements say otherwise.

Relevant types:

- `RTCCertificate`: generated or provided DTLS certificate material.
- `RTCCryptoOptions`: SRTP crypto suites, encrypted RTP header extensions, and SFrame requirement.
- `RTCFrameCryptor` and `RTCFrameCryptorKeyProvider`: frame-level encryption for senders/receivers.

Be careful with:

- `srtpEnableAes128Sha1_32CryptoCipher`: header notes call it potentially insecure.
- `sframeRequireFrameEncryption`: senders/receivers must have frame encryptors/decryptors attached before packets can be sent/received.
- Persisting private key material from `RTCCertificate`.
- Logging SDP, ICE candidates, TURN credentials, room IDs, device details, or user identifiers.

## Permissions

The app owns permission flow and privacy copy. Before starting capture:

- Check/request camera permission.
- Check/request microphone permission.
- Ensure Info.plist strings match product language.

Do not start capture first and rely on implicit system behavior.

## Error Handling

Never ignore completion-handler errors from:

- SDP creation.
- Setting local/remote descriptions.
- Adding ICE candidates.
- Starting capture.
- Configuring audio session.
- Starting diagnostics.

Map errors into product call state and logs. Close or retry according to explicit
product policy.
