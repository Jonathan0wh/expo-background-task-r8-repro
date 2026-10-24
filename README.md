# expo-background-task: SDK 58 Android R8 startup crash

A blank Expo SDK 58 app with `expo-background-task` added crashes on startup in a minified Android release build. Keeping the reflectively created Room and WorkManager constructors fixes the startup crash.

The app uses the unchanged blank-template `App.js`. It does not import the background-task API, define a task, or register a task. It has no Firebase dependency, backend, credentials, or EAS project association.

## Environment

- Expo 58.0.17, React Native 0.88.0
- expo-background-task 58.0.11, expo-task-manager 58.0.12
- WorkManager 2.9.1, Room 2.5.0
- Android Gradle Plugin 9.2.1, Gradle 9.4.1, R8 9.2.14
- Node 24.22.1, JDK 17.0.20.1
- Pixel 9 emulator, Android 15, arm64-v8a
- Expo Doctor: 20/20 checks passed

The package lock preserves the dependencies used in the test. Native versions were also read from the baseline APK; see `evidence/native-versions.json`.

## Reproduce the crash

Prerequisites: an Android SDK and JDK configured for Expo SDK 58, and a running arm64 Android emulator. Run these commands from this directory:

```sh
npm ci
npx expo prebuild --platform android --no-install
cd android
./gradlew :app:assembleRelease --no-daemon --max-workers=2 -PreactNativeArchitectures=arm64-v8a -Pandroid.enableMinifyInReleaseBuilds=true
adb install -r app/build/outputs/apk/release/app-release.apk
adb shell am start -W -n com.example.expobackgroundtaskr8repro/.MainActivity
adb logcat -d -s AndroidRuntime
```

The release build succeeds, but the app exits before rendering:

```text
Unable to get provider androidx.startup.InitializationProvider
Failed to create an instance of class androidx.work.impl.WorkDatabase.canonicalName
```

`evidence/baseline-crash.log` contains the captured stack trace. DEX inspection shows that `WorkDatabase_Impl`, `OverwritingInputMerger`, and `ArrayCreatingInputMerger` have no constructors.

## Apply the workaround

Return to the project root, append the two rules, and rebuild without running prebuild again:

```sh
cd ..
cat workaround.pro >> android/app/proguard-rules.pro
cd android
./gradlew :app:assembleRelease --no-daemon --max-workers=2 -PreactNativeArchitectures=arm64-v8a -Pandroid.enableMinifyInReleaseBuilds=true
adb install -r app/build/outputs/apk/release/app-release.apk
adb shell am force-stop com.example.expobackgroundtaskr8repro
adb shell am start -W -n com.example.expobackgroundtaskr8repro/.MainActivity
```

`workaround.pro` contains:

```proguard
-keep class * extends androidx.room.RoomDatabase { <init>(); }
-keep class * extends androidx.work.InputMerger { <init>(); }
```

R8 remains enabled. The app now renders the blank-template greeting. Three consecutive force-stop/cold-launch checks succeeded, and DEX inspection confirms that all three constructors are present.

This native-file edit is for the reproduction only. Running prebuild again can overwrite it. An Expo app can apply the same rules through `expo-build-properties` and `android.extraProguardRules`.

## Evidence and scope

- `evidence/results.json`: environment, outcomes, and APK checksums.
- `evidence/baseline-crash.log`: startup exception before the workaround.
- `evidence/baseline-constructors.json` and `patched-constructors.json`: DEX constructor comparison.
- `evidence/patched-launches.json`: three successful cold launches, each starting without an existing app process.
- `evidence/patched-screen.png`: the rendered default screen.
- `evidence/patched-runtime.log`: AndroidRuntime error capture during the successful launches; empty means no matching errors were emitted.
- `evidence/expo-doctor.log`: dependency diagnostics.

Only the two constructor keep rules changed between the baseline and patched builds. This project tests native startup initialization, not background-task execution or notification delivery. Debug and unminified builds were not tested in this reproduction.

## Related upstream reports

- https://github.com/expo/expo/issues/51337
- https://github.com/expo/expo/pull/51349

Those reports cover the same WorkManager/Room rule issue through `expo-widgets`. This project isolates the dependency path through `expo-background-task`.
