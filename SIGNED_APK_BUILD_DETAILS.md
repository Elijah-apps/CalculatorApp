# CalculatorApp — Signed Release APK Build Details

This document records everything required to build, sign, and verify the
**release APK** for the **CalculatorApp** Android project. It is intended as a
repeatable build/runbook for anyone regenerating the signed APK.

---

## 1. Overview

| Field               | Value                                |
|---------------------|--------------------------------------|
| Application label   | CalculatorApp                        |
| Package name        | `com.example.calculatorapp`          |
| `versionCode`       | `1`                                  |
| `versionName`       | `1.2`                                |
| `minSdkVersion`     | `27`  (Android 8.1)                  |
| `targetSdkVersion`  | `33`  (Android 13)                   |
| `compileSdkVersion` | `33`                                 |
| Build variant       | `release`                            |
| Signing             | V2 scheme (apksigner), verified `exit=0` |

---

## 2. Build Environment

| Component                          | Version / Path                                              |
|------------------------------------|-------------------------------------------------------------|
| Gradle (wrapper)                   | `8.2`  (`gradle-8.2-bin.zip`)                               |
| Gradle wrapper                     | `./gradlew`                                                 |
| Android Gradle Plugin (AGP)        | `8.2.2`  (`com.android.application`)                          |
| Kotlin                             | `1.9.0`  (`org.jetbrains.kotlin.android`)                   |
| Java / JDK                         | Temurin `17.0.20.1`  (`/home/elijah/Android/jdks/jdk-17.0.20.1+1`) |
| Android SDK root                   | `/home/elijah/Android/Sdk`                                  |
| Platform                           | `platforms;android-33` (installed)                          |
| Build-Tools                          | `34.0.0` (installed, used for packaging/signing)          |

> **Note on JDK:** AGP 8.2 requires **JDK 17**. The OS default on this machine
> is OpenJDK 11, so the build was run with `JAVA_HOME` pointed at the bundled
> Temurin 17 JDK:
> ```
> export JAVA_HOME=/home/elijah/Android/jdks/jdk-17.0.20.1+1
> export ANDROID_HOME=/home/elijah/Android/Sdk
> export ANDROID_SDK_ROOT=/home/elijah/Android/Sdk
> ```

---

## 3. The Signed APK

| Property        | Value                                                                          |
|-----------------|--------------------------------------------------------------------------------|
| Output path     | `app/build/outputs/apk/release/app-release.apk`                                |
| File name       | `app-release.apk`                                                              |
| Size            | `1,811,685 bytes`  (~1.8 MB)                                                   |
| MD5             | `2587c4a138c7562ace1fe65ef8865717`                                              |
| SHA-1           | `5ca2dd08e0d4e02b5475e45588fa583cc673df22`                                     |
| SHA-256         | `2469d59adeb2dd5f686f764eb3cd30bf42195b3c274a7fe617041f0c047b3b93`            |
| Apksigner verify| `Verified using v2 scheme (APK Signature): true`  (`exit=0`)                   |

### Signing certificate (from the APK, via `apksigner verify --print-certs`)

| Digest | Value (lowercase, colon-separated) |
|--------|-------------------------------------|
| SHA-256 | `4f54d3bbc8f3560e283ead1a447fd7d81a6c7a36c5ba3d8d5bb3ceef66b2c17b` |
| SHA-1   | `f3fd32e8313794e3f98bf177a24162d6d214f58e` |
| MD5     | `c1ed324722d710030d075240bb8c3803` |

These digests **match the keystore** (see §4), confirming the APK is signed with
the project's release key.

---

## 4. Signing Configuration

The release build is signed with a **PKCS#12** keystore checked out in the
working tree (gitignored — never committed):

| Property          | Value                            |
|-------------------|----------------------------------|
| Keystore file     | `calculator.keystore` (project root) |
| Keystore type     | `PKCS12` (provider: SUN)         |
| Key alias         | `calculatorkey`                  |
| Signature algorithm | `SHA256withRSA`                |
| Key strength     | `2048-bit RSA`                   |
| Certificate owner | `CN=CalculatorApp, OU=Mobile, O=Example Org, L=., ST=., C=US` |
| Valid from        | `2026-10-16 07:48 WAT`           |
| Valid until       | `2054-03-03 07:48 WAT`           |

### How the credentials are supplied

The signing passwords/alias live in **`keystore.properties`** at the project root
(gitignored). The file exposes exactly these four keys:

```
storeFile=calculator.keystore
storePassword=<redacted — keep in keystore.properties>
keyAlias=calculatorkey
keyPassword=<redacted — keep in keystore.properties>
```

⚠️ The actual secret values must remain in `keystore.properties` only. They are
**not** duplicated into this document or committed to the repository. Treat
`keystore.properties` and `calculator.keystore` as sensitive secrets.

