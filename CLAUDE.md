# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Xamarin.Forms + Xamarin.Android sample app that integrates the **TrueMetrics Android SDK** via a classic Xamarin.Android Binding project. The repo exists to reproduce/fix dependency alignment issues and runtime problems when consuming the native AAR in Xamarin.

## Build

```bash
msbuild TrueMetricsBindingSample.sln -t:Build -p:Configuration=Debug -m
```

Requires macOS, Mono/msbuild, Xamarin.Android tooling, and Android SDK (API 33). Deploy `TrueMetricsSample.Droid` to a physical device (sensors/background location needed).

## Architecture

Three projects in the solution:

- **TrueMetricsSdk.Binding** (`src/TrueMetricsSdk.Binding/`) — Xamarin.Android binding library wrapping the TrueMetrics SDK AAR (downloaded from Maven at build time). Binding transforms in `Transforms/Metadata.xml` remove internal/shaded types (Sentry internals, Koin DI classes, R/BuildConfig). Vendored JARs/AARs for transitive dependencies live in `Jars/` (legacy versions) and `Jars/vendor/` (Maven-aligned versions).

- **TrueMetricsSample** (`src/TrueMetricsSample/`) — Xamarin.Forms shared UI (netstandard2.0). Defines `ITrueMetricsExerciseService` interface and `MainViewModel` which delegates all SDK calls through that interface via `DependencyService`.

- **TrueMetricsSample.Droid** (`src/TrueMetricsSample.Droid/`) — Android platform project. Implements `ITrueMetricsExerciseService` in `TrueMetricsExerciseService.cs` which wraps every SDK call with `RunSdkAsync()` (background thread + timeout) to avoid ANR. `MainActivity.cs` handles the two-phase permission flow (foreground permissions first, then background location separately).

## Dependency Modes (UseVendorMavenDeps)

The `UseVendorMavenDeps` MSBuild property (set in both `.csproj` files, currently `true`) switches between two dependency resolution strategies:

- **`true` (vendor mode, active)** — Uses JARs/AARs from `Jars/vendor/` with versions matching the SDK's actual Maven POM (Ktor 3.3.0, Kotlin 2.2.20, Coroutines 1.10.2, Serialization 1.9.0, Koin 4.1.1, Room, SQLCipher, WorkManager). MSBuild targets strip NuGet-sourced Kotlin stdlib and coroutines JARs to avoid duplicates.

- **`false` (NuGet mode, legacy)** — Uses Xamarin NuGet packages for Kotlin/Coroutines/WorkManager and legacy JARs from `Jars/` root (Ktor 3.0.3, Serialization 1.6.3) plus a `spilling-stub.jar` for Kotlin 2.x compatibility.

When modifying dependencies, keep both modes consistent — update the conditional `<ItemGroup>` blocks in both `TrueMetricsSdk.Binding.csproj` and `TrueMetricsSample.Droid.csproj`.

## Key Files

- `src/TrueMetricsSample/Helpers/Constants.cs` — API key placeholder (`INJECT_API_KEY`), injected at build time from `.secrets.properties` (local) or GitHub secret (CI)
- `src/TrueMetricsSdk.Binding/Transforms/Metadata.xml` — Binding type removals/modifications; update when changing SDK version
- `src/TrueMetricsSample.Droid/TrueMetricsExerciseService.cs` — All SDK interaction logic; SDK calls must go through `RunSdkAsync()` to prevent ANR
- `src/TrueMetricsSample.Droid/Properties/AndroidManifest.xml` — Required permissions and service declarations

## SDK Integration Notes

