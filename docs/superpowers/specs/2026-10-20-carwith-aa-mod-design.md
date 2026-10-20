# CarWith Android Auto APK Modification Design

Status: Design draft for user review
Date: 2026-10-20
Owner: User + local research and validation

## Decision

Modify and repackage the user-supplied CarWith APK as the target artifact.
Android Auto is the first target channel. The first protocol experiment is
USB-wired; wireless projection is out of scope until wired interoperability is
proven. The required product outcome remains the complete CarWith UI displayed
by a stock Android Auto head unit, with head-unit input returned to CarWith.

This is an APK modification/repackaging effort, not a source build of Xiaomi's
original project. JADX output is analysis evidence, not a compilable source
tree. The first technical decision after static inspection is whether the APK
can be safely rebuilt and whether its existing application boundaries can host
the required projection-source behavior.

No AADisplay or external adapter/box is part of this design.

## Context and Forces

- Full CarWith UI is mandatory; a category-compliant replacement UI does not
  meet the requested outcome.
- The Android phone is rooted, but no model, Android build, or head-unit
  compatibility matrix has been supplied. Root does not establish AAP
  interoperability or restore a lost signing identity.
- The artifact is closed-source. The workspace contains no original source or
  build project, so a bytecode/resource patch and repackage is the available
  candidate, subject to inspection.
- A stock head unit expects an Android Auto projection session. Replacing
  ICCOA/CarLife transport is not sufficient by itself; session establishment,
  authentication, service discovery, video/audio channels, lifecycle, and
  input return must work together.
- Re-signing may prevent installation over the original package and may remove
  privileges that depend on Xiaomi's signing certificate. This is a
  feasibility gate, not an implementation detail.
- Work is for the user's private research. The supplied filename identifies a
  third-party distribution source, so authenticity and "latest official
  release" are not established by the filename or upload.

## Artifact Record

- Supplied path: `com.miui.carlink_4.0.21-20260929_MemeOSUpdates.com.apk`
- Observed SHA-256: `66aa970c4b0bf92ea814b7c87b26be8900ac0bc02b5426e30045194a19f817cc`
- `file` identified it as an Android package with Gradle app metadata.
- ZIP inventory reported 4,975 entries and 105,078,197 uncompressed bytes.
- The archive contains `META-INF/PLATFORM.RSA`, `META-INF/PLATFORM.SF`, and
  `META-INF/MANIFEST.MF`.
- Signer identity, APK signature validity, package manifest, DEX contents,
  native libraries, and runtime behavior have not been verified.
- The original uploaded APK is an immutable input. Analysis and build outputs
  must use separate directories and retain their own digests.

## Alternatives

| Approach | Fit | Cost and reason |
| --- | --- | --- |
| Patch/repackage CarWith and investigate an in-app AA source | Best fit to the requested APK and full UI | High protocol and signing risk; proceed only through staged proof gates |
| Rebuild from JADX-generated Java | Poor | Decompiled Java is not the original build source and does not provide the original project configuration |
| Rebuild a category-compliant navigation UI | Does not meet the hard requirement | Could be a later fallback only if full projection is abandoned |

## Proposed Architecture

1. **Artifact custody:** Preserve the supplied APK byte-for-byte; record its
   digest, signer observations, package/version declarations, and all tool
   versions used.
2. **Static inspection:** Decode the manifest/resources and inspect DEX classes
   to map CarWith startup, navigation rendering, existing ICCOA/CarLife
   transport boundaries, required permissions, native code, and signature-
   dependent behavior. Static results are hypotheses until runtime-tested.
3. **Build feasibility gate:** Rebuild an unchanged copy in an isolated
   workspace and validate the resulting APK structure and signing. Do not
   uninstall or overwrite the original app. If the package requires the Xiaomi
   platform signer for install or required privileges, stop before implementing
   the AA stack and report the blocker.
4. **AA source prototype:** If the gate passes, add the smallest isolated
   experimental source component to the repackaged CarWith. Start with a
   synthetic video pattern for protocol and channel proof; then connect the
   encoded source to CarWith's actual UI. Verify input return separately.
5. **Device validation:** Use a test device or a user-approved, backed-up
   installation path. Test only while parked. Keep the original package and
   recovery instructions available.

The experimental AA source owns its transport/session state; CarWith remains
the owner of navigation state and UI. A narrow adapter connects captured
CarWith frames to the AA video channel and maps accepted head-unit input back
to the relevant CarWith display. If the APK cannot expose the UI safely or the
head unit rejects the session, the prototype stops rather than silently
falling back to another architecture.

## Scope

In scope:

- Static inspection of this exact supplied APK.
- A reproducible, separate modified APK build if the signing and install gates
  pass.
- USB-wired Android Auto proof of session, video, and input behavior.
- Private, parked-only validation and rollback documentation.

Out of scope:

