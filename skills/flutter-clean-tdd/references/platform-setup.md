# Platform setup checklist

Ask which platforms are targeted first. Configure each selected one, then list external steps in the setup report. Verify each requirement against the current docs of the package involved - minimum versions change.

## All platforms
- Bundle/application ID set consistently (`--org` on create, then per platform).
- Env config via `--dart-define-from-file=env/<flavor>.json`; add env files with secrets to `.gitignore`; provide `env/example.json`.
- App name, icons and launch screen (report as external if assets aren't provided).

## Android
- `android/app/src/main/AndroidManifest.xml`: add `<uses-permission android:name="android.permission.INTERNET"/>` - release builds have no network without it (debug manifest hides this bug).
- Check `minSdk` meets every plugin's requirement (ObjectBox, secure storage).
- Release signing: `key.properties` + keystore, never committed. External step: create keystore, explain `keytool` command.
- Cleartext HTTP only for local dev, via a debug-only network security config.
- External: Google Play Console app, signing key / Play App Signing.

## iOS
- Deployment target set in both `ios/Podfile` (`platform :ios, 'X'`) and the Xcode project to meet plugin minimums; run `cd ios && pod install`.
- `Info.plist` usage-description strings for every permission requested (camera, location, photos...). Missing strings crash on request or fail review.
- External: Apple Developer account, Team selection in Xcode (Signing & Capabilities), App Store Connect record, capabilities (Push, Sign in with Apple, Keychain sharing).

## macOS
- `macos/Runner/DebugProfile.entitlements` AND `Release.entitlements`: add `com.apple.security.network.client` = true, or Dio calls fail silently in the sandbox.
- ObjectBox: add an application group entitlement and pass `macosApplicationGroup` to `openStore`.
- Deployment target in `macos/Podfile` and Xcode.

## Web
- ObjectBox is not supported. Provide a web `LocalDataSource` implementation (hive_ce or sembast) chosen by a provider using `kIsWeb`, behind the same interface. Confirm with the user before adding.
- CORS must be allowed by the backend - external step.

## Windows / Linux
- ObjectBox native libraries ship via `objectbox_flutter_libs`.
- Linux: list required system packages for building (clang, cmake, ninja-build, pkg-config, libgtk-3-dev) in the report.
- Secure storage on Linux needs libsecret - report it.