- The binding uses namespaces `IO.Truemetrics.Truemetricssdk`, `IO.Truemetrics.Truemetricssdk.Config`, `IO.Truemetrics.Truemetricssdk.Engine.State`, `IO.Truemetrics.Truemetricssdk.Engine.Stats`
- SDK properties like `DeviceId`, `SensorStatistics`, `ActiveConfig` may block on IO/binder — always read from a background thread with a timeout
- The native SDK's `StatusListener.askPermissions()` callback is not available in the binding; permissions must be pre-granted before `Init`
- `IForegroundNotificationFactory` must be implemented and passed via `SdkConfiguration.Builder` to provide the foreground service notification
- slf4j-api is pinned to 1.7.36 to avoid D8 NPE with 2.x multi-release JARs
- `kotlinx-io-core` and `kotlinx-io-bytestring` JARs are required as transitive deps of Ktor 3.x — without them SDK cannot load config from backend

## SDK AAR Policy

**Never commit the pre-built `truemetricssdk-*.aar` to the `main` branch.** The CI workflow downloads it from the TrueMetrics Maven repository at build time (controlled by `TRUEMETRICS_SDK_VERSION` env var in `.github/workflows/build.yml`).

On the `develop` branch, a temporary AAR may be committed for testing unreleased SDK builds, but it must be removed before merging to `main`.

The AAR pattern `truemetricssdk-*.aar` is in `.gitignore` — use `git add -f` if you need to temporarily commit one to develop.

## API Key Management

- **Locally**: `.secrets.properties` in repo root (gitignored) with `TRUEMETRICS_API_KEY=<key>`. Multiple keys can be listed — uncomment the active one.
- **CI**: GitHub secret `TRUEMETRICS_API_KEY` (masked in logs). Falls back if `.secrets.properties` is absent.
- The workflow injects the key into `Constants.cs` via `sed` before build.

## Git Workflow & Remotes

This repo has two remotes:

- **`origin`** → `git@github.com:dharamhbtik/truemetrics-xamarin-android-binding-sample.git` — upstream public repo (PostNord's fork)
- **`truemetrics`** → `git@github.com-truemetrics:TRUE-Metrics-io/truemetrics-xamarin-android-binding-sample.git` — TrueMetrics org fork (uses SSH host alias `github.com-truemetrics` from `~/.ssh/config`)

### Branches

- **`main`** — clean branch, no pre-built AARs, no workaround patches. PRs to upstream go from here.
- **`develop`** — working branch for testing. May contain temporary AAR commits, workarounds, and experimental changes. CI runs on push.

### Day-to-day workflow

1. Work on `develop`, push to `truemetrics` remote: `git push truemetrics develop`
2. CI builds APK automatically on push (GitHub Actions on `macos-15-intel`)
3. Download artifact via `gh run download <run-id> --repo TRUE-Metrics-io/truemetrics-xamarin-android-binding-sample`
4. Install on device: `adb install <path-to-apk>`

### Opening a PR to the upstream public repo

1. Clean up `develop` — remove temporary AAR commits, ensure AAR is downloaded from Maven
2. Merge/cherry-pick clean changes into `main`
3. Push `main` to `truemetrics` remote: `git push truemetrics main`
4. Open PR from `TRUE-Metrics-io/truemetrics-xamarin-android-binding-sample:main` → `dharamhbtik/truemetrics-xamarin-android-binding-sample:main` via GitHub UI or `gh pr create --repo dharamhbtik/truemetrics-xamarin-android-binding-sample --head TRUE-Metrics-io:main`

### Opening issues for native SDK bugs

File issues in the native SDK repo: `TRUE-Metrics-io/truemetrics_android_SDK_p`

```bash
gh issue create --repo TRUE-Metrics-io/truemetrics_android_SDK_p --title "..." --body "..."
```

Include: root cause analysis, crash logs, bytecode analysis if relevant, suggested fix, and environment details (device, SDK version, binding version).

### CI (GitHub Actions)

- Workflow: `.github/workflows/build.yml`
- Runner: `macos-15-intel` (GitHub-hosted)
- Triggers: push to `main`/`develop`, PR to `main`, manual dispatch
- Artifacts: APK + DLLs, retained 14 days
- Secrets: `TRUEMETRICS_API_KEY` in repo Settings → Secrets → Actions
- SDK version: `TRUEMETRICS_SDK_VERSION` env var in workflow
