# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Open Headunit is a revival fork of the original `headunit` app (GPLv3-AGPL) — an Android app
that turns a phone/tablet into an Android Auto **head unit receiver** (the "car" side of the
Android Auto protocol, not a phone-side client). Source repository:
https://github.com/andreknieriem/open-headunit. Published on Play Store as
`com.andrerinas.headunitrevived`; the GitHub codebase package/namespace is
`com.andrerinas.openheadunit`. The `applicationId` intentionally differs from the `namespace` —
this preserves the original Play Store listing after the app was renamed (see the comment above
`applicationId` in `app/build.gradle.kts`). Same mismatch shows up in Intent actions/receivers;
see `contract/README.md`.

## Modules

- **`:app`** — the entire product: UI, the Android Auto Protocol (AAP) engine, all transports
  (USB/WiFi/Bluetooth/Self-mode), audio/video codec pipeline, and native (JNI/NDK) code.
- **`:contract`** — a tiny, dependency-light module publishing the public inter-app Intent
  contract (broadcast actions/extras) that lets Tasker, MacroDroid, or `adb` drive the app and
  observe session state, without depending on `:app`. Full API reference: `contract/README.md`.

## Build, test, lint

Toolchain: JDK 21 (Temurin — Gradle 8.13 rejects Java 24+), NDK `29.0.14206865` (CI pins
`27.0.12077973`), CMake `3.22.1`. Only the `github` product flavor currently builds — `playstore`
fails because `VpnControl.kt` lives only under `app/src/github/...` but is referenced from shared
`main` source (known issue).

```bash
# Build debug APK
./gradlew :app:assembleGithubDebug --stacktrace

# Run all unit tests
./gradlew :app:testGithubDebugUnitTest --stacktrace

# Run a single test class
./gradlew :app:testGithubDebugUnitTest --tests "com.andrerinas.openheadunit.aap.AapMessageFramingTest"

# Lint (advisory only — abortOnError = false; not run in CI, hangs on generated protobuf sources)
./gradlew :app:lintGithubDebug
```

Test reports land in `app/build/reports/tests/testGithubDebugUnitTest`. Tests live under
`app/src/test/...` and predominantly target the `*Policy` classes (see Architecture below).

Release signing reads, in order: `key.properties`, then `secrets.properties`, then the env vars
`HEADUNIT_KEYSTORE_PASSWORD` / `HEADUNIT_KEY_PASSWORD` (all git-ignored / not committed).

## Architecture

### Connection/transport layer (`app/src/main/java/.../connection/`)

Each transport is organized by type, each with its own Manager/Launcher plus many small,
independently unit-tested `*Policy` classes — the dominant design idiom in this codebase (logic
extracted out of the old god-object service into small testable decision objects; see
CHANGELOG v3.3.0). When adding transport-level behavior, follow this pattern rather than growing
a Manager or Service directly.

- **`connection/usb/`** — wired Android Auto: accessory-mode negotiation, device identity/
  blacklist filtering, native libusb interop (`UsbNative.kt` binds to the `usbhelper` JNI lib).
- **`connection/wifi/`** — wireless Android Auto, split by strategy under `wifi/modes/`:
  - `modes/nativeaa/` — "Native AA": reimplements Google's wireless bring-up (WiFi Direct/SoftAP,
    WPP handshake over TCP+TLS, Bluetooth HFP "poke" wake-up, driver/vehicle selection, QR setup).
  - `modes/helper/` — "Wireless Helper" companion-app launch via Google Nearby Connections
    (deprecated/broken on Android Auto ≥17.4).
  - `wifi/direct/` — WiFi P2P/WiFi Direct management (group identity stability, channel/band
    policy).
  - `wifi/server/` — "Headunit Server" developer-mode connection (phone-initiated TCP on 5277).
  - Root of `wifi/` — `WifiLauncherManager.kt` orchestrates which wireless mode is active;
    policies coordinate wireless bring-up with an active USB session
    (`WirelessBringUpDeferralPolicy.kt`, `UsbSessionQuiescePolicy.kt`).
- **`connection/self/`** — "Self Mode": projects the device onto itself, with launcher variants
  versioned around Android Auto's v17.4 breaking change (`SelfLauncherLegacy.kt`,
  `SelfLauncherV17_4.kt`).
- **`connection/projection/`** — the transport abstraction the AAP layer actually talks to:
  `ProjectionConnection` (USB or WIFI, blocking send/recv), implemented by
  `StandardUsbProjectionConnection`, `LibusbProjectionConnection`, `SocketProjectionConnection`.
  This is the seam where USB/WiFi become interchangeable to the rest of the app.
- **`connection/carkey/`** — steering-wheel/OEM key integration, including a vendor-specific
  `fyt/` module for a common Chinese head-unit hardware vendor.

### Protocol layer (`app/src/main/java/.../aap/`)

Implements the Android Auto wire protocol itself: message framing (`AapMessageFraming.kt`,
`AapReadSingleMessage.kt`), SSL/TLS session setup, channel multiplexing, and service handlers for
audio/video/control/media/navigation/sensors. Wire messages are defined as protobuf schemas in
`app/src/main/proto/*.proto`, compiled to `aap/protocol/proto/*.java`.

`AapService.kt` is the central foreground `Service` owning the notification, wakelocks, and
wiring between USB/wireless/self-mode/BT/audio/GPS. `connection/CommManager.kt` is the actual
state-machine core — it owns both the `ProjectionConnection` (transport) and `AapTransport`
(protocol) lifecycles and exposes one `StateFlow<connectionState>` that the service, activity, and
UI fragments all observe reactively. Read `CommManager.kt`'s class doc (it has a full ASCII state
diagram) before touching connection lifecycle code.

### Media pipeline

- **`decoder/`** — video via `MediaCodec` (H.264/HEVC) with a software fallback
  (`FfmpegHevcDecoder.kt` → native `hur_soft_hevc` lib, built from a vendored FFmpeg tree under
  `app/src/main/cpp/ffmpeg/`); audio via a multi-stream mixer plus mic uplink for voice commands.
  Each has stall-detection/keyframe-recovery/backpressure policies following the same
  policy-object pattern.
- **`view/`** — rendering surfaces (`GlProjectionView`, `TextureProjectionView`), DPI scaling,
  touch/overlay handling.

### Native code (`app/src/main/cpp/`, built via CMake)

Two JNI libraries: `usbhelper` (C, wraps a prebuilt `libusb1.0.so` for libusb-based USB transport)
and `hur_soft_hevc` (C++, wraps the vendored FFmpeg tree for software HEVC decode). Prebuilt
`.so`s live under `jniLibs/<abi>/` for 4 ABIs.

### External automation surface

`:contract`'s `HeadUnitIntent.kt` declares the public Intent API; `automation/AutomationReceiver.kt`
in `:app` is the exported `BroadcastReceiver` that implements it (gated by
`AutomationCommandPolicy.kt`, executed via `AutomationEffectRunner.kt` against
`AapService`/`CommManager`). This is the only cross-app IPC surface — there's no AIDL/Binder
service for external callers (the one `.aidl` file, `IShizuku.aidl`, is solely for Shizuku
privilege elevation, unrelated to automation). See `contract/README.md` for the full action/extra
reference and its documented security constraints (credential redaction on settings export, path
restrictions on writes).

### Product flavors

`playstore` (minSdk 21) and `github` (minSdk 16) differ mainly in whether `VpnControl.kt` /
`DummyVpnService.kt` are present (github-only) and in Conscrypt version pinned for 16 KB page
alignment vs minSdk 16 compatibility.
