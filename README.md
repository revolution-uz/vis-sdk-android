# Vis Android SDK

Native Android SDK for Vis identity verification. It runs the whole capture
flow — consent, document scan, liveness selfie — against the Vis platform and
returns one typed result to your app. The SDK never decides a verdict and never
exposes scores: the server decides, the SDK reports.

## Requirements

- Android `minSdk 24` (Android 7.0); built against `compileSdk 35`
- Kotlin (the SDK's screens use Jetpack Compose internally; your app does not need Compose)
- An attempt handoff minted by **your backend** (see below)
- A front camera (optional in the manifest; devices without one finish with `DeviceUnsupported`)

## Installation

The SDK is published on Maven Central, which every Android project already
lists in `settings.gradle.kts`. In your app module:

```kotlin
implementation("uz.vis:vis-sdk-android:1.0.3")
```

## Backend handoff

Your backend starts every verification with `POST /api/sdk/v1/session/start`
using its OAuth client credentials (server to server). **Credentials never
ship inside the app.** The response contains the attempt handoff, which your
backend passes to the app:

| Field | Use |
|---|---|
| `attempt_id` | `VerifyRequest.attemptId` |
| `bootstrap_secret` | `VerifyRequest.bootstrapSecret` (one-shot, consumed by bind) |
| `bootstrap_expires_at` | `VerifyRequest.bootstrapExpiresAt` (ISO-8601; launch before it) |
| `bind_nonce` | `VerifyRequest.bindNonce` |
| `capture_policy_version` | `VerifyRequest.capturePolicyVersion` |
| `app_check` | `VerifyRequest.appCheck`, verbatim (`required`, `graded` or `off`) |
| `documents` | `VerifyRequest.documents`, verbatim (`required` or `off`) |

The handoff is one-shot and short-lived. If the app dies mid-flow there is no
resume: ask your backend for a new attempt.

## Configuration

Call `Vis.configure` once, for example in `Application.onCreate`:

```kotlin
Vis.configure(
    application,
    VisConfig(
        baseUrl = "https://vis.uz",
        pins = setOf("sha256/<leaf>", "sha256/<backup>"),
        playIntegrityCloudProjectNumber = 123456789012L,
        locale = Locale("uz"),
        theme = VisTheme(primary = Color(0xFF0E5468)),
    ),
)
```

| `VisConfig` parameter | Meaning |
|---|---|
| `baseUrl` | Platform origin, `https://...`, no trailing slash. Plain `http` is accepted only for loopback development hosts. |
| `pins` | SPKI certificate pins (`sha256/...`). Supply the leaf and one backup pin in production. |
| `allowUnpinned` | Must be `true` to run with no pins; development servers only. Default `false`. |
| `playIntegrityCloudProjectNumber` | Google Cloud project number that receives Play Integrity verdicts. **Required whenever app check is required** (the default); without it the flow ends with `Failed(Internal)`. |
| `locale` | `uz`, `ru` or `en`; default `uz`. Overrides the system language on SDK screens. |
| `theme` | `VisTheme`: colours, corner radius and `VisTypography` (one `Typeface` per weight). Every token has a default. |
| `processingTimeout` | Seconds (10..600, default 90) to wait for a verdict before `VerifyResult.Timeout`. The attempt keeps processing and your webhook still arrives. |
| `showsConsentScreen` | Show the SDK's own consent screen first. Default `false`: your app obtains biometric consent before launching. |

`Vis.isConfigured` and `Vis.sdkVersion` (the SDK's semantic version) are available at runtime.

## Launching a verification

```kotlin
private val verify = registerForActivityResult(Vis.VerifyContract()) { result ->
    when (result) {
        is VerifyResult.Approved  -> onApproved(result.attemptId)
        is VerifyResult.Rejected  -> showReason(result.reason)
        is VerifyResult.Review    -> showPending()
        is VerifyResult.Timeout   -> showCheckBackLater()
        is VerifyResult.Cancelled -> Unit
        is VerifyResult.Failed    -> showError(result.error)
    }
}

verify.launch(
    VerifyRequest(
        attemptId = handoff.attemptId,
        bootstrapSecret = handoff.bootstrapSecret,
        bootstrapExpiresAt = handoff.bootstrapExpiresAt,
        bindNonce = handoff.bindNonce,
        capturePolicyVersion = handoff.capturePolicyVersion,
        appCheck = handoff.appCheck,
        documents = handoff.documents,
    ),
)
```

`VerifyRequest.toString()` redacts the secret and the nonce.

## Results

| Result | Meaning | What to do |
|---|---|---|
| `Approved(attemptId)` | Verified. | Proceed. |
| `Rejected(reason)` | The person failed a check. | Show a message for the `RejectReason` (below). |
| `Review` | Sent to human review. | Treat as pending, neither pass nor fail. |
| `Timeout` | No verdict within `processingTimeout`. | Tell the user to check back; the webhook still arrives. |
| `Cancelled` | The user backed out. | Nothing. |
| `Failed(error)` | The platform or SDK could not complete. | Never present as a rejection of the user (below). |

### RejectReason

| Reason | Tell the user |
|---|---|
| `LIVENESS_FAILED` | The live check did not pass (head turn, video, more than one face). Try again in good light, alone, facing the camera. |
| `FACE_MISMATCH` | The face did not match the face on the document. |
| `INVALID_DOCUMENT` | The document was not accepted: unreadable, expired, not a valid card, or not the person's card. |
| `DOCUMENT_DATA_MISMATCH` | The typed document number or birth date does not match the document. Type it again. |
| `UNKNOWN` | Generic failure. Categories added later reach older SDKs as `UNKNOWN`, so always handle it. |

The partner webhook carries the same category; decide policy there, not on the device.

### SdkError (inside `Failed`)

| Error | Meaning | Action |
|---|---|---|
| `SessionNotFound` | Attempt unknown or its credentials already consumed. | New attempt from your backend. |
| `SessionExpired` | Bootstrap window or attempt lifetime elapsed. | New attempt from your backend. |
| `DeviceUnsupported(reason)` | No front camera, OS too old, attestation unsupported, retired SDK version. | Another device, or update the app. |
| `AttestationFailed` | The server rejected the device/app attestation. | Your policy decides; check the Play Integrity setup. |
| `CaptureRejected(hint)` | Capture could not satisfy the server, or the last selfie try was missed. Use `reason`, `attempt` and `maxAttempts` rather than parsing `hint`. | New attempt. |
| `RateLimited(retryAfterSeconds)` | Throttled. | Retry after the given seconds. |
| `NetworkUnavailable` | Network down after the SDK's own retries. | Retry when online. |
| `PermissionDenied` | Camera permission denied. | Send the user to system settings. |
| `Internal(detail)` | Unexpected failure; `detail` is safe to log and contains no personal data. | Log and retry. |

### Counting attempts

The SDK retakes the selfie locally, up to 3 recordings per challenge, only when
the user did not do the head turn. Nothing is uploaded for a missed try, so
your app never hears of it. Everything else ends the flow at once, so one
`Rejected`, or one selfie `CaptureRejected` reaching your app, is one finished
attempt. `Cancelled` and technical `Failed` errors are not the user's failure.

## Events

`Vis.events` is a `SharedFlow<SdkEvent>` for analytics. It carries no images, scores or tokens.

```kotlin
scope.launch { Vis.events.collect { event -> track(event) } }
```

- `StepStarted(step)`, `StepCompleted(step, durationMs)`; `FlowStep` is one of `CONSENT`, `BINDING`, `DOCUMENT_CHOICE`, `IDENTITY`, `LIVENESS_INTRO`, `CHALLENGE`, `CAPTURING_DOCUMENT_FRONT`, `CAPTURING_DOCUMENT_BACK`, `CAPTURING_SELFIE`, `UPLOADING`, `PROCESSING`
- `CaptureRetaken(step, gate)`
- `FlowCompleted(outcome)`
- `DocumentTypeChosen(documentType)`
- `LivenessIntroShown`, `LivenessIntroStartTapped(timeOnScreenMs)`, `LivenessIntroBack`
- `LivenessHelpOpened(phase, attempt, turnSide)`, `LivenessHelpClosed(method, timeOpenMs, attempt, turnSide)`

## Permissions

Merged into your manifest automatically: `CAMERA`, `INTERNET`, `VIBRATE`.
`android.hardware.camera.front` is declared `required="false"`, so your app
stays installable on every device; at launch the SDK finishes with
`Failed(DeviceUnsupported)` on a phone without a front camera, before any
network call. The SDK requests the camera permission itself.

## Play Integrity setup

1. In Google Cloud, create or pick a project and link it to your app in Play Console (App integrity). Enable the Play Integrity API.
2. Give the project number to `VisConfig.playIntegrityCloudProjectNumber`, and share the project with the Vis platform team so it can verify the verdicts.
3. Distribute the app through Google Play. Sideloaded or locally built APKs usually fail the integrity check.
4. Put testers on an internal or closed testing track so they get real verdicts.

With `app_check = "graded"` a phone that cannot attest still binds with the evidence it has; with `"required"` it fails with `AttestationFailed`; with `"off"` no attestation is attempted.

## App Bundle language splits

The SDK ships `uz`, `ru` and `en` strings. App Bundles strip languages the
device does not use, so keep them:

```kotlin
android {
    bundle { language { enableSplit = false } }
}
```

## ProGuard / R8

Consumer rules are included in the AAR and keep the public API. Nothing to add;
do not add a blanket `-keep class uz.vis.sdk.**`.

## Privacy and security

- No media is persisted on the device; uploads are in memory. Images are normalised JPEG with EXIF stripped.
- SDK screens are `FLAG_SECURE`: no screenshots, recents thumbnails or screen recording.
- No tokens, secrets, URLs or request bodies ever reach logs. Diagnostics are silent unless you run `adb shell setprop log.tag.VisSDK DEBUG`.
- Model scores are never received or exposed; results are categories only.
- Transport uses certificate pinning and DPoP-bound requests; the per-attempt device key lives in the Android Keystore and is deleted with the attempt.
- `VisConfig.debugClipExportDir` is a debug-only hook behind `@VisDebugApi` that writes selfie clips to disk. Never use it in production.

## Versioning

Semantic Versioning. Every request carries the SDK version in `X-SDK-Version`;
the server can retire old versions, which surfaces as `DeviceUnsupported`.
See [CHANGELOG.md](CHANGELOG.md).

## License

See [LICENSE](LICENSE).
