# CarWith APK Local Feasibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a reproducible, evidence-backed local feasibility result for the supplied CarWith APK without changing or installing the original artifact.

**Architecture:** Place APK analysis tools and working artifacts outside the repository, then inspect the immutable APK and attempt an unchanged decode/rebuild in separate scratch directories. Record any shared tool cache use as an exception to isolation. Save versions, checksums, signer/manifest/DEX findings, rebuild outcome, and remaining device-dependent questions in a concise report; stop before behavior patches if a gate fails.

**Tech Stack:** Java already present; official JADX and Apktool distributions; Android SDK build-tools (`aapt2`, `zipalign`, `apksigner`); `curl`, `unzip`, and `sha256sum`.

---

### Task 1: Establish isolated tool environment

**Files:**

- Create outside repository: `/tmp/carwith-aa-tools/`
- Create outside repository: `/tmp/carwith-aa-work/`
- Create report: `docs/superpowers/reports/carwith-aa-apk-feasibility.md`

- [x] **Step 1: Record baseline and ensure scratch directories are outside the repository**

Run:

```bash
pwd
uname -a
java -version
mkdir -p /tmp/carwith-aa-tools /tmp/carwith-aa-work
```

Expected: repository root is `/workspace`, Java version is recorded, and both temporary directories are created under `/tmp`.

- [x] **Step 2: Resolve official tool releases and record provenance**

Use proxy-aware HTTPS downloads from the official Android SDK, JADX, and Apktool release locations. Select Linux-compatible distributions, record exact release versions and source URLs, and verify published checksums where available before extracting into `/tmp/carwith-aa-tools/`. Do not alter REA configuration or install tools into the repository.

Expected: `jadx`, `apktool`, `aapt2`, `zipalign`, and `apksigner` are available from the isolated tool directory, or a specific download/compatibility blocker is recorded. Execution note: JADX, Apktool, Android command-line tools, and Apache-licensed apksig were downloaded and integrity checked; Google SDK build-tools were not installed and their license was not accepted. Apktool's bundled aapt2 rebuilt resources, and the apksig library provided APK signing/verification.

- [x] **Step 3: Record tool versions**

Run each installed tool's version command and save its output and resolved release/source/checksum information in the feasibility report.

Expected: the report identifies the exact tool builds used, not merely executable names.

### Task 2: Inspect the immutable APK

**Files:**

- Input, read-only: `com.miui.carlink_4.0.21-20260929_MemeOSUpdates.com.apk`
- Output: `/tmp/carwith-aa-work/`
- Modify: `docs/superpowers/reports/carwith-aa-apk-feasibility.md`

- [x] **Step 1: Verify artifact identity before analysis**

Run:

```bash
sha256sum com.miui.carlink_4.0.21-20260929_MemeOSUpdates.com.apk
```

Expected: SHA-256 equals the pinned design value `66aa970c4b0bf92ea814b7c87b26be8900ac0bc02b5426e30045194a19f817cc`; otherwise stop and report the mismatch.

- [x] **Step 2: Verify signatures and extract package metadata**

Run APK Signature Scheme verification and certificate fingerprint extraction against the original APK; decode the manifest with Apktool. Do not accept an SDK license to install build-tools without explicit authorization.

Expected: report package name, version, signer certificate fingerprints, signature-scheme verification results, declared permissions, exported components, and any tool errors. Execution used apksig 9.4.1 `ApkVerifier` APIs because the standalone library has no CLI; it verified original v3 signature and emitted no errors/warnings. Do not infer that embedded `META-INF` files alone prove APK Signature Scheme validity.

- [x] **Step 3: Decode resources and inspect code/native inventory**

Run Apktool decode and JADX decompile into separate directories under `/tmp/carwith-aa-work/`; inventory DEX files, native libraries, resources, services, receivers, providers, and references to CarLife/ICCOA/projection-related classes.

Expected: record concrete static observations separately from hypotheses; do not execute the APK or treat decompiled Java as original source.

- [x] **Step 4: Recheck the original artifact**

Run `sha256sum` on the original APK again.

Expected: digest still equals the pinned design value; otherwise stop and investigate before any rebuild.

### Task 3: Test unchanged rebuild feasibility

**Files:**

- Input, read-only: original APK and decoded scratch tree
- Output: `/tmp/carwith-aa-work/rebuilt/`
- Modify: `docs/superpowers/reports/carwith-aa-apk-feasibility.md`

- [x] **Step 1: Rebuild without behavior changes**

Use Apktool to build the decoded tree into an unsigned APK under `/tmp/carwith-aa-work/rebuilt/`; if required, align it with the isolated Android `zipalign`.

Expected: record the exact command, exit status, and build errors. If decoding or rebuilding fails, stop this task and document the blocker rather than patching around it.

- [x] **Step 2: Sign only the rebuilt copy with a throwaway test key**

Generate a test-only keystore under `/tmp/carwith-aa-work/` and sign only the rebuilt copy using the Apache-licensed apksig `ApkSigner` API; never overwrite or resign the supplied APK.

Expected: a separate APK is produced with a newly recorded test signer; the key remains outside the repository and is not presented as Xiaomi's signing identity.

- [x] **Step 3: Independently validate rebuilt package**

Run apksig `ApkVerifier` and ZIP integrity validation on the rebuilt APK; use the Apktool-decoded manifest and package metadata for the resource/manifest inspection.

Expected: record whether structure and signature checks pass. A passing local signature check does not prove installability, platform privileges, or Android Auto interoperability.

### Task 4: Finalize evidence and decision

**Files:**

- Modify: `docs/superpowers/reports/carwith-aa-apk-feasibility.md`
- Read-only verification: original APK and temporary outputs

- [x] **Step 1: Record the go/no-go decision**

Summarize artifact identity, tool provenance, package/signer/manifest/code findings, unchanged rebuild results, signing limitations, and what remains unknown because no phone/head unit is connected.

Expected: choose one explicit result: `local feasibility passed`, `blocked before rebuild`, or `blocked at rebuild/signature validation`; state that no Android Auto session, display, touch-return, wired/wireless, or stability test has been performed.

- [x] **Step 2: Verify repository scope**

Run:

```bash
git status --short
sha256sum com.miui.carlink_4.0.21-20260929_MemeOSUpdates.com.apk
```

Expected: the original APK digest is unchanged and only intended report/plan files plus any pre-existing user data appear as repository changes. Downloaded tools, decoded files, rebuilt APKs, and keystore material are under `/tmp`; Apktool also logged use of `/root/.local/share/apktool/framework/1.apk`, whose provenance and whether this run changed it were not verified.

### Execution result

The local decode and unchanged rebuild passed. Original APK verification passed with v3 only; the separate test-signed rebuild passed v3 verification and ZIP integrity checks. The original APK hash remained unchanged. Static evidence shows Xiaomi/MIUI signer identity and platform-level permissions, so behavior modification stopped before an AA prototype. No phone or head unit was connected, and no projection or runtime test was performed. JADX produced 85 method-generation errors, so its Java output is partial evidence.

### Stop conditions

- Stop if a published tool artifact cannot be authenticated sufficiently, the pinned APK digest differs, analysis tools cannot parse the package, or unchanged rebuilding/signing fails.
- Do not patch application behavior, install an APK on the phone, or attempt to suppress Android Auto safety restrictions in this feasibility run.
- Wired and wireless projection, head-unit input, and stability require separately approved device testing with connected equipment while parked.
