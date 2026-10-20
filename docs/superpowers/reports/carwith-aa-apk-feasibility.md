# CarWith APK Local Feasibility Report

## Result

Local APK feasibility checks completed. The supplied APK is structurally valid,
its APK Signature Scheme v3 signature verifies, and an unchanged Apktool rebuild
can be independently signed and verified with a temporary test key. No
behavior-modified APK was produced or installed. Android Auto projection is
blocked at the current gate: the package is Xiaomi/MIUI-signed, declares
platform-level permissions, and the inspected connection path is CarLife/
Carlink rather than an Android Auto projection source.

## Environment and Tool Provenance

| Tool | Version | Source and integrity |
| --- | --- | --- |
| Java | Eclipse Temurin 25.0.4.1 LTS | Preinstalled in the workspace |
| Apktool | 3.0.3 | [Official GitHub release](https://github.com/iBotPeaches/Apktool/releases/tag/v3.0.3); downloaded JAR SHA-256 `dbf930b076c6b9be08d57c449cacefc3bdd6b71ebd59b3066fc0e1f5b14f9423`, matching the GitHub release API digest |
| JADX | 1.5.6 | [Official GitHub release](https://github.com/skylot/jadx/releases/tag/v1.5.6); downloaded ZIP SHA-256 `545ea2be9c242511bc145755cf4bda2485ade42966e096f8b4d3da2a230e8974`, matching the GitHub release API digest |
| apksig | 9.4.1 | [Google Maven artifact](https://dl.google.com/dl/android/maven2/com/android/tools/build/apksig/9.4.1/apksig-9.4.1.jar); SHA-256 `7ae2e5980c77d853e3513074ee7c822bbdcdcde1668889d17faa7fe8bc8aa821`, matching the published sidecar; POM specifies Apache License 2.0 |
| Android command-line tools | 20.0 | Downloaded from the [official Android Studio tools page](https://developer.android.com/studio) and extracted under `/tmp`; the page's 40-hex checksum matched SHA-1 `48833c34b761c10cb20bcd16582129395d121b27` (the page labels the column SHA-256 despite its 40-character value). SDK build-tools were not installed and no SDK license was accepted. |

Downloaded tools, decoded trees, rebuilt APKs, and a temporary PKCS#12 test
keystore are isolated under `/tmp/carwith-aa-tools/` and
`/tmp/carwith-aa-work/`. The SDK build-tools `apksigner`, `zipalign`, and
`aapt2` were not installed; resource rebuilding used Apktool's bundled aapt2,
and signature validation/signing used the Apache-licensed apksig library.

## Supplied Artifact

- File: `com.miui.carlink_4.0.21-20260929_MemeOSUpdates.com.apk`
- SHA-256 before and after analysis: `66aa970c4b0bf92ea814b7c87b26be8900ac0bc02b5426e30045194a19f817cc`
- ZIP integrity: passed; no compressed-entry errors.
- Package: `com.miui.carlink`
- Version: `4.0.21-20260929`, version code `104000021`
- SDK declarations: min 29, target 33, compile 34
- Code/native inventory: 8 DEX files and 30 native libraries, all for `arm64-v8a`
- APK signature: apksig verified successfully with v3; v1, v2, and v3.1 are absent; no apksig errors or warnings.
- Signer certificate: subject `EMAILADDRESS=miui@xiaomi.com, CN=MIUI, OU=MIUI, O=Xiaomi, L=Beijing, ST=Beijing, C=CN`; SHA-256 fingerprint `C9:00:9D:01:EB:F9:F5:D0:30:2B:C7:1B:2F:E9:AA:9A:47:A4:32:BB:A1:73:08:A3:11:1B:75:D7:B2:14:90:25`. The certificate is self-issued and uses SHA1withRSA for its certificate signature; its subject alone does not independently prove distribution authenticity.

## Static Findings

- The manifest requests platform or MIUI capabilities including `CAPTURE_VIDEO_OUTPUT`, `CAPTURE_SECURE_VIDEO_OUTPUT`, `READ_FRAME_BUFFER`, `INTERNAL_SYSTEM_WINDOW`, `ADD_TRUSTED_DISPLAY`, `READ_PRIVILEGED_PHONE_STATE`, and `MODIFY_PHONE_STATE`.
- The manifest defines a `signature|privileged` permission and declares a `signatureOrSystem` permission protecting the Carlink service. Re-signing with the local test key cannot reproduce the original Xiaomi signer identity or its signature-level grants.
- The USB accessory entry point is `AOAActivity`; inspected branches select CarLife/Baidu or Carbit/Carlink flows. `CarlifeConnService` exposes CarLife AIDL cast callbacks, and its inspected caller check only accepts `com.baidu.carlife.xiaomi` or `com.baidu.carlife`.
- CarLife touch return is implemented through `CarlifeUibc`, which maps touch coordinates and injects input into a selected display. This is evidence for an existing CarLife path, not evidence of Android Auto compatibility.
- Apktool decoded resources, the binary manifest, and all 8 DEX files into smali. JADX produced source output but exited with 85 method-generation errors; observed examples affect generated `com.miui.carlink.R.*` constructors. Decompiled source is partial analysis evidence, not a compilable source tree.
- No Android Auto session, registered Android Auto projection service, head-unit session, video channel, or Android Auto input path was verified.

## Rebuild Verification

- Apktool 3.0.3 decoded the original APK successfully and rebuilt an unsigned copy without behavior changes at `/tmp/carwith-aa-work/rebuilt/unaligned-unsigned.apk`.
- A new, temporary local-only test certificate was used to sign a separate copy at `/tmp/carwith-aa-work/rebuilt/local-test-signed.apk`; the original Xiaomi-signed APK was never altered.
- apksig verified the rebuilt sample with v3 and reported no errors; ZIP integrity validation passed. The test signer is not Xiaomi's signer and this does not establish installability, privileged-permission access, or Android Auto operation.
- The original APK SHA-256 remained unchanged at the pinned value.

## Gate Decision and Remaining Blockers

**Local decode and unchanged rebuild: passed. Android Auto modification/install gate: no-go pending a viable authorized runtime path.** The supplied build is signed by a Xiaomi/MIUI certificate and declares protected platform capabilities; the corresponding private signing key is not available, and a test signature cannot preserve those privileges. No behavior patch was made because static evidence does not establish that the required projection/capture/input behavior can run after re-signing.

The REDMI K100 PRO MAX / Android 17 / HyperOS 4 configuration is user-reported but was not connected to this environment. The head-unit model, Android Auto version, and physical USB/wireless test path are unknown. Wired and wireless session establishment, full-screen UI, touch return, and 30-minute stability remain entirely untested. Do not treat the rebuilt test APK as installable or compatible.
