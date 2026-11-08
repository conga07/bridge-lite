# Repository Guidelines

## Project Structure & Architecture

- `crates/bridge-core/` contains Rust scanning, grouping, metadata, XMP, and cache logic; integration tests and fixtures are under `tests/`.
- `crates/bridge-ffi/` exposes the Rust core to Swift through `swift-bridge`.
- `xcode/BridgeLite/BridgeLite/` contains the SwiftUI application, pipelines, stores, and resources. `project.yml` defines the Xcode project.
- `manual-src/` contains the user manual; `docs/` contains published documentation. Build scripts live in `tools/`; the patched XMP toolkit is in `vendor/`.

The filesystem and XMP are the source of truth; SQLite is a cache. Keep core logic in Rust and UI, image rendering, and OS integration in Swift. Preserve the application's focus on viewing, comparing, and rating photos.

## Build and Test

- `make build` builds the release Rust library and the macOS app; `make rust` and `make xcode` run those steps separately.
- `make generate` regenerates the Xcode project after changes to `xcode/BridgeLite/project.yml`.
- `make test` runs `cargo test -p bridge-core`; `cargo build --workspace` builds the Rust workspace.
- `make open` opens the Xcode project. Requirements include macOS, Xcode, Rust, and XcodeGen.

After changing the FFI API, rebuild with `./tools/build-rust-xcframework.sh --release`. Do not edit generated bindings or `project.pbxproj` by hand.

## Style and Testing

Follow existing Rust and Swift formatting and naming conventions: Rust uses `snake_case` for functions and modules, Swift uses `lowerCamelCase` for members and `UpperCamelCase` for types. Keep changes focused and add user-facing strings to both Japanese and English localizations.

Place Rust integration tests in `crates/bridge-core/tests/integration_*.rs` and use committed fixtures, not absolute paths. For manual testing, copy fixtures to a temporary directory: the application can modify XMP metadata. Build the affected application and run relevant tests after changes; no coverage threshold is documented.

## Commits and Pull Requests

Recent history mixes descriptive imperative subjects with scoped prefixes such as `test(bridge-core):` and `perf(macos):`. Use a concise subject that describes the change. For a pull request, explain the behavior and validation, link any relevant issue, and include screenshots for visible UI changes. Commit or push only when requested.

## Fork Constraints

Do not re-enable upstream Sparkle updates. This fork is for personal use; do not distribute builds. Keep upstream's version number and record fork releases in `CHANGELOG.md`. See `CLAUDE.md` for performance constraints and detailed build guidance.
