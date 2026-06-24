# Repo Guide

Use this file to choose what to read in `~/workspace/webrtc/src` before
answering WebRTC iOS integration questions.

The app integration examples should be Swift-first. Use Obj-C headers as the
exported SDK source of truth, then translate names through the Swift importer.
For exact spelling, inspect the generated WebRTC module interface or
autocomplete in the consuming app.

## Orientation

Read:

- `README.md`
- `sdk/objc/README.md`
- `native-api.md`
- `docs/native-code/ios/README.md`
- `api/g3doc/threading_design.md`
- `pc/g3doc/peer_connection.md`

Skip build/package/release pipeline files unless the user asks for them.

## Public API Boundary

Read:

- `sdk/objc/api/peerconnection/RTCPeerConnectionFactory.h`
- `sdk/objc/api/peerconnection/RTCPeerConnectionFactoryBuilder.h`
- `sdk/objc/api/peerconnection/RTCPeerConnectionFactoryBuilder+DefaultComponents.h`
- `sdk/objc/api/peerconnection/RTCPeerConnection.h`
- `sdk/objc/api/peerconnection/RTCConfiguration.h`
- `sdk/objc/api/peerconnection/RTCSessionDescription.h`
- `sdk/objc/api/peerconnection/RTCIceCandidate.h`
- `sdk/objc/api/peerconnection/RTCIceServer.h`

Avoid:

- `+Private.h` files for app integration.
- C++ internals under `pc/`, `p2p/`, or `modules/` unless modifying WebRTC itself.

## Fork-Specific Integration APIs

Read:

- `../issues.md`
- `webrtc.gni` for `stream_enable_rendering_backend` availability context only.
- `sdk/objc/api/peerconnection/RTCPeerConnectionFactory+Stream.h`
- `sdk/objc/api/peerconnection/RTCAudioDeviceModule.h`
- `sdk/objc/api/peerconnection/RTCAudioDeviceModule.mm`
- `sdk/objc/api/peerconnection/RTCPeerConnectionFactory.h`
- `sdk/objc/api/peerconnection/RTCPeerConnectionFactory.mm`
- `modules/audio_device/audio_engine_device.h`
- `modules/audio_device/audio_engine_device.mm`
- `sdk/objc/native/src/objc_frame_buffer_stream.mm`
- `sdk/objc/unittests/RTCPeerConnectionFactoryAudioEngine_xctest.mm`

Use this section for integration behavior only. Do not drift into
package/build/release advice unless the user asks.

## PeerConnection and Signaling Flow

Read:

- `examples/objc/AppRTCMobile/ARDAppClient.m`
- `examples/objc/AppRTCMobile/ARDAppClient.h`
- `examples/objc/AppRTCMobile/ARDSignalingMessage.m`
- `examples/objc/AppRTCMobile/ARDWebSocketChannel.m`
- `examples/objc/AppRTCMobile/RTCIceCandidate+JSON.m`
- `examples/objc/AppRTCMobile/RTCSessionDescription+JSON.m`
- `examples/objc/AppRTCMobile/RTCIceServer+JSON.m`

Key areas in `ARDAppClient.m`:

- Factory creation and TURN fetch in `connectToRoomWithId:settings:isLoopback:`.
- Delegate callback threading comments.
- `startSignalingIfReady`.
- `processSignalingMessage:`.
- `sendSignalingMessage:`.
- `createMediaSenders`.
- `createLocalVideoTrack`.
- `disconnect`.

## Media Capture

Read:

- `sdk/objc/components/capturer/RTCCameraVideoCapturer.h`
- `sdk/objc/components/capturer/RTCFileVideoCapturer.h`
- `sdk/objc/base/RTCVideoCapturer.h`
- `sdk/objc/api/peerconnection/RTCVideoSource.h`
- `sdk/objc/api/peerconnection/RTCVideoTrack.h`
- `sdk/objc/components/video_frame_buffer/RTCCVPixelBuffer.h`
- `examples/objc/AppRTCMobile/ARDCaptureController.m`
- `examples/objc/AppRTCMobile/ARDExternalSampleCapturer.m`
- `examples/objc/AppRTCMobile/ios/ARDFileCaptureController.m`
- `examples/objc/AppRTCMobile/ios/broadcast_extension/ARDBroadcastSampleHandler.m`

## Rendering and UI

Read:

- `sdk/objc/base/RTCVideoRenderer.h`
- `sdk/objc/components/renderer/metal/RTCMTLVideoView.h`
- `sdk/objc/components/renderer/pip/RTCPictureInPictureVideoRenderer.h`
- `sdk/objc/helpers/RTCCameraPreviewView.h`
- `examples/objc/AppRTCMobile/ios/ARDVideoCallView.m`
- `examples/objc/AppRTCMobile/ios/ARDVideoCallViewController.m`

## Audio Session

Read:

- `sdk/objc/components/audio/RTCAudioSession.h`
- `sdk/objc/components/audio/RTCAudioSessionConfiguration.h`
- `sdk/objc/components/audio/RTCAudioSessionConfiguration.m`
- `sdk/objc/helpers/RTCDispatcher.h`
- `sdk/objc/components/audio/RTCAudioProcessingConfig.h`
- `sdk/objc/components/audio/RTCAudioCustomProcessingDelegate.h`
- `sdk/objc/components/audio/RTCDefaultAudioProcessingModule.h`
- `sdk/objc/api/peerconnection/RTCAudioDeviceModule.h`
- `sdk/objc/api/peerconnection/RTCAudioDeviceModule.mm`
- `modules/audio_device/audio_engine_device.mm`
- `sdk/objc/unittests/RTCAudioSessionTest.mm`
- `sdk/objc/unittests/RTCPeerConnectionFactoryAudioEngine_xctest.mm`
- `examples/objc/AppRTCMobile/ios/ARDMainViewController.m`
- `examples/objc/AppRTCMobile/ios/ARDVideoCallViewController.m`

