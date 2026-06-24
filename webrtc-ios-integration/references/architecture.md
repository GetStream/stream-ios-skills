# WebRTC iOS Architecture

## Scope

This reference is for using WebRTC from an iOS app through the Objective-C SDK
in `~/workspace/webrtc`. It is not a guide for packaging, publishing, or release
artifacts.

## Repo Landmarks

- `docs/native-code/ios/README.md`: upstream iOS development and app usage notes.
- `sdk/objc/README.md`: Obj-C SDK organization.
- `native-api.md`: public native API boundary. Prefer `api/`; legacy/internal paths may change incompatibly.
- `api/g3doc/threading_design.md`: API threading model and callback caveats.
- `pc/g3doc/peer_connection.md`: C++ PeerConnection architecture.
- `sdk/objc/api/peerconnection`: public Obj-C wrappers for PeerConnection, configuration, SDP, ICE, RTP, stats, certificates, crypto, and data channels.
- `sdk/objc/base`: base protocols and value types such as `RTCVideoRenderer`, `RTCVideoCapturer`, `RTCVideoFrame`, and logging.
- `sdk/objc/components`: iOS/macOS platform components for audio session, camera/file capture, renderers, codecs, and networking.
- `examples/objc/AppRTCMobile`: practical sample for signaling, call lifecycle, capture, rendering, stats, and audio routing.
- `examples/objcnativeapi`: Obj-C to native C++ bridge example.

## Public API Boundary

For Swift app integration, import the WebRTC module and use the Swift names for
exported Objective-C SDK types. Inspect the Obj-C headers when exact behavior or
Swift importer spelling matters.

Public surfaces to prefer:

- Factory: `RTCPeerConnectionFactory`, `RTCPeerConnectionFactoryBuilder`.
- Connection: `RTCPeerConnection`, `RTCPeerConnectionDelegate`, `RTCConfiguration`.
- Signaling values: `RTCSessionDescription`, `RTCIceCandidate`, `RTCIceServer`.
- Media: `RTCAudioSource`, `RTCAudioTrack`, `RTCVideoSource`, `RTCVideoTrack`, `RTCRtpSender`, `RTCRtpReceiver`, `RTCRtpTransceiver`.
- Capture and rendering: `RTCCameraVideoCapturer`, `RTCFileVideoCapturer`, `RTCVideoCapturer`, `RTCMTLVideoView`, `RTCPictureInPictureVideoRenderer`.
- Audio: `RTCAudioSession`, `RTCAudioSessionConfiguration`, `RTCAudioDeviceModule`, audio processing types.
- Diagnostics: `RTCFileLogger`, `RTCCallbackLogger`, `RTCStatisticsReport`, `RTCMetrics`, RTC event logs, AEC dump.
- Security: `RTCCertificate`, `RTCCryptoOptions`, `RTCFrameCryptor`, `RTCFrameCryptorKeyProvider`.
- Data: `RTCDataChannel`, `RTCDataBuffer`, `RTCDataChannelConfiguration`.

Avoid private categories such as `+Private.h` unless modifying WebRTC itself.

## Fork-Specific Integration Surfaces

This checkout includes GetStream fork extensions that may or may not be present
in a consumed binary depending on how the framework was built.

AudioEngine ADM:

- `RTCAudioDeviceModuleTypeAudioEngine` selects an AVAudioEngine-backed audio device module.
- `RTCAudioDeviceModule` exposes device lists, output/input selection, engine lifecycle observer callbacks, manual rendering, ducking, mute mode, voice-processing controls, AGC, and stereo playout controls.
- Default factory paths use the platform default ADM. Use AudioEngine APIs only when product audio behavior requires low-level engine control, and verify the framework was built with those headers.
- On iOS, route policy is still primarily `RTCAudioSession`/`AVAudioSession` driven; do not treat AudioEngine device lists as production route selection.

Stream rendering backend:

- `RTCPeerConnectionFactory+Stream.h` adds `frameBufferPolicy`.
- Policies control whether decoded frames remain I420 or bridge/wrap/copy/convert to NV12.
- The policy is evaluated per frame; changing it mid-call can produce mixed buffer formats for in-flight frames.
- Property access is not synchronized; set it from one owner/thread when consistency matters.

Before recommending these APIs, verify the headers exist in the app's linked
framework.

## Obj-C SDK Organization

The Obj-C SDK wraps the C++ PeerConnection API and platform components:

- `api/`: wrappers around C++ APIs for creating/configuring peer connections and related objects.
- `base/`: shared protocols/classes used by wrappers and platform components.
- `components/`: iOS/macOS audio, capture, rendering, codec, network, and frame-buffer implementations.
- `helpers/`: Cocoa helper utilities such as `RTCDispatcher`.
- `native/`: adapters between Obj-C components and C++ APIs.
- `unittests/`: XCTest coverage for Obj-C SDK behavior.

## PeerConnection Model

`RTCPeerConnectionFactory` creates sources, tracks, streams, and peer
connections. The C++ docs recommend avoiding multiple factories unless needed
because factories own shared resources.

`RTCPeerConnection` is the call object. It owns the negotiation, RTP
sender/receiver/transceiver list, data channels, ICE/DTLS transports, call
state, and stats collection.

Create a peer connection with:

- `RTCConfiguration`: ICE servers, transport policy, SDP semantics, crypto options, candidate policies, and network tuning.
- `RTCMediaConstraints`: legacy-style constraints used by parts of the Obj-C API.
- `RTCPeerConnectionDelegate`: event sink for signaling state, ICE, connection state, candidates, receivers, transceivers, and data channels.

Prefer Unified Plan:

- Set `RTCConfiguration.sdpSemantics = RTCSdpSemanticsUnifiedPlan`.
- Use `addTrack:streamIds:` or transceiver APIs.
- Do not use Plan B-only stream APIs (`addStream`, `removeStream`, `senderWithKind`) in new code.
- Use `didAddReceiver` or `didStartReceivingOnTransceiver` for remote media.

## App-Owned Responsibilities

WebRTC does not provide production signaling. The app must own:

- Room/session identity and authentication.
- SDP and ICE message transport.
- TURN/STUN provisioning.
- Permission prompts and Info.plist usage strings.
- UI state and main-thread dispatch.
- App-level call queue/actor ownership and callback handoffs.
- Audio route policy and CallKit coordination.
- Call lifecycle, reconnect policy, and cleanup.
- Product logging, metrics upload, and privacy policy.

## Sample App Caveats

`AppRTCMobile` is valuable for flow, but it is sample architecture:

- `ARDAppClient` combines room server, TURN fetch, signaling, PeerConnection, media, stats, and error handling.
- `ARDWebSocketChannel` uses AppRTC-specific WebSocket/REST semantics.
- Sample settings, room names, and loopback paths should not become app-level dependencies.

Copy flows, not structure.
