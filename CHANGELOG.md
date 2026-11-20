# Changelog

All notable changes to the Vis Android SDK. This project follows Semantic Versioning.

## [1.0.3] — 2026-11-20

- Selfie framing: the recorded clip is what the screen shows, centred on the circle. Before, on devices whose camera stream is not exactly 1280×720 (most front cameras deliver 4:3), the clip was read from one edge of the frame, so part of the picture on one side was cut off and the face sat off-centre in the clip. The camera preview and the analysis stream are now bound to the screen's viewport (one field of view on every device), both ask for 16:9, and the clip window is a 9:16 crop around the drawn circle, scaled to 720×1280.
- The positioning check uses the circle actually drawn on the screen: recording starts only when the face is inside it, so the face is centred in the clip. The face-size minimum (224 px) is measured in the clip.
- Fewer "frame gap exceeds 250 ms" failures on slow phones: the video encoder starts while the user is positioning instead of on the first recorded frame; the camera keeps at least 15 frames per second in dim light; the per-frame video copy and the face-detection copy no longer make per-pixel buffer calls or allocate per frame; the camera thread runs above normal priority. A clip that still has a frame gap is recorded again (shown as "Find better lighting") instead of ending the verification.
- Smaller selfie clips: the encoder uses constant bit rate (2 Mbit/s) where the device supports it; the default variable mode overshot to 3–5 MB per clip.

## [1.0.2] — 2026-11-19

- `minSdk` lowered from 26 to 24: the SDK now runs on Android 7.0 and 7.1. Partner apps do not need core library desugaring.

## [1.0.1] — 2026-11-18

- Documentation wording.

## [1.0.0] — 2026-11-18

First public release.

- Verification flow: consent, bind, challenge, capture and upload, status, delivered as one typed `VerifyResult` through the Activity Result API.
- Document capture with on-device quality gates (sharpness, glare, framing, steadiness) and an MRZ pre-check with ICAO 9303 check digits; passport or ID card, or a session whose document photos the partner's backend sends; the SDK records only the selfie; the user types the document number and birth date, and they are compared with the document.
- Liveness selfie recorded as one continuous clip that follows the server's action timeline, including the colour flash, with guidance, an intro screen and a help sheet.
- Play Integrity attestation with a per-attempt hardware-backed Keystore key; graded mode for devices that cannot attest.
- DPoP-bound transport with certificate pinning, nonce handling and bounded retries.
- Typed results: `Approved`, `Rejected(RejectReason)`, `Review`, `Timeout`, `Cancelled`, `Failed(SdkError)`; an analytics event stream with no images, scores or tokens.
- Languages: Uzbek, Russian, English. Theming through `VisTheme` and `VisTypography`.
- Explicit public API; consumer R8 rules included; `Vis.sdkVersion`.
- Diagnostics are silent unless enabled with `adb shell setprop log.tag.VisSDK DEBUG`.
- Published on Maven Central as `uz.vis:vis-sdk-android`.
