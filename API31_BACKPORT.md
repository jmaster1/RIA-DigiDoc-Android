# Android 12 / API 31 compatibility backport

This branch carries a local Android 12 compatibility backport on top of the current RIA DigiDoc source.
The official app currently declares Android 14 (API 34) as its minimum;
this branch lowers the install floor to API 31 for devices such as the Huawei P60 Pro running EMUI 14.2.

This is an unofficial personal build. It is not distributed or supported by RIA, and it does not receive updates from the official app.

## Changes in this branch

- Lower `minSdkVersion` to 31 (Android 12). `compileSdkVersion` and `targetSdkVersion` stay at 37.
- Do not request `POST_NOTIFICATIONS` on Android 12 and earlier. Android 13 and later retain the existing notification permission flow.
- Use the API 33 `PackageInfoFlags` overload only on Android 13 and later; use the legacy overload on Android 12.
- Add `.api31` to the release application ID and `-api31` to its version name. The release package is `ee.ria.DigiDoc.api31`, so it can be installed alongside the official package.
- Restrict the Google Maven repository to Google-owned dependency groups during dependency resolution.

The `minSdkVersion` change only lowers the install check; it does not make APIs introduced after Android 12 available.
Newer platform-specific code paths must remain guarded by runtime API checks.

## Build on Windows

Requirements:

- JDK 21.
- Android SDK Platform 37 and Android Build Tools installed through Android Studio.
- A project-local SDK path: `local.properties` with `sdk.dir=...` or the `ANDROID_HOME` environment variable. `local.properties` is ignored by Git, so configure it separately in each checkout.
- A local `app/google-services.json`. The repository intentionally ignores this Firebase configuration; use a file supplied for your own build or CI setup. Do not commit it.

The Gradle Wrapper allows up to 120 seconds for the initial Gradle distribution download. On a slow connection, retry Sync after the first download has completed; later runs reuse the cached distribution.

In PowerShell, from the repository root:

```powershell
Set-Location C:\my\RIA-DigiDoc-Android

# Point JAVA_HOME at your installed JDK 21 if it is not already configured.
$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.10.7-hotspot'
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
# If Android Studio installed the SDK in the default Windows location:
$env:ANDROID_SDK_ROOT = "$env:LOCALAPPDATA\Android\Sdk"
java -version

# Fetch and verify RIA's signed default configuration, then build the API 31 release.
.\gradlew.bat :config-lib:fetchAndPackageDefaultConfiguration :app:assembleRelease --no-configuration-cache
```

This is one Gradle invocation: it fetches the default configuration from `https://id.eesti.ee` using the repository's configured update interval, then assembles the release APK. Without the generated assets the APK can install but will fail to initialize configuration on first launch. Generated default-configuration assets are ignored by Git; the repository's existing TSL inputs remain tracked.

The release APK is initially unsigned at `app\build\outputs\apk\release\app-release-unsigned.apk`.
To install it locally, align and sign it with a local key.
The following creates a standard debug key only if one does not already exist, then signs the release APK.
If you want to update the `.api31` APK already installed on your phone,
copy the same key used for that APK into this checkout before running these commands;
otherwise Android will reject the update.
Keep the key private and out of Git.
This key is only for local testing, not public distribution.

```powershell
$key = 'app\src\main\assets\keystore\debug.keystore'
if (-not (Test-Path $key)) {
    New-Item -ItemType Directory -Force (Split-Path $key) | Out-Null
    keytool -genkeypair -v -keystore $key -storepass android -keypass android `
        -alias androiddebugkey -keyalg RSA -keysize 2048 -validity 10000 `
        -dname 'CN=Android Debug,O=Android,C=EE'
}

$buildTools = Get-ChildItem "$env:ANDROID_SDK_ROOT\build-tools" -Directory |
    Sort-Object { [version]$_.Name } -Descending |
    Select-Object -First 1 -ExpandProperty FullName
$releaseDir = 'app\build\outputs\apk\release'
& "$buildTools\zipalign.exe" -f -p 4 `
    "$releaseDir\app-release-unsigned.apk" `
    "$releaseDir\app-release-unsigned-aligned.apk"
& "$buildTools\apksigner.bat" sign `
    --ks $key --ks-pass pass:android --key-pass pass:android `
    --out "$releaseDir\RIA-DigiDoc-3.2.0-api31.apk" `
    "$releaseDir\app-release-unsigned-aligned.apk"
```

Verify the signed artifact with `& "$buildTools\apksigner.bat" verify --verbose "$releaseDir\RIA-DigiDoc-3.2.0-api31.apk"` before installing it.

The app's debug build uses the `.debug` package suffix. The release backport uses `ee.ria.DigiDoc.api31`; neither package can update or replace the official `ee.ria.DigiDoc` installation.

## Verification status

The earlier 3.0.5 API 31 backport was installed and verified by the user on a Huawei P60 Pro. This branch is based on newer upstream source; its build and on-device behavior have not yet been verified. Build and test this branch on the target phone before relying on it for signing.
