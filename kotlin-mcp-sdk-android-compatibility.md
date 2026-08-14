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

## Should the project add `androidTarget()`?

Probably worth requesting, and it would be a cheap change relative to the payoff:

- **Its own dependencies already support Android.** Ktor, `kotlinx.coroutines`, `kotlinx.serialization`,
  and `kotlinx.collections.immutable` all publish an Android target already. Adding
  `androidTarget()` to `kotlin-sdk-core`/`-client`/`-server` wouldn't require porting any
  platform-specific code — `commonMain` would compile as-is.
- **It converts "happens to work" into "actually tested."** Right now Android usage relies on Gradle
  resolving the `jvm` variant for a non-KMP Android module — that's an unofficial pattern, not something
  the project's own Gradle Module Metadata declares support for. A real `androidTarget()` publishes a
  proper `-android` variant with correct `org.jetbrains.kotlin.platform.type=androidJvm` attributes, so
  resolution is intentional rather than coincidental.
- **It would have caught #272 automatically.** A `androidTarget()` build (or the smoke test the
  maintainers said they'd add) runs D8's duplicate-class check in CI, catching bugs like the
  `LibVersionKt` collision before release instead of after a user reports it.
- **It's a smaller ask than what #234 requested and was refused.** That issue asked for a
  purpose-built Android SDK (cross-app AIDL/Binder IPC) — a genuinely large, opinionated feature.
  Plain `androidTarget()` support for library consumption is a much narrower, mostly mechanical change
  and wasn't really what got turned down.
- **Downsides for the maintainers**: an Android SDK/emulator toolchain in CI, a `compileSdk`/`minSdk`
  policy to define and maintain, and one more published variant to keep green — real but bounded costs.

If you want Android to be a first-class target, the actionable move is a *scoped* feature request (or
PR) that asks specifically for `androidTarget()` on the existing modules — distinct from re-opening #234.

## How to test Android compatibility cheaply

Ordered from fastest/no-tooling to most realistic:

1. **Duplicate-class check without any Android SDK** (mirrors exactly what broke in #272 — Android's
   D8/R8 step rejects a build if the same class appears in two dependency jars, which plain JVM apps
   never check for). Download the published `-jvm` artifacts and diff their class lists:

   ```bash
   for m in kotlin-sdk-core-jvm kotlin-sdk-client-jvm kotlin-sdk-server-jvm; do
     curl -sSO "https://repo1.maven.org/maven2/io/modelcontextprotocol/$m/<version>/$m-<version>.jar"
     unzip -l "$m-<version>.jar" | awk '{print $4}' | grep '\.class$' | sort > "$m.classes.txt"
   done
   comm -12 kotlin-sdk-core-jvm.classes.txt kotlin-sdk-client-jvm.classes.txt   # should be empty
   comm -12 kotlin-sdk-core-jvm.classes.txt kotlin-sdk-server-jvm.classes.txt   # should be empty
   comm -12 kotlin-sdk-client-jvm.classes.txt kotlin-sdk-server-jvm.classes.txt # should be empty
   ```

   **Ran this against the current latest release (0.15.0) as part of this research: no duplicate
   classes across any pair of the three `-jvm` jars, and `LibVersionKt` now exists only in
   `kotlin-sdk-core-jvm`** — confirming PR #274's fix holds in the released artifact.

2. **Dependency resolution / variant check** (needs Android Gradle Plugin, no emulator): create a
   throwaway `com.android.library` (or `application`) module, add the SDK as a dependency, and run
   `./gradlew :app:dependencies --configuration debugRuntimeClasspath` to confirm Gradle actually
   resolves a compatible variant instead of failing or silently doing something unexpected.

3. **Full Android build with minification** (needs Android SDK, no emulator): same throwaway module,
   `minifyEnabled true`, run `./gradlew :app:assembleRelease`. This exercises the exact D8/R8 duplicate-
   class and API-availability checks that Step 1 approximates without tooling, plus catches any
   `NoClassDefFoundError`-style issues from `minSdk`. Try a couple of `minSdk` values (e.g. 24 and 26)
   to see whether core-library desugaring is required.

4. **Robolectric unit test** (JVM-only, no emulator, no real device): exercise an actual
   `Client`/`Server` round trip using `ChannelTransport` (already used in the SDK's own tests) inside a
   Robolectric-hosted JVM test in the Android module — verifies `kotlinx.coroutines`/`serialization`
   behave correctly under Android's simulated runtime, fast enough for routine CI.

5. **Real emulator/device smoke test** (slowest, most realistic): a small instrumented test app using
   `ktor-client-okhttp` (or `-cio`) as the client engine, talking SSE/Streamable-HTTP to a local MCP
   test server, run on a couple of emulator API levels. This is the only step that validates real
   networking/socket behavior on Android rather than just packaging/compilation.

6. **Fork-and-compile check** (fastest way to see *why* Android isn't a target, if it's ever attempted):
   in a local clone, add `androidTarget()` + `id("com.android.library")` to the multiplatform blocks and
   run `./gradlew :kotlin-sdk-client:compileDebugKotlinAndroid`. Any `expect`/`actual` gaps or
   Android-unavailable JVM APIs surface immediately, without needing a sample app at all.

Steps 1 and 6 need no Android SDK/emulator and are cheap enough to run as part of routine research;
steps 2–3 need only `compileSdk`/build-tools (no emulator); steps 4–5 are the ones worth reserving for
an actual CI pipeline or a "does this really work at runtime" spot-check.

## Sources

- [modelcontextprotocol/kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk) — repository, `build.gradle.kts` files, `buildSrc/src/main/kotlin/mcp.multiplatform.gradle.kts`, `gradle/libs.versions.toml`
- [MCP Kotlin SDK docs site](https://kotlin.sdk.modelcontextprotocol.io/)
- [Issue #272 — Duplicate LibVersionKt causing Android build failure](https://github.com/modelcontextprotocol/kotlin-sdk/issues/272)
- [PR #274 — fix for the duplicate class issue](https://github.com/modelcontextprotocol/kotlin-sdk/pull/274)
- [Issue #234 — request for an Android-specific SDK (closed, not planned)](https://github.com/modelcontextprotocol/kotlin-sdk/issues/234)
- [AnswerZhao/android-mcp-sdk](https://github.com/AnswerZhao/android-mcp-sdk) — community Android IPC extension

*Compiled 2026-08-14.*
