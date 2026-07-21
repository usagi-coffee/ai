---
name: android-aarch64
description: Build and validate Android projects on aarch64 or arm64 Linux hosts when the Android SDK, NDK, Gradle, or JDK tools are x86_64 and require muvm. Use for Android Gradle builds, Rust/JNI cross-compilation, NDK linker failures, missing Gradle executables, muvm environment errors, and debug or signed release APK verification on ARM Linux.
---

# Android aarch64 Builds

Check `uname -m` before invoking Android tools. Apply this workflow only on `aarch64` or `arm64`; use the project's normal Android commands on other hosts.

## Choose the build path

- For Kotlin, resources, manifests, or Gradle changes, assemble a debug APK through Gradle under `muvm`.
- For Rust/JNI changes, run native Cargo checks and the project's Android Rust build script first, then assemble the APK.
- Do not rebuild Rust for Kotlin-only changes when the required JNI libraries already exist.
- Do not run a signed release build merely for validation.

Inspect project build scripts before wrapping commands. A well-designed ARM-host Rust script keeps Cargo, rustc, build scripts, and C compilation native to ARM and sends only final links through the official x86_64 NDK Clang under `muvm`. Run such a script normally; do not wrap the whole script in `muvm`.

## Run Gradle

A bare `gradle` may be absent from `PATH`. Resolve the required executable under the Gradle wrapper cache or use the project wrapper when it is compatible with the host:

```sh
find "$HOME/.gradle/wrapper/dists" \
  -path '*/gradle-*/bin/gradle' -type f -print
```

Confirm `ANDROID_HOME` and `ANDROID_SDK_ROOT` exist, resolve explicit absolute paths, then run:

```sh
muvm -e ANDROID_HOME -e ANDROID_SDK_ROOT -i -t -- \
  /absolute/path/to/gradle/bin/gradle \
  -p /absolute/path/to/project/android --no-daemon :app:assembleDebug
```

Pass `-e NAME` only for variables that are set in the host environment. `muvm` fails before launching Gradle when asked to forward an unset variable. In particular, omit `-e GRADLE_USER_HOME` unless it exists. Forward signing variables only for an explicitly requested release.

In a sandboxed agent session, request escalation for `muvm`: it launches a VM and reads SDK, Gradle, and JDK caches outside the workspace.

## Verify output

Confirm the Gradle process exits successfully and inspect the expected APK, commonly:

```sh
ls -lh android/app/build/outputs/apk/debug/app-debug.apk
```

For signed releases, prefer the project's release script because it should handle architecture selection, credentials, signing, and `apksigner` verification. Never expose signing secrets in command output.

## Diagnose common failures

- `gradle: command not found`: locate the cached Gradle executable and invoke its absolute path.
- `Failed to get NAME env var`: remove `-e NAME` or define the required variable before invoking `muvm`.
- Missing NDK Clang or sysroot: verify `ANDROID_NDK_HOME`, `ANDROID_HOME`, or `ANDROID_SDK_ROOT` and inspect the project's NDK-selection logic.
- Missing JNI `.so` files: run the project's Rust/JNI build before Gradle.
- Sandbox or VM startup denial: rerun the same scoped `muvm` command with escalation.