Key areas:

- `RTCAudioSession.h`: delegate threading, CallKit activation delegate, manual audio.
- `RTCAudioSession.mm`: activation count, lock checks, route/interruption notifications, `canPlayOrRecord`.
- `RTCAudioSessionConfiguration.m`: `webRTCConfiguration` defaults and mutable global behavior.
- `RTCAudioDeviceModule.h/.mm`: AudioEngine-only Obj-C surface, observer callbacks, no-op/default behavior on non-AudioEngine modules.
- `audio_engine_device.mm`: manual rendering, ducking, mute mode, voice processing, AGC, stereo, route/interruption reactions.
- `ARDMainViewController.m`: sample manual audio setup and playback coordination.
- `ARDVideoCallViewController.m`: sample speaker override and playout glitch callback.

## Codecs and RTP Tuning

Read:

- `sdk/objc/components/video_codec/RTCDefaultVideoEncoderFactory.h`
- `sdk/objc/components/video_codec/RTCDefaultVideoDecoderFactory.h`
- `sdk/objc/api/peerconnection/RTCRtpSender.h`
- `sdk/objc/api/peerconnection/RTCRtpParameters.h`
- `sdk/objc/api/peerconnection/RTCRtpEncodingParameters.h`
- `sdk/objc/api/peerconnection/RTCRtpCodecParameters.h`
- `sdk/objc/api/peerconnection/RTCRtpTransceiver.h`
- `examples/objc/AppRTCMobile/ARDSettingsModel.m`
- `examples/objc/AppRTCMobile/ios/RTCVideoCodecInfo+HumanReadable.m`

## Threading

Read:

- `sdk/objc/helpers/RTCDispatcher.h`
- `api/g3doc/threading_design.md`
- `g3doc/implementation_basics.md`
- `pc/g3doc/peer_connection.md`
- Threading comments in `examples/objc/AppRTCMobile/ARDAppClient.m`
- Dispatch examples in `examples/objc/AppRTCMobile/ios/ARDVideoCallViewController.m`

Use public online docs as a sanity check when giving general architecture
guidance, especially `https://webrtc.github.io/webrtc-org/native-code/native-apis/`
and `https://webrtc.googlesource.com/src/+/HEAD/api/g3doc/threading_design.md`.
Prefer the local checkout for exact current behavior.

Swift mapping examples:

- `RTCDispatcherTypeAudioSession` usually imports as `.typeAudioSession`.
- `dispatchAsyncOnType:block:` usually imports as `RTCDispatcher.dispatchAsync(on:block:)`.
- Completion-handler Obj-C APIs usually import as trailing closures or throwing
  calls when they use `NSError **`; verify before showing final code.

## Data Channels

Read:

- `sdk/objc/api/peerconnection/RTCDataChannel.h`
- `sdk/objc/api/peerconnection/RTCDataChannelConfiguration.h`
- `pc/g3doc/sctp_transport.md`

## Security and Crypto

Read:

- `sdk/objc/api/peerconnection/RTCCertificate.h`
- `sdk/objc/api/peerconnection/RTCCryptoOptions.h`
- `sdk/objc/api/peerconnection/RTCFrameCryptor.h`
- `sdk/objc/api/peerconnection/RTCFrameCryptorKeyProvider.h`
- `sdk/objc/api/peerconnection/RTCSSLAdapter.h`
- `pc/g3doc/srtp.md`
- `pc/g3doc/dtls_transport.md`

## Diagnostics

Read:

- `docs/native-code/logging.md`
- `sdk/objc/base/RTCLogging.h`
- `sdk/objc/api/peerconnection/RTCFileLogger.h`
- `sdk/objc/api/logging/RTCCallbackLogger.h`
- `sdk/objc/api/peerconnection/RTCStatisticsReport.h`
- `sdk/objc/api/peerconnection/RTCMetrics.h`
- `sdk/objc/api/peerconnection/RTCTracing.h`
- Stats/log setup in `examples/objc/AppRTCMobile/ARDAppClient.m`
- Stats UI in `examples/objc/AppRTCMobile/ios/ARDStatsView.m`
- Stats conversion in `examples/objc/AppRTCMobile/ARDStatsBuilder.m`

## Tests

Read:

- `sdk/objc/unittests/RTCPeerConnectionTest.mm`
- `sdk/objc/unittests/RTCPeerConnectionFactory_xctest.m`
- `sdk/objc/unittests/RTCPeerConnectionFactoryBuilderTest.mm`
- `sdk/objc/unittests/RTCAudioSessionTest.mm`
- `sdk/objc/unittests/RTCCameraVideoCapturerTests.mm`
- `sdk/objc/unittests/RTCDataChannelConfigurationTest.mm`
- `sdk/objc/unittests/RTCCertificateTest.mm`
- `sdk/objc/unittests/RTCMTLVideoView_xctest.m`
- `sdk/objc/unittests/RTCCallbackLogger_xctest.m`
- `examples/objc/AppRTCMobile/tests/ARDAppClient_xctest.mm`

## Agent Response Checklist

Before answering:

- Name which repo files informed the answer.
- State whether advice is public API, sample-app pattern, or WebRTC internal.
- If suggesting app code, use Swift first, Unified Plan, and completion-handler APIs.
- If touching UI/audio, include the required dispatch/locking behavior.
- If troubleshooting, suggest the narrowest verification: callback state, stats, logs, or an existing XCTest/sample flow.
- If the user asks about packaging/build/release, explicitly switch scope and inspect those files first.