- CarPlay, wireless projection, or support for multiple phone/head-unit
  combinations in the first prototype.
- Installing software on the head unit or using an external adapter/box.
- Replacing or deleting the original APK, disabling driving restrictions, or
  bypassing protected-content controls.
- Distribution of the modified APK.

## Acceptance Gates

| Gate | Pass condition | Failure response |
| --- | --- | --- |
| Artifact identity | Digest is pinned; package identity and signer observations are recorded | Preserve the input and report what could not be verified |
| Decode/build feasibility | Manifest and DEX are inspectable; unchanged repackaging produces a structurally valid, separately signed APK | Stop before protocol implementation if platform-signature dependencies prevent the required install/runtime behavior |
| AA session | Stock head unit establishes a wired session with the modified phone app | Stop and retain protocol logs/evidence; do not claim compatibility |
| Full UI | CarWith navigation UI renders continuously through the AA video path | No-go for full-screen requirement if only a test pattern or partial UI works |
| Input | Head-unit touch is mapped to the intended CarWith UI and verified on-device | Mark full interactive projection as failed |
| Stability and recovery | At least 30 minutes parked without session loss, app crash, or unrecoverable state; original app remains recoverable | Stop testing, restore the original state, and record the failure |

No performance or real-device result is claimed by this design document.

## Risks and Reversibility

- **Signing/privilege loss:** Re-signing changes package identity of the signer.
  The modified package may not update the official install or receive
  signature-level privileges. Reconsider the route if the unchanged repack
  fails its install/runtime gate.
- **Protocol incompatibility:** Android Auto source behavior may depend on
  undocumented protocol details or head-unit-specific expectations. Keep the
  first implementation isolated and USB-only; abandon it if a minimal session
  cannot be established.
- **UI capture/input mismatch:** CarWith may use protected surfaces, native
  rendering, or display assumptions that prevent capture or correct touch
  mapping. Treat those as explicit no-go outcomes, not reasons to bypass
  platform safeguards.
- **Source rights and terms:** The user's authorization covers their device and
  test activity, but does not itself grant Xiaomi's permission. Keep the work
  private and do not characterize it as officially supported or authorized by
  Xiaomi.
- **Reversal:** All edits, signing keys, and outputs remain separate from the
  supplied APK. Revert by discarding the experimental build and restoring the
  original package from the preserved input/backup. Device installation is a
  separate mutation and requires an explicit test-device/backup plan.

## Verification and Evidence

- Hash the original before and after the work; the values must match.
- Record package/version, signer certificate fingerprints, manifest
  permissions/components, DEX/native inventory, and decoded-tool versions.
- Build an unchanged repack before introducing behavior changes.
- Validate APK structure and signature independently; a successful build alone
  does not prove installability or Android Auto interoperability.
- For each device run, record phone build, Android Auto version, head-unit
  model, transport, start/stop outcome, display result, input result, duration,
  and bounded logs. Do not store unrelated ambient environment data.
- Distinguish static observations, source-derived inferences, device-tested
  results, and unknowns in the progress record.

## Current Blockers

- The workspace has no `rea`, `jadx`, `apktool`, `aapt2`, or `apksigner`
  executable on `PATH`, and the REA MCP tools are not connected in this
  session.
- Java, `jarsigner`, and OpenSSL are available, but they do not substitute for
  APK manifest/DEX decoding or Android APK signature-scheme verification.
- The user reports a REDMI K100 PRO MAX running Android 17 and HyperOS 4, but
  no phone or Android Auto head unit is connected for validation; the head-unit
  model and Android Auto version remain unknown.

The next approved work after spec review is to establish the Android analysis
tool path without changing REA configuration, inspect the APK, and update this
design if the actual manifest, signer, or code structure contradicts its
assumptions.

## Approved Local Feasibility Run

The user approved proceeding with all work that can be verified in the current
workspace, stopping at any step that requires unavailable device access or
authorization.

1. Install analysis utilities into an isolated, non-repository directory from
   their official distribution sources; record source, version, and available
   checksums. Do not change REA configuration.
2. Preserve the supplied APK and confirm its SHA-256 before and after analysis.
   Inspect package identity, signing certificates/signature validity,
   manifest, permissions, components, DEX inventory, resources, and native
   libraries, recording tool versions and results.
3. If decoding permits, rebuild an unchanged copy in a separate output
   directory and independently validate its structure and APK signature. Do
   not install either the original or rebuilt APK on the supplied phone.
4. Stop before behavior changes if APK inspection or unchanged repackaging
   fails, or if required platform-signature privileges cannot be preserved.
   Report evidence, unknowns, and the next decision rather than claiming
   Android Auto compatibility.
5. Any later projection prototype requires a separate feasibility decision and
   connected test equipment. Validate only while parked; do not disable driving
   restrictions or bypass protected-content controls. Wired and wireless
   support require separate device evidence.
