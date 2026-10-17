# ClearCut Gate-1 Acceptance — Build-Readiness Evidence (prepared area)

## Local area state
- Pinned working tree: `E:\clearcut-gate1-preflight\src` — detached HEAD =
  `3b470a2e87058d78a065d49e87a3100140c3ae43` = "chore(release): v3.82.0" (tag `v3.82.0` points at this commit). Unmodified since clone (git status clean at every recorded moment).
- Temporary patch-building tree `E:\clearcut-gate1-preflight\patchgen` was a full copy of `src`; used only for diff generation and clean-apply verification. It was reverted to a clean state after verification and is safe to delete (see cleanup section).
- Deliverables live under `E:\clearcut-gate1-preflight\prepared\`.

## Provenance identities (recorded now)
| Identity | Value |
|---|---|
| Upstream source commit (pinned) | `3b470a2e87058d78a065d49e87a3100140c3ae43` (tag v3.82.0) |
| Build-driver commit SHA | recorded at the CI run as `github.sha` (workflow joins it into provenance.json) |
| Acceptance patch SHA-256 | `35704cd0ae75d59e85d01c0198cb2f019b9dd86ffd5c18fa2b3de7fa98b75981` (3602 bytes, UTF-8, produced via temporary Git index `git add -N` so the new file is included; `git apply --check` verified on a pristine tree) |
| Patched `app/build.gradle.kts` SHA-256 | `43ad32d5985a7e74c516ded9e9f0d685c81cf1e515ab9088c682b4946d31f2af` |
| New `app/src/acceptance/res/values/strings.xml` SHA-256 | `3ef7372f13b51491d98326dabdff73ab97b84561d7dd291c5a22a24144345302` |
| Vendored FFmpegKitNext AAR SHA-256 (recomputed locally) | `7971240aff84ce59a4ab28400bb4af59d24c20ce68c25525d41d910246ccff62` — equals both the `verifyFfmpegKitAar` pin and `native-lock.json` artifact sha256 |
| wrapper JAR SHA-256 | see below |

## Toolchain verification (no local Gradle; no version changes)
1. **Gradle 9.8.0 distribution**: downloaded `gradle-9.8.0-bin.zip`, computed SHA-256 =
   `bafd5ce9cfaea0fbccfdc8439a1ac42fbd4cd9c89dc9a988228d8a2639a58e6c` — matches the pin in
   `gradle/wrapper/gradle-wrapper.properties` (`distributionSha256Sum`).
2. **Wrapper JAR**: vendored `gradle/wrapper/gradle-wrapper.jar` SHA-256 =
   `498495120a03b9a6ab5d155f5de3c8f0d986a449153702fb80fc80e134484f17` — found on the Official
   Gradle release checksums page (https://gradle.org/release-checksums/), i.e. it is a
   published, verifiable Gradle wrapper JAR. Additional reference digests recorded during the
   check (NOT used as gates):
   - `gradle/gradle` tag `v9.8.0` `gradle/wrapper/gradle-wrapper.jar` = `238e777fcddd7e34f9708186085def2abd6e08e658505b38718d79d74c21abd5` (build-time variant; differs by design).
   - Inside the checksum-verified distribution zip: `gradle-9.8.0/lib/gradle-wrapper-shared-9.8.0.jar` (38.2 KB) and `lib/plugins/gradle-wrapper-main-9.8.0.jar` (47.6 KB) — Gradle 9 splits the wrapper internals; the standalone project wrapper JAR is the authoritative artifact and is validated above.
3. **AGP 9.4.1**: `com.android.tools.build:gradle:9.4.1` confirmed on the Official Google Maven
   repository (POM + Gradle Module Metadata fetched). Module metadata declares
   `org.gradle.jvm.version: 17` → AGP 9.4.1 runs on JDK 17+; runner uses Temurin JDK 21.
4. **Kotlin compose plugin 2.4.10**: `org.jetbrains.kotlin.plugin.compose.gradle.plugin:2.4.10`
   POM confirmed on the Gradle Plugin Portal (https://plugins.gradle.org/).
5. **JDK runtime range**: official Gradle 9.8.0 release notes state Gradle supports running the
   daemon on Java through 27 (JDK 21 is safely inside the supported range).
6. **Android SDK**: compileSdk 37; runner installs `platforms;android-37` and
   `build-tools;37.0.0` via `sdkmanager` (licenses accepted). AGP fetches any remaining
   components itself, all covered by `gradle/verification-metadata.xml` (12,328 lines sha256
   verification active once the file exists in `gradle/`; enforced by Gradle on download).
7. No version in the toolchain was changed (explicitly NOT upgraded/downgraded).

## Changed-file integrity (correction 5)
- Patch was produced with a temporary Git index (`git add -N` = intent-to-add) so that the
  untracked new `strings.xml` appears in `git diff --name-only HEAD`.
- Recorded check: `git status --porcelain` shows exactly:
  ` M app/build.gradle.kts` and ` A app/src/acceptance/res/values/strings.xml`.
- Permitted source changes, exactly two files (neither modifies application behavior):
  1. `app/build.gradle.kts` (patched in place)
  2. `app/src/acceptance/res/values/strings.xml` (new resource-only file)
- The same acceptance workflow re-asserts both with the temporary-index technique before running Gradle and fails closed otherwise.

## Gate-by-gate security review (correction 4)
| Gate (upstream wiring) | Carried by acceptance lane | Reason / gap |
|---|---|---|
| `verifyFfmpegKitAar` (preBuild, every variant) | YES | Vendor AAR checksum gate runs for acceptance automatically (app/build.gradle.kts:516-519). |
| `generateRuntimeOpenSourceNotices` (preBuild, every variant) | YES | Runtime license/notices gate; no network, no Python; fails the build on stale curated versions. |
| `generateNativeSbom` (`python scripts/verify_native_supply_chain.py`, preReleaseBuild only) | YES (preserved deliberately) | We wired it to `preAcceptanceBuild` as well. Verified the script uses stdlib only (argparse/copy/datetime/hashlib/json/tempfile/pathlib), performs no network calls (verified by import/usage grep incl. no urlopen/requests hits). ubuntu-latest provides python3; **no pip packages required**. |
| `verifyResolvedAdvisoryFloors` (preReleaseBuild only) | YES (preserved deliberately) | Pure Gradle resolution gate (protobuf advisory floor + kotlin-stdlib resolved-version policy + wrapper checksum presence). Wired to `preAcceptanceBuild`. |
| `verifyReleaseSigningIdentity` (preReleaseBuild only) | NO — reason: upstream's pinned release certificate does not apply to our separate acceptance signer; enforcement remains untouched for the real release lane (no task exclusions, no verification disabling). **Gap:** the acceptance signer has no pinned in-tree identity; compensating control = the workflow records the acceptance APK's signing-certificate SHA-256 via `apksigner verify --print-certs` into provenance.json; the keystore retention plan (KEY_SETUP_AND_RETENTION.md) governs future updates. |
| `lint/lintVital` | NO | Lint is not part of `assemble*` in either lane (upstream release lane runs it separately); we deliberately do not add it to the acceptance build (out of Gate-1 scope). Gap: none relative to the upstream assemble lane. |
| APK-size / 16 KB alignment / distribution-readiness / public-claims scripts (`scripts/*.py`) | NO | Not wired into any `pre*Build` by upstream (they were release-pipeline-side scripts; upstream carries no `.github` workflows). They would add Pip deps (`opentimelineio`) and new tooling — reported here rather than silently installed. Gap: release-distribution-only checks keep running in the upstream release lane only. |
| Gradle wrapper distribution checksum | YES | Present upstream (distributionSha256Sum), unchanged; re-verified. |

## Provenance caveats (unchanged upstream facts)
- The FFmpegKitNext AAR is **vendored, prebuilt** (59.36 MB). Its SHA-256 pin proves the exact
  artifact being compiled; it does NOT prove the binary was built by us from the pinned
  sources (`arthenica/ffmpeg-kit-next@3e223118…`, FFmpeg 8.1.2 rev `38b88335…`,
  security patch SHA `0433ae68…`, built 2026-08-02, vendor review-until 2026-10-31 noted).
  GitHub building does not convert it into source-build, and this report claims otherwise
  nowhere.
- ONNX Runtime 1.28.0 / MediaPipe tasks-vision 1.0.0 / DeepFilterNet 0.0.8 / smart-exception
  0.2.1 are Maven prebuilts with bundled native code; covered by Gradle dependency
  verification (sha256, verify-metadata=true) in `gradle/verification-metadata.xml`.
- The Gradle wrapper JAR itself is verified (see above) against the official Gradle checksum
  list, and the wrapper's downloaded distribution is checksum-pinned.

## Remote-verification artifacts
Fetched/downloaded during preflight (toolchain verification only; nothing executed):
`gradle-9.8.0-bin.zip` (130 MB, distribution), the two wrapper internals extracted from it,
the `gradle/gradle v9.8.0` tag wrapper jar, Google Maven POM/module files, Gradle jars via
Invoke-WebRequest. Raw binaries are deleted after hashing (see cleanup) — they carry no
further value for the prepared driver.

## Cleanup planned on this workstation (within E:)
- `E:\clearcut-gate1-preflight\patchgen` — full source copy used to generate the patch; now
  reverted to clean and can be removed (deleted in this session's cleanup step).
- `E:\clearcut-gate1-preflight\prepared\remote-verify` — downloaded distribution and
  extracted wrapper internals (all hashes recorded above); deleted after this evidence file
  is written.
- Nothing outside `E:\clearcut-gate1-preflight` was created or modified; the Demi/Concat
  repository and `Turkce-Ses` were not touched; no Gradle was executed locally; no
  repositories, keys, commits, pushes, CI runs, builds, or installations were performed.

## Acceptance configuration summary (see gate1-acceptance.patch)
- Build type `acceptance`: initWith(release) — minify + shrinkResources + the same proguard
  rules; `applicationIdSuffix = ".acceptance"`; `versionNameSuffix = "-acceptance"`;
  `QA_TIMELINE_HARNESS_ENABLED=false`; signingConfig = dedicated acceptance keystore (env-only
  ACCEPTANCE_* secrets); `ndk { abiFilters = mutableSetOf("arm64-v8a") }` → arm64-v8a ONLY. The
  `splits` block is unchanged for the release lane and simply switched off when acceptance
  tasks are requested, producing exactly one arm64-only APK (no universal, no other-ABI APKs).
- App name override "ClearCut Acceptance" via the new `app/src/acceptance/res/values/strings.xml`
  (`app_name` is defined exactly once in `res/values/strings.xml`; no locale-specific override
  exists, so the acceptance value is authoritative).
- Runtime behavior is unchanged: no BuildConfig fields changed for the app logic, no certificate
  or package-name checks exist in app code (the FileProvider/authorities and the diagnostics
  derive from `BuildConfig.APPLICATION_ID`, so they follow the suffix safely); no editor/export/codec/persistence/networking code touched.
- The APK the workflow discovers is whatever the build emits under `app/build/outputs/apk/acceptance/` — the filename is NOT assumed; the workflow asserts exactly one APK and records its name + SHA-256 + package badging (`package: name='com.novacut.editor.acceptance' ...` line from aapt2).
