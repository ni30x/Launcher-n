# Fix GitHub Actions Debug Signing Build Failure

Resolve the `:app:validateSigningDebug` build failure on GitHub Actions caused by missing `debug.keystore` file in non-local environments.

> [!IMPORTANT]
> **Confirmed Decision**: Auto-detect `${rootDir}/debug.keystore` if present (for local container environments), and fall back to AGP's default built-in debug keystore mechanism (`~/.android/debug.keystore`) on GitHub Actions runners.

---

## 1. Overview & Core Concept

When running `./gradlew assembleDebug` on GitHub Actions, the task `:app:validateSigningDebug` failed with:
`Keystore file '/home/runner/work/Launcher-n/Launcher-n/debug.keystore' not found for signing config 'debugConfig'`.

This occurs because `app/build.gradle.kts` explicitly assigned `signingConfig = signingConfigs.getByName("debugConfig")`, pointing to `${rootDir}/debug.keystore`. On clean CI runners where this file does not exist, AGP fails validation instead of using its built-in default debug keystore.

---

## 2. Technical Solution & Changes

### Conditional Debug Signing in `app/build.gradle.kts`
Modify `app/build.gradle.kts` so `debugConfig` is only created and assigned if `${rootDir}/debug.keystore` actually exists on disk:

```kotlin
signingConfigs {
    if (file("${rootDir}/debug.keystore").exists()) {
        create("debugConfig") {
            storeFile = file("${rootDir}/debug.keystore")
            storePassword = "android"
            keyAlias = "androiddebugkey"
            keyPassword = "android"
        }
    }
    // ... release signing config ...
}

buildTypes {
    debug {
        if (file("${rootDir}/debug.keystore").exists()) {
            signingConfig = signingConfigs.getByName("debugConfig")
        }
    }
}
```

### Benefits
1. **GitHub Actions Compatibility**: On CI runners without `${rootDir}/debug.keystore`, AGP uses standard default debug signing (`~/.android/debug.keystore`), automatically created on demand.
2. **Local Studio Compatibility**: On local container environments where `${rootDir}/debug.keystore` exists, it uses the local keystore seamlessly.

---

## 3. Architecture & Data Flow

```
┌─────────────────────────────────────────────────────────┐
│              app/build.gradle.kts Build                 │
└────────────────────────────┬────────────────────────────┘
                             │
            Check file("${rootDir}/debug.keystore").exists()
                             │
              ┌──────────────┴──────────────┐
              │                             │
          [ Exists ]                  [ Not Exists ]
              │                             │
  Use custom debugConfig         Use AGP default debug
(Local Studio container)        keystore (~/.android/debug.keystore)
                                 (GitHub Actions CI)
```

---

## 4. Verification Plan

1. Verify build locally via `compile_applet`.
2. Confirm both local builds and CI workflow runs pass task `:app:validateSigningDebug` without errors.
