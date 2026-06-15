# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Build Commands
- `cargo run` - Build and run the desktop application (requires libsciter library)
- `python3 build.py --flutter` - Build Flutter version (desktop)
- `python3 build.py --flutter --release` - Build Flutter version in release mode
- `python3 build.py --hwcodec` - Build with hardware codec support
- `cargo build --release` - Build Rust binary in release mode
- `cargo build --features hwcodec` - Build with specific features

### Testing
- `cargo test` - Run all Rust tests
- `cargo test <test_name>` - Run a single Rust test by name
- `cd flutter && flutter test` - Run Flutter tests

### Flutter-Rust Bridge Code Generation
The bridge between Flutter and Rust is auto-generated. After changing `src/flutter_ffi.rs`, regenerate:
```sh
flutter_rust_bridge_codegen \
  --rust-input ./src/flutter_ffi.rs \
  --dart-output ./flutter/lib/generated_bridge.dart \
  --c-output ./flutter/macos/Runner/bridge_generated.h
cp ./flutter/macos/Runner/bridge_generated.h ./flutter/ios/Runner/bridge_generated.h
```
Required tool versions: `flutter_rust_bridge_codegen` 1.80.1, `cargo-expand` 1.0.95.

**Do not edit these generated files manually:**
- `src/bridge_generated.rs` / `src/bridge_generated.io.rs`
- `flutter/lib/generated_bridge.dart` / `flutter/lib/generated_bridge.freezed.dart`
- `flutter/macos/Runner/bridge_generated.h` / `flutter/ios/Runner/bridge_generated.h`

### Platform-Specific Build Scripts
- `flutter/build_android.sh` - Android build script
- `flutter/build_ios.sh` - iOS build script
- `flutter/build_fdroid.sh` - F-Droid build script

## Project Architecture

### Toolchain Requirements
- Minimum Rust version: **1.75** (see `rust-version` in `Cargo.toml`)
- Flutter version: **3.22.3** (used in CI)
- vcpkg dependencies: `libvpx`, `libyuv`, `opus`, `aom` — set `VCPKG_ROOT`

### Communication Architecture

```
Flutter UI (Dart)
    ↕  flutter_rust_bridge FFI (sync/async calls via generated_bridge.dart)
Rust library (librustdesk)
    ↕  IPC socket (parity-tokio-ipc named pipe)
Rust service (background daemon)
    ↕  Network (TCP/UDP, protobuf)
rustdesk-server (rendezvous + relay)
```

- **FFI layer**: `src/flutter_ffi.rs` exposes functions to Dart. Dart calls them via `flutter/lib/generated_bridge.dart`.
- **IPC layer**: `src/ipc.rs` — Flutter app communicates with the background service process via named pipe/unix socket. The service runs separately from the UI process.
- **Network layer**: `src/rendezvous_mediator.rs` handles peer discovery and relay via rustdesk-server. Direct connections use TCP/KCP; relay uses the server.
- **Session layer**: `src/client.rs` manages outbound peer connections; `src/server/connection.rs` manages inbound.

### Key Source Files
- `src/flutter_ffi.rs` - All Rust functions callable from Flutter (FFI boundary)
- `src/ui_interface.rs` / `src/ui_session_interface.rs` - UI-facing abstractions
- `src/server/` - Per-service implementations: `video_service.rs`, `audio_service.rs`, `input_service.rs`, `display_service.rs`, `clipboard_service.rs`
- `src/client/` - Client-side session handling and file transfer
- `libs/scrap/` - Screen capture (platform-specific)
- `libs/enigo/` - Keyboard/mouse injection (platform-specific)
- `libs/hbb_common/` - Shared types: protobuf definitions, config, network utilities

### Config System
All config lives in `libs/hbb_common/src/config.rs`, four config types:
- `Config` (Settings) - global settings
- `LocalConfig` - per-machine settings
- `PeerConfig` - per-peer settings
- Built-in defaults

### Flutter UI Structure
- `flutter/lib/desktop/` - Desktop-specific screens and widgets
- `flutter/lib/mobile/` - Mobile-specific screens
- `flutter/lib/common/` - Shared widgets and utilities
- `flutter/lib/models/` - State models (GetX-based): `model.dart` is the main session model, `server_model.dart` for the controlled-side panel

### Feature Flags
- `hwcodec` - Hardware video encoding/decoding
- `vram` - VRAM optimization (Windows only)
- `flutter` - Enable Flutter UI (required for Flutter builds)
- `unix-file-copy-paste` - Unix file clipboard support
- `screencapturekit` - macOS ScreenCaptureKit

### Ignore Patterns
- `target/` - Rust build artifacts
- `flutter/build/` - Flutter build output
- `flutter/.dart_tool/` - Flutter tooling files

### Cross-Platform Notes
- Windows: requires additional DLLs and virtual display drivers; `vram` feature available
- macOS: requires signing/notarization for distribution; use `screencapturekit` feature
- Linux: supports deb, rpm, AppImage; Wayland support via `scrap/wayland`
- Android/iOS: many desktop-only modules are `cfg`-gated with `#[cfg(not(any(target_os = "android", target_os = "ios")))]`
