# Changelog

All notable changes to the Vis Android SDK. This project follows Semantic Versioning.

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
