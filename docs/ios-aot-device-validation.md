# Verifying standalone iOS AOT startup

A successful build and installation do not prove that an iOS release app can
start independently of the development session. Test a cold start on a physical
device after changing the AOT compiler, framework packaging, or signing path.

## Before testing

Record the xcross version or commit, Flutter version, engine revision, compiler
source, build mode, and device/iOS version. Use an AOT compiler and release engine
that match the Flutter SDK used to generate the kernel. Do not mix a release
snapshot with a debug engine.

From the xcross checkout, after `dart pub get`, inspect the built app:

```sh
cd packages/xcross
dart run tool/verify_flutter_aot.dart /path/to/build/xcross-ios --dsym=required
```

The verifier checks the current snapshot symbols, snapshot section digests, a
precompiled engine, absence of JIT-only assets, and the dSYM UUID. It also rejects
an App dylib without any dependent library load command. These structural checks
complement the device test; they do not replace it. The current snapshot-symbol
check targets xcross's direct Mach-O output, not every historical Flutter layout.

## Physical-device acceptance test

1. Build and install the release app using `xcross flutter run --release`.
2. Confirm the first Flutter frame and basic interaction. Capture startup logs
   if diagnosing a failure.
3. End the development run and any debugger session, stop the host tunnel used
   for this test, and disconnect USB. Ensure a wireless development session has
   not remained attached.
4. Fully dismiss the app in the iOS app switcher, then open it from the Home
   Screen. Returning to an already running app is not a cold-start test.
5. Confirm it stays open and exercise a simple app action. Record success only
   after this step, including any features not exercised.

AOT changes how Dart code runs; it does not extend the provisioning profile's
validity. An expired or invalid signing profile is a separate failure.

## Failure before the first Flutter frame

Reconnect and unlock the device, collect the app's crash report and process logs,
then launch once more from the Home Screen while logging. Preserve the original
failing artifact for comparison.

`VM snapshot invalid and could not be inferred from settings` is not by itself
proof of an incompatible compiler. The engine may have failed to load the
framework at all. Check the framework path, its exported snapshot symbols,
signing, and Mach-O load commands. A crash inside engine startup can be a
consequence of the earlier loader failure.

For example, when experimenting with an assembly-based AOT pipeline:

```sh
llvm-objdump --macho --dylibs-used App.framework/App
```

An install name (`LC_ID_DYLIB`) is not a dependency. Apple's
[dyld validation](https://github.com/apple-oss-distributions/dyld/blob/main/common/MachOAnalyzer.cpp)
rejects application dylibs with no dependent libraries under its modern format
checks. When linking snapshot assembly directly with `ld64.lld`, include
`-syslibroot <iOS SDK> -lSystem` even if the snapshot has no undefined symbols.
The verifier checks for a dependency rather than requiring a particular path,
so indirect linkage through another library remains valid.

## Independent assembly-path observation

A Linux-hosted experiment on 2026-10-13 used Flutter 3.44.0, engine
`4c525dac5ebe5971c5708ef73558ed8edcf4a362`, and Dart revision
`98116461144f4429ab873f8497023a5ec3b08127`. A locally built Product
`gen_snapshot` generated iOS ARM64 assembly, which clang 21 and ld64.lld 22.1.8
assembled and linked using iPhoneOS SDK 27.0. The app used xcross signing and
existing Runner/plugin artifacts.

On an iPhone 17 Pro running iOS 27.0.1, the first bundle installed successfully
but failed before the first Flutter frame. Adding the libSystem dependency,
re-signing, and reinstalling allowed startup; the tester then confirmed a full
Home Screen cold start and basic interaction with USB disconnected and no
debugger or JIT tunnel started for the test.

This observation motivates the loader check and acceptance procedure. It does
**not** certify a build made by current xcross: the experiment used a custom
assembly path, while current xcross requests `app-aot-macho-dylib` directly. Nor
does it establish byte identity with the official Mac compiler or exhaustive
runtime equivalence. No app binary, signing material, device identifier, or
unfiltered device log is needed in a validation report.
