# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build commands

```bash
# Build debug APK
./gradlew assembleDebug

# Build release APK (all ABIs)
./gradlew assembleRelease

# Build for a single ABI (arm64-v8a, armeabi-v7a, x86, x86_64)
./gradlew assembleDebug -PbuildABI=arm64-v8a

# Install debug build to device
./gradlew installDebug

# Run unit tests (JVM)
./gradlew test

# Run instrumentation tests (requires device/emulator)
./gradlew connectedAndroidTest

# Clean build
./gradlew clean
```

## Build prerequisites

- Android SDK Platform & Build-Tools 35+ (see `build-logic/convention/src/main/kotlin/Versions.kt` for exact versions)
- Android NDK 28+ (Side by side) & CMake 3.22.1+
- `extra-cmake-modules` (KDE ECM) and GNU Gettext >= 0.20
- On Windows: enable Developer Mode for symlinks, `git config --global core.symlinks true`

## Architecture overview

This is an Android input method (IME) that ports the Fcitx5 framework to Android. It has a Kotlin UI layer, a JNI bridge to native C++ fcitx5 libraries, and a plugin system where each input method engine ships as a separate APK.

### Threading model

Fcitx5 runs on a dedicated native thread managed by **`FcitxDispatcher`** (`app/src/main/java/.../core/FcitxDispatcher.kt`). All fcitx API calls are dispatched to this thread via `CoroutineDispatcher.dispatch()`. The native event loop (`nativeLoopOnce()`) blocks until an event arrives, then schedules pending jobs from a concurrent queue.

### Lifecycle: Daemon → Connection → API

1. **`FcitxDaemon`** (`app/src/main/java/.../daemon/FcitxDaemon.kt`) is a singleton that manages the single `Fcitx` instance using reference counting. Clients call `connect(name)` (returns `FcitxConnection`) and `disconnect(name)`. When no clients remain, fcitx is stopped.

2. **`FcitxConnection`** is a lightweight wrapper that exposes `runImmediately(block)` (blocking) and `runIfReady { }` (coroutine). All operations ultimately dispatch through `FcitxDispatcher` onto the fcitx thread.

3. **`FcitxAPI`** (`app/src/main/java/.../core/FcitxAPI.kt`) defines the full interface: sendKey, select candidate, get/set config, manage input methods, etc.

4. **`FcitxLifecycle`** (`app/src/main/java/.../core/FcitxLifecycle.kt`) is a state machine: `STOPPED → STARTING → READY → STOPPING → STOPPED`. Observers can `whenReady { }` or `whenStopped { }` to suspend until a state is reached.

### Key layers

- **`app/src/main/java/.../core/`** — Fcitx engine wrapper. `Fcitx.kt` is the JNI-backed implementation. C++ side is in `app/src/main/cpp/native-lib.cpp`.
- **`app/src/main/java/.../input/`** — IME layer. `FcitxInputMethodService.kt` is the `InputMethodService`. Keyboard views (`keyboard/`), candidate views (`candidates/`), popup previews (`popup/`), emoji/symbol picker (`picker/`).
- **`app/src/main/java/.../data/`** — Preferences (`prefs/AppPrefs.kt` uses `SharedPreferences` with a declarative DSL), theme manager, clipboard manager, user data (pinyin dict, quick phrase, punctuation).
- **`app/src/main/java/.../ui/`** — Main app UI (settings, theme picker, plugin list, about, debug log) and setup wizard (`ui/setup/`).
- **`app/src/main/cpp/`** — JNI native code. `native-lib.cpp` is the main entry point. Addon modules: `androidfrontend/`, `androidkeyboard/`, `androidnotification/`, `androidaddonloader/`.
- **`app/src/main/java/.../FcitxApplication.kt`** — `Application` subclass, initializes prefs, theme, clipboard, crash handling.

### Native libraries (`lib/`)

Each is a Gradle module with CMake native code, treated as prebuilt libraries installed into app assets:
- `lib/fcitx5/` — core fcitx5 framework
- `lib/fcitx5-chinese-addons/` — Chinese input methods (pinyin, shuangpin, wubi, cangjie)
- `lib/libime/` — libime input method engine
- `lib/fcitx5-lua/` — Lua scripting

### Plugin system

Each plugin in `plugin/<name>/` is a separate APK (separate `applicationId`). They communicate with the main app via AIDL IPC:

- **`lib/plugin-base/`** — base library that all plugins depend on
- **`lib/common/`** — `FcitxPluginService` (abstract `Service` base class for plugins) and `FcitxRemoteConnection` (IPC via `IFcitxRemoteService` AIDL)
- Plugins load their native libraries independently and call into the main app's native code through the fcitx addon loader

### Build conventions (`build-logic/`)

Custom Gradle plugins in `build-logic/convention/src/main/kotlin/`:
- `FcitxComponentPlugin` — builds CMake targets and installs output into `assets/usr/`
- `NativeBaseConventionPlugin` / `NativeAppConventionPlugin` — CMake + NDK configuration
- `AndroidAppConventionPlugin` / `AndroidLibConventionPlugin` — base Android config
- `BuildMetadataPlugin` — generates version metadata
- `DataDescriptorPlugin` — generates asset manifest JSON

### Code generation (`codegen/`)

KSP annotation processor that reads fcitx header files and generates `FcitxKeyMapping` and `ScancodeMapping` object declarations mapping fcitx key constants to Kotlin.

### Keyboard layouts

Keyboard layouts are defined programmatically in `KeyDefPreset.kt` as `List<List<KeyDef>>` grids. There's a `BaseKeyboard` class that renders keys using ConstraintLayout, with subclasses for different layouts. The `KeyView` handles touch events (press, swipe up/down/left/right for symbols), long-press popup, and preview.

### Preferences system

`AppPrefs` (`app/src/main/java/.../data/prefs/AppPrefs.kt`) is a declarative preferences tree backed by `SharedPreferences`. Each preference is declared with a key and default value. `ManagedPreference` supports `by` delegation and `OnChangeListener` for reactive changes. Categories are nested inner classes.