In `app/build.gradle.kts` the release signing config is wired up as:

```kotlin
signingConfigs {
    create("release") {
        val propsFile = rootProject.file("keystore.properties")
        val props = Properties()
        propsFile.inputStream().use { props.load(it) }
        storeFile = rootProject.file(props.getProperty("storeFile"))
        storePassword = props.getProperty("storePassword")
        keyAlias = props.getProperty("keyAlias")
        keyPassword = props.getProperty("keyPassword")
    }
}
```

and bound to the release build type:

```kotlin
buildTypes {
    getByName("release") {
        isDebuggable = false
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
        signingConfig = signingConfigs.getByName("release")
    }
    debug { isDebuggable = true }
}
```

The release build uses **R8** code shrink + resource shrink
(`isMinifyEnabled = true`, `isShrinkResources = true`).

---

## 5. Build Command

From the project root, with the environment set as in §2:

```bash
export JAVA_HOME=/home/elijah/Android/jdks/jdk-17.0.20.1+1
export ANDROID_HOME=/home/elijah/Android/Sdk
export ANDROID_SDK_ROOT=/home/elijah/Android/Sdk

# Clean (optional) then signed release build:
./gradlew :app:assembleRelease --no-daemon
```

The signed APK is emitted at:

```
app/build/outputs/apk/release/app-release.apk
```

### Verifying the built APK

```bash
# Signature + v2 scheme verification (uses installed build-tools 34.0.0):
$ANDROID_HOME/build-tools/34.0.0/apksigner verify --print-certs \
    app/build/outputs/apk/release/app-release.apk
```
(A successful verification prints the signer certificate digests and exits `0`.)

---

## 6. Issues Found & Fixes Applied

Several issues blocked a clean signed build of the existing source tree. Each
fix is recorded below (all changes are in the working tree and staged for commit).

1. **Stray `"` in `app/src/main/res/values/themes_dark.xml`**
   The `colorAccent` item had a trailing quote:
   `<item name="colorAccent">@color/sandyTeal</item>"`.
   This is invalid XML and aborts resource processing. Removed the stray
   character.

2. **Missing Kotlin import in `app/build.gradle.kts`**
   The signing-config block referenced `java.util.Properties()`, which is **not**
   among Kotlin's default imports, producing
   `Unresolved reference: util` and a Gradle script-compilation error.
   Fix: added `import java.util.Properties` at the top of the file and changed the
   call to `Properties()`.

3. **Keystore resolved from the wrong directory**
   `storeFile = file(props.getProperty("storeFile"))` resolved relative to the
   `app/` module directory, so Gradle looked for `app/calculator.keystore` and
   failed:
   `Keystore file '.../app/calculator.keystore' not found for signing config 'release'`.
   Fix: `storeFile = rootProject.file(props.getProperty("storeFile"))`, resolving the
   keystore relative to the project root where `calculator.keystore` lives.

4. **No `local.properties` (SDK location unknown to Gradle)**
   `ANDROID_HOME`/`ANDROID_SDK_ROOT` were unset and no `local.properties` was
   present. Created a **gitignored** `local.properties` pointing at the local SDK:
   ```
   sdk.dir=/home/elijah/Android/Sdk
   ```

5. **JDK mismatch**
   The OS default JDK is 11, but AGP 8.2 requires JDK 17. Resolved by running
   the build with `JAVA_HOME` set to the bundled Temurin JDK 17.

---

## 7. Post-Build Files (build artifacts)

```
app/build/outputs/apk/release/app-release.apk   # <-- the signed APK (1.8 MB)
app/build/outputs/apk/release/app-release.apk.id.sig
app/build/outputs/mapping/release/              # ProGuard/R8 mapping (deobfuscation)
app/build/intermediates/lint_vital_intermediate_text_report/release/lint-results-release.txt
                                                              # "No issues found."
build_release.log                              # full gradle console output
```

Keep `app/build/outputs/mapping/release/` for de-obfuscating stack traces from
this exact build (it corresponds 1:1 to `app-release.apk` by SHA-256 above).

---

## 8. Reproducibility Checklist

- [x] `calculator.keystore` present at project root (gitignored).
- [x] `keystore.properties` present at project root with `storeFile`, `storePassword`, `keyAlias`, `keyPassword` (gitignored).
- [x] JDK 17 on `JAVA_HOME`.
- [x] Android SDK at `/home/elijah/Android/Sdk` with `platforms;android-33` and `build-tools;34.0.0`.
- [x] `local.properties` present (gitignored) with `sdk.dir`.
- [x] `./gradlew :app:assembleRelease` succeeds → `BUILD SUCCESSFUL`.
- [x] `apksigner verify` returns exit code `0`.
