# MCP Kotlin SDK on Android — Compatibility Summary

Research notes on whether [`modelcontextprotocol/kotlin-sdk`](https://github.com/modelcontextprotocol/kotlin-sdk)
(the official Kotlin Multiplatform SDK for the Model Context Protocol) can be used on Android hardware.

## TL;DR

**Usable, but not officially supported.** There is no dedicated Android target, artifact, or test
coverage. Apps have successfully depended on the SDK's plain **JVM** artifact from Android modules
(consuming a Kotlin/JVM library from Android is a common, if unofficial, pattern), but this has already
surfaced Android-specific build problems, and the maintainers have explicitly declined to add
first-class Android support.

## What the SDK targets

The project is a Kotlin Multiplatform (KMP) library. Its Gradle build (`buildSrc/.../mcp.multiplatform.gradle.kts`
and the per-module `build.gradle.kts` files) declares these targets:

- `jvm` (Java 11 bytecode, built with JDK 21 toolchain)
- Native: `macosArm64`, `linuxX64`, `linuxArm64`, `mingwX64`, plus (in `kotlin-sdk-core`/`kotlin-sdk-client`)
  `iosArm64`, `iosX64`, `iosSimulatorArm64`, `watchosArm64`, `watchosSimulatorArm64`, `tvosArm64`, `tvosSimulatorArm64`
- `js` (browser + Node.js)
- `wasmJs` (browser + Node.js)

There is **no `androidTarget()`** anywhere in the build, no Android Gradle Plugin applied, and no
`android`-suffixed publication on Maven Central. The official docs site
(kotlin.sdk.modelcontextprotocol.io) also makes no mention of Android as a supported platform.

## Why it *can* still run on Android

- Android apps compile Kotlin/JVM bytecode, so a plain Android module (not itself a KMP project) can
  typically resolve and consume the SDK's `jvm` publication as an ordinary Java/Kotlin dependency —
  the same trick many JVM-only libraries rely on for de-facto Android usage.
- Core runtime dependencies (`kotlinx.serialization`, `kotlinx.coroutines`, `kotlinx.io`,
  `kotlinx.collections.immutable`) are pure Kotlin/multiplatform and Android-friendly.
- `kotlin-sdk-client` only depends on `ktor-client-core`, which is engine-agnostic — you supply your
  own Android-compatible Ktor engine (e.g. OkHttp or CIO).
- Netty (used by the server module) is scoped to `jvmTest` only in this repo — it is **not** a runtime
  dependency shipped to consumers, so it isn't a concern for a production Android app.
- The JVM target compiles to Java 11 bytecode; Android's D8/R8 toolchain generally handles this fine,
  though apps with `minSdk` below API 26 may need core library desugaring for certain APIs.

## Evidence people are already trying this — and hitting rough edges

- **[Issue #272](https://github.com/modelcontextprotocol/kotlin-sdk/issues/272) — "Duplicate LibVersionKt
  causing Android build failure"**: an Android build failed at the duplicate-class check because
  `LibVersionKt` was emitted into both `kotlin-sdk-client-jvm` and `kotlin-sdk-core-jvm`. This is an
  Android/R8-specific failure mode (plain JVM apps don't run this check), confirming real attempts to
  use the SDK from Android apps. It was fixed via
  [PR #274](https://github.com/modelcontextprotocol/kotlin-sdk/pull/274), which moved the
  `generateLibVersion` task so the constant is only generated once.
- **[Issue #234](https://github.com/modelcontextprotocol/kotlin-sdk/issues/234) — request for a
  dedicated Android SDK**: closed as **"not planned"** by the maintainers, i.e. they have decided
  against building first-class/native Android support (e.g. an `androidTarget()` or Android-specific
  APIs like cross-app IPC) into this SDK.
- **[AnswerZhao/android-mcp-sdk](https://github.com/AnswerZhao/android-mcp-sdk)**: a community project
  that wraps/extends `kotlin-sdk` with Android AIDL/Binder IPC so separate Android apps can talk MCP to
  each other on-device — built precisely because the official SDK doesn't address that use case.

## Practical guidance

- **MCP client in an Android app**: workable today by depending on the JVM artifacts
  (`kotlin-sdk-core`, `kotlin-sdk-client`) and adding a suitable Ktor HTTP engine yourself. Expect to
  potentially hit JVM/Android packaging quirks (as in #272) that plain-JVM users won't see, since this
  path isn't part of the project's CI.
- **Embedding an MCP server on-device** is technically possible via `kotlin-sdk-server` + a JVM Ktor
  server engine, but this is a heavier, less common use case on mobile and is even further from what
  the project tests.
- **Cross-app / OS-level MCP integration on Android** (e.g. exposing tools between installed apps) is
  explicitly out of scope for the official SDK — look at community efforts like `android-mcp-sdk`
  instead.
- Because there's no Android target in the project's own test matrix, treat any Android usage as
  unsupported/best-effort: pin versions carefully and verify builds after upgrades.

## Sources

- [modelcontextprotocol/kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk) — repository, `build.gradle.kts` files, `buildSrc/src/main/kotlin/mcp.multiplatform.gradle.kts`, `gradle/libs.versions.toml`
- [MCP Kotlin SDK docs site](https://kotlin.sdk.modelcontextprotocol.io/)
- [Issue #272 — Duplicate LibVersionKt causing Android build failure](https://github.com/modelcontextprotocol/kotlin-sdk/issues/272)
- [PR #274 — fix for the duplicate class issue](https://github.com/modelcontextprotocol/kotlin-sdk/pull/274)
- [Issue #234 — request for an Android-specific SDK (closed, not planned)](https://github.com/modelcontextprotocol/kotlin-sdk/issues/234)
- [AnswerZhao/android-mcp-sdk](https://github.com/AnswerZhao/android-mcp-sdk) — community Android IPC extension

*Compiled 2026-08-14.*
