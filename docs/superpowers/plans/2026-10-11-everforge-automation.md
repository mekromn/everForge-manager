# EverForge Automation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a device-native EverForge release that discovers original APK and patch updates, preserves complete enrolled recipes, recovers interrupted jobs, and installs validated updates under each app's chosen policy.

**Architecture:** Add a Room-backed automation subsystem around Morphe's existing patch/sign/split engine and installers. Immutable run snapshots and a global execution lease connect strict planning, source providers, recoverable workers, and installation. UI and notifications observe persisted state.

**Tech Stack:** Kotlin, Compose, Koin, Room 2.8.5, WorkManager 2.12.0, Ktor 3.6.0, Morphe patcher 1.15.1; retain the repository's existing version catalog except the additions specified below. JVM tests use the existing Kotlin/JUnit dependency; database/worker/device tests use Android instrumentation.

**Spec:** [Approved EverForge design](../specs/2026-10-11-everforge-automation-design.md).
**Status:** Proposed implementation plan; design approved by the user on 2026-10-11. Execution method and plan review are pending.
**Base:** `4ba5d37030136e48fa13c31158dddee59fb86864`; product base `da8545c92c94b83e23adf7b573b46b282fa2b062`.
**Execution recommendation:** Native execution in this session, followed by one independent whole-branch review. Tasks share model, snapshot, and worker contracts; one implementer can carry those changes consistently across the existing flows.

## Global Constraints

These lines are copied from the approved spec; every task includes them:

- All download and patch work runs on the Android device. There is no mandatory account or hosted build service.
- Android 8 and newer remain supported; the main acceptance device is an unrooted Pixel 9 Pro XL on Android 16.
- Stable app and patch releases are the default. Experimental releases require a separate per-app choice.
- Existing Morphe features remain available. Root mount continues as a manual flow; unattended root mount is outside the first release.
- Check every six hours; offer hourly, six-hourly, daily, and manual schedules.
- Unmetered downloads by default; explicit mobile-data opt-in.
- Require battery-not-low and storage-not-low for automatic patch work; charging-only is configurable.
- Retry transient download/network failures with exponential backoff, capped at three automatic attempts for an unchanged fingerprint. Source/patch/recipe changes or an explicit retry permit another attempt.
- Release application ID: `app.everforge.manager`; debug suffix remains separate. Morphe can coexist.
- Preserve GPL licensing, copyright notices, Morphe attribution, and the existing NOTICE. Identify the modified program clearly.
- Keep Morphe patch feeds and library coordinates unless explicitly changed. Upstream patch authors remain responsible for their patch releases.

Never reduce an enrolled recipe to get an automatic build through. Do not automatically uninstall on conflict, downgrade, rotate a signing key, force incompatible versions, or install a stale recipe revision. A retained APK is recovery material, not proof that downgrade preserves data.

## Review Focus

These five additional cases have assigned tests below:

1. Long version codes beyond `Int.MAX_VALUE`: compare as Long; a matching version name must not hide a different build (Task 3).
2. Imported keystore containing several aliases: use the active signing alias, not any certificate the keystore happens to contain (Task 8).
3. Source server returns HTML or changes ETag during a range resume: reject the HTML and restart changed content (Task 4).
4. Split archive repeats a split or contains an escaping entry path: reject it before extraction or patching (Task 4).
5. User edits a recipe or changes clocks while an install/retry is pending: block stale commits; clock changes cannot steal a live lease (Tasks 2, 7, 8).

## File Structure and Contracts

All file paths in this plan are repository-relative. New subsystem files live together under `app/src/main/java/app/morphe/manager/domain/automation/`; persistence adapters live under `app/src/main/java/app/morphe/manager/data/room/automation/`.

| Unit | New files and ownership |
| --- | --- |
| Domain records/results | `domain/automation/AutomationRecipe.kt`, `AutomationRun.kt`, `AutomationResult.kt`, `ApkCandidate.kt`, `PatchSnapshot.kt` |
| Persistence | `data/room/automation/AutomationEntities.kt`, `AutomationDao.kt`; `domain/automation/AutomationRepository.kt`, `RoomAutomationRepository.kt`, `AutomationFiles.kt` |
| Enrollment/planning | `AutomationEnrollment.kt`, `RecipeAssets.kt`, `PatchSnapshotBuilder.kt`, `StrictOptionCodec.kt`, `AutomationPlanner.kt` |
| Acquisition | `sources/ApkSourceProvider.kt`, `SourceProviderRegistry.kt`, `AutomationDownloader.kt`, `ArtifactVerifier.kt`, `SavedOriginalSourceProvider.kt`, `GitHubSourceProvider.kt`, `ManifestSourceProvider.kt`, `FixedUrlSourceProvider.kt`, `ApkMirrorSourceProvider.kt`, `UptodownSourceProvider.kt`, `MirrorPageParser.kt` |
| Execution/recovery | `AutomationPatchExecutor.kt`, `AutomationRunner.kt`, `AutomationRecovery.kt`, `AutomationRetryPolicy.kt`; `app/src/main/java/app/morphe/manager/worker/AutomationRunWorker.kt` |
| Installation | `AutomationInstaller.kt`, `AutomationInstallReceiver.kt`, `RecipeInstallGate.kt` |
| Scheduling/presentation | `AutomationScheduler.kt`, `AutomationChecks.kt`, `AutomationRetention.kt`; worker, viewmodel, UI, and notification files named in their tasks |

The abbreviated filenames in the table inherit the full directory stated in their row. Task file lists identify existing modification sites.

Define these domain types in Task 2; subsequent tasks use the same names:

- `ReleaseChannel`: `STABLE` or `EXPERIMENTAL`; `InstallMode`: `PREPARE_ONLY` or `AUTO_INSTALL`; `CheckInterval` exposes `minutes: Long?`: `HOURLY` (60), `SIX_HOURLY` (360), `DAILY` (1440), or `MANUAL` (null/no periodic work).
- `SourceConfiguration`: serializable `SavedOriginal`, `GitHub(owner: String, repository: String, assetPattern: String)`, `Manifest(url: String)`, `Mirror(providerId: String, startUrl: String)`, or `FixedUrl(url: String)`. A fixed URL discovers only its inspected build.
- `AutomationPolicy`: `channel: ReleaseChannel = STABLE`, `installMode: InstallMode = PREPARE_ONLY`, `interval: CheckInterval = SIX_HOURLY`; `allowMobileData = false`, `chargingOnly = false`, `batteryNotLow = true`, and `storageNotLow = true`.
- `SourceIdentity(uid: Int, endpoint: String?, localBundleHash: String?)` binds an enrolled source independently of its display name.
- `AutomationRecipe`: immutable UUID `id: String`, `revision: Long`, `originalPackageName: String`, `outputPackageName: String`, `isClone: Boolean`, `patches: Map<Int, Set<String>>`, complete encoded `options` and option type descriptors, `sourceIdentities: Map<Int, SourceIdentity>`, `sourceConfiguration: SourceConfiguration`, `originalSignerSha256: Set<String>`, `activeSignerSha256: String`, `enabled: Boolean`, and `policy: AutomationPolicy`. Encoded options use `Map<Int, Map<String, Map<String, Option.SerializedValue>>>`.
- `AutomationRun`: immutable `id` and fingerprint; recipe ID/revision; recipe/bundle/asset snapshot; `stage: RunStage`; worker/session IDs, attempt counters, download checkpoint, target, artifact/receipt, blocker and timestamps.
- `RunStage`: `QUEUED`, `RESOLVING`, `DOWNLOADING`, `VALIDATING_ORIGINAL`, `PATCHING`, `SIGNING`, `VALIDATING_OUTPUT`, `READY_TO_INSTALL`, `INSTALLING`, `INSTALLED`, `WAITING_CONSTRAINTS`, `WAITING_PATCHES`, `SOURCE_ATTENTION`, `WAITING_PERMISSION`, `WAITING_USER_ACTION`, `FAILED`, or `CANCELED`.
- `AutomationEvent(runId: String, atMillis: Long, stage: RunStage, message: String)` and `DownloadProgress(bytes: Long, total: Long?)` are serializable records; progress is not an installation receipt.
- `AutomationResult<T>`: `Success(value: T)`, `Blocked(blocker: AutomationBlocker)`, `TransientFailure(message: String, retryAfterMillis: Long?)`. Cancellation propagates as cancellation.
- `AutomationBlocker(code: BlockerCode, detail: String)`; codes include `MISSING_PATCH`, `INVALID_OPTIONS`, `MISSING_ASSET`, `UNSUPPORTED_VERSION`, `SOURCE_ATTENTION`, `INVALID_ARTIFACT`, `SIGNER_MISMATCH`, `RECIPE_CHANGED`, `INSTALL_PERMISSION`, `USER_ACTION`, and `EXECUTION_BUSY`.
- `DeviceProfile(sdk: Int, abis: List<String>)`; `CandidateQuery(packageName: String, requiredVersions: Set<String>, channel: ReleaseChannel, device: DeviceProfile)`.
- `ApkCandidate`: provider ID, immutable release/asset identity, package/version name, optional Long version code, page/artifact locator, publication time, format/ABI/SDK hints, expected hash/certificates, and provenance.
- `AcquiredApk(file: File, candidate: ApkCandidate)`; `VerifiedApk` adds inspected package/version/build, signer evidence, content hash, and verified split/device configuration. Metadata hints alone never make an APK verified.
- `PatchGraphSnapshot`: qualified node IDs, exact selected keys, dependency closure, actual compatibility and option descriptors. `BundleSnapshot` pins source identity/UID, version, file and SHA-256.
- `AutomationPlan`: target candidate plus frozen graph and recipe; `PatchExecutionSnapshot` adds verified input, bundles/assets, output path, patcher version and expected signer.
- `PatchExecutionEvent`: `NodeSucceeded(nodeId: String)`, `NodeFailed(nodeId: String, message: String)`, `Stage(stage: RunStage)`, `Progress(message: String, completed: Int, total: Int)`, or `Restarted`. `PatchedArtifact`: output path/hash, inspected identity/signer, completed graph, `effectiveRecipe: AutomationRecipe`, and snapshot.
- `InstallReceipt(sessionId: Int?, packageName: String, versionName: String, versionCode: Long, signerSha256: String, artifactSha256: String)` holds verified installation evidence. `DownloadCheckpoint`: partial path, bytes, strong validator and resolved object identity.
- Place enums/policy/source configuration in `AutomationRecipe.kt`, stage/event/receipt in `AutomationRun.kt`, download/query types in `ApkCandidate.kt`, and graph/execution types in `PatchSnapshot.kt`. Declare `MirrorPage(candidates: List<ApkCandidate>, nextUrl: String?)` in Task 5's parser file.

Do not serialize callbacks, Android Contexts, KType objects, live Patch objects, or plaintext signing-key passwords into run payloads. Persist serializable option descriptors and values; reload and validate runtime objects against pinned bundles.

## Task 1: Establish an independently installable EverForge build

**Files:** Modify `app/build.gradle.kts`, `app/gradle.properties`, `settings.gradle.kts`, `app/src/main/AndroidManifest.xml`, `app/src/main/java/app/morphe/manager/util/Constants.kt`, `app/src/main/java/app/morphe/manager/network/api/MorpheAPI.kt`, `app/src/main/java/app/morphe/manager/ui/viewmodel/UpdateViewModel.kt`, `app/src/main/java/app/morphe/manager/ManagerApplication.kt`, `app/src/main/java/app/morphe/manager/ui/viewmodel/SettingsViewModel.kt`, `app/src/main/java/app/morphe/manager/domain/manager/PreferencesManager.kt`, `app/src/main/res/values/strings.xml`, launcher foreground vectors, `.github/workflows/build_pull_request.yml`, `.github/workflows/release.yml`, `.releaserc`, and release scripts.
Create `app/src/main/java/app/morphe/manager/domain/automation/ManagerArtifactValidator.kt`, `app/src/test/java/app/morphe/manager/domain/automation/ManagerArtifactValidatorTest.kt`, and `.github/scripts/verify-everforge-apk.py`.
Remove inherited `app-release.json`, `app/google-services.json`, FCM service/topic files, `.github/workflows/test_fcm.yml`, and `.github/scripts/send_fcm.py`.

**Interfaces:** Produce `ManagerArtifactValidator.validate(apk: File): Boolean` and pure `matchesManagerIdentity(expectedPackage: String, expectedSigner: String, actualPackage: String, verifiedSignerLineage: Set<String>): Boolean`. Existing update UI must validate before installation.

- [ ] Write negative identity tests with `assertFalse(matchesManagerIdentity("app.everforge.manager", signer, "app.morphe.manager", setOf(signer)))` and a same-package/different-signer assertion; valid EverForge identity must pass.
- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*ManagerArtifactValidatorTest'`; confirm the new tests fail for the missing validator.
- [ ] Set application ID to `app.everforge.manager`, initial version name `0.1.0`, min SDK 26, compile SDK 37 and explicit target SDK 36. Keep internal Kotlin namespace. Use a new monotonic version-code sequence without Morphe's legacy offset. Name artifacts `EverForge-<version>-debug.apk` or `EverForge-<version>-release.apk`.
- [ ] Rebrand visible manager text/vectors/actions and authorities; retain copyright/NOTICE and deliberate compatibility with the existing APK helper action. Point manager JSON and GitHub release fallback exclusively at `mekromn/everForge-manager`; use separate stable/prerelease JSON filenames on main. Do not offer inherited upstream manager metadata. Verify signing cryptographically with `ApkUtils.verifiedSigningCertificates`.
- [ ] Remove Firebase Messaging and Google Services plugin/configuration; remove GMS-gated update scheduling. Keep unrelated GmsCore detection and normal Android notifications. Default new installs to dark theme, AMOLED available, and custom color `#4285F4`; retain imported theme preferences.
- [ ] Adapt CI to main PRs and manual dispatch, JDK 21, tests/lint/build, and APK identity verification. Build debug when release secrets are absent; release publication is manual and requires the persistent EverForge keystore. Remove upstream website dispatch, FCM, and dev backmerge. Recreate release JSON only for an uploaded, verified EverForge artifact.
- [ ] Run the new unit tests plus `./gradlew :app:lint :app:assembleDebug`; verify APK identity, signer and label with the script. Commit `build: establish independent EverForge identity and CI`.

## Task 2: Persist automation records and exclusive execution ownership

**Files:** Create the five domain-model files and six persistence/files files in the structure table; create `app/src/test/java/app/morphe/manager/domain/automation/AutomationFixtures.kt`, `AutomationRepositoryTest.kt`, `ExecutionLeaseTest.kt`, and `app/src/androidTest/java/app/morphe/manager/domain/automation/AutomationMigrationTest.kt`.
Modify `app/src/main/java/app/morphe/manager/data/room/AppDatabase.kt`, `Migrations.kt`, `app/src/main/java/app/morphe/manager/di/DatabaseModule.kt`, `RepositoryModule.kt`, `app/build.gradle.kts`, `gradle/libs.versions.toml`; emit schema `app/schemas/app.morphe.manager.data.room.AppDatabase/18.json`.

**Interfaces:** `AutomationRepository` exposes `observeRecipes(): Flow<List<AutomationRecipe>>`, `getRecipe(id: String): AutomationRecipe?`, `putRecipe(recipe: AutomationRecipe)`, `getRun(id: String): AutomationRun?`, `enqueueIfAbsent(run: AutomationRun): AutomationRun`, `transition(id: String, expected: RunStage, next: AutomationRun): Boolean`, `appendEvent(event: AutomationEvent)`, and `claimExecution(workId: String): Boolean` / `releaseExecution(workId: String)`; mutations/reads are suspend except observation.

- [ ] Write tests asserting duplicate fingerprints return one run, two output-package clones remain separate, obsolete stage transitions fail, only one lease owner succeeds, and clock jumps cannot transfer a live lease. Fixtures define builders for the types above and a fake repository sharing the same contract. Example snippets use per-test setup variables named for their condition; build those in each named test class, not in production code.
```kotlin
@Test fun identicalFingerprintsShareOneRun() = runBlocking {
    val first = repo.enqueueIfAbsent(run)
    assertEquals(first.id, repo.enqueueIfAbsent(run.copy(id = "other")).id)
}
```

- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*AutomationRepositoryTest' --tests '*ExecutionLeaseTest'`; confirm failure for missing contracts.
- [ ] Implement Room recipe/run/event, active execution lease, and last-successful executed-recipe entities. Use transactional compare-and-update, unique fingerprint indexing, and explicit IDs. New schema version is 18 with additive `MIGRATION_17_18`; preserve all existing tables.
- [ ] Put durable files under private `filesDir/automation`, separated into assets, originals, bundles, runs and artifacts. Atomic staging renames commit complete files. Introduce `AutomationFiles.runDir(runId: String): File`.
- [ ] Add instrumentation dependencies: Room/work testing from their existing version aliases, AndroidX test core/runner 1.7.0 and ext.junit 1.3.0. Configure `androidx.test.runner.AndroidJUnitRunner` and schema assets.
- [ ] Seed schema 17 with selections/options/installed clones, migrate to 18, and assert unchanged records plus usable automation tables. Run the fingerprint/lease contract assertions against the real Room adapter, including concurrent claim attempts; fake-repository tests alone do not prove transactional ownership. Run `./gradlew :app:connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=app.morphe.manager.domain.automation.AutomationMigrationTest` and the new JVM tests. Commit `feat: persist automation recipes runs and execution leases`.

## Task 3: Enroll exact recipes and plan against all selected patches

**Files:** Create `AutomationEnrollment.kt`, `RecipeAssets.kt`, `PatchSnapshotBuilder.kt`, `StrictOptionCodec.kt`, `AutomationPlanner.kt` in the subsystem; tests `app/src/test/java/app/morphe/manager/domain/automation/StrictRecipeTest.kt` and `AutomationPlannerTest.kt`.
Modify `app/src/main/java/app/morphe/manager/patcher/patch/PatchBundle.kt` and `app/src/main/java/app/morphe/manager/domain/repository/PatchOptionsRepository.kt` only for strict reads/exact snapshot access.

**Interfaces:** `AutomationEnrollment.enroll(outputPackage: String, source: SourceConfiguration, policy: AutomationPolicy): AutomationResult<AutomationRecipe>`; `PatchSnapshotBuilder.snapshot(recipe: AutomationRecipe): AutomationResult<PatchGraphSnapshot>`; `AutomationPlanner.plan(recipe: AutomationRecipe, candidates: List<ApkCandidate>, graph: PatchGraphSnapshot, device: DeviceProfile): AutomationResult<AutomationPlan>`.

- [ ] Write assertions for missing selected patch → `MISSING_PATCH`, option type/nullable/default changes → `INVALID_OPTIONS`, missing managed asset → `MISSING_ASSET`, and incompatible selected source/dependency → `UNSUPPORTED_VERSION`. A universal patch with valid constraints must accept a new build.
- [ ] Include `longVersionCodeDoesNotWrap`: codes `3_000_000_000L` and `3_000_000_001L` remain distinct despite equal version names; compare allowed Int declarations after widening to Long.

```kotlin
@Test fun anIncompatibleSelectedSourceBlocksTheCandidate() {
    val result = planner.plan(recipe, candidates, incompatibleGraph, device)
    assertEquals(BlockerCode.UNSUPPORTED_VERSION,
        (result as AutomationResult.Blocked).blocker.code)
}
```
- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*StrictRecipeTest' --tests '*AutomationPlannerTest'`; confirm failures.
- [ ] Load real `Patch.dependencies` from pinned bundle files, preserve qualified source/selection keys and collision identity, and intersect every selected/dependency constraint. Detect cycles and unresolved nodes; do not use catalog unions as proof.
- [ ] Strictly decode using `Option.SerializedValue`, recorded type descriptors, validators and null semantics; freeze effective defaults. Copy and hash file/folder/list assets without image resizing or omission. Unreadable/ambiguous values block. Establish an original signer pin from verified enrollment evidence or bundle declarations; unavailable original/active-signer evidence blocks enrollment.
- [ ] Capture enrollment only from complete successful executed-recipe evidence. Upstream `snapshotSelection` stores `options = emptyMap()`; existing incomplete records must require a successful manual rebuild rather than reconstructing settings from global preferences. Increment revision on approved edits; persist source endpoint/local-bundle identity.
- [ ] Pin bundle files/hashes and managed assets in the plan. Preserve selected patches exactly, including explicit deselections; newly default patches and changed options need review. Return blocker details naming the relevant source/patch/option. Run both tests and commit `feat: enforce exact automation recipes and compatibility planning`.

## Task 4: Acquire and verify originals through native providers

**Files:** Create `sources/ApkSourceProvider.kt`, `SourceProviderRegistry.kt`, `AutomationDownloader.kt`, `ArtifactVerifier.kt`, `SavedOriginalSourceProvider.kt`, `GitHubSourceProvider.kt`, `ManifestSourceProvider.kt`, `FixedUrlSourceProvider.kt`; tests `app/src/test/java/app/morphe/manager/domain/automation/SourceProviderTest.kt`, `DownloadResumeTest.kt`, `ArtifactVerifierTest.kt`, and signed/split test fixtures under `app/src/test/resources/automation/`.
Modify existing split inspector/preparer only where the rejection tests identify a missing check.

**Interfaces:** `ApkSourceProvider.discover(query: CandidateQuery): AutomationResult<List<ApkCandidate>>`; `acquire(candidate: ApkCandidate, target: File, onProgress: (DownloadProgress) -> Unit): AutomationResult<AcquiredApk>`. `ArtifactVerifier.verifyOriginal(acquired: AcquiredApk, recipe: AutomationRecipe, device: DeviceProfile): AutomationResult<VerifiedApk>`; `verifyPatched(file: File, snapshot: PatchExecutionSnapshot, completed: Set<String>): AutomationResult<PatchedArtifact>`.

- [ ] Write tests for GitHub stable/experimental filtering and configured exact asset matching, manifest Long build codes, exact saved-original reuse, and no source promotion of an older saved APK.
- [ ] Add range tests: valid same-object resume; ignored range response restarts; changed ETag restarts. Assert an HTML `200 OK` body produces `INVALID_ARTIFACT`, not a saved original.
- [ ] Add verifier tests for wrong package/build/hash/signer, truncated APK, required split omission, duplicate split and escaping ZIP paths. Use cryptographically signed fixture APKs, not certificate text alone.
```kotlin
@Test fun wrongSignerIsNeverAnOriginal() = runBlocking {
    val result = verifier.verifyOriginal(wrongSignerApk, recipe, device)
    assertEquals(BlockerCode.SIGNER_MISMATCH,
        (result as AutomationResult.Blocked).blocker.code)
}
```

- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*SourceProviderTest' --tests '*DownloadResumeTest' --tests '*ArtifactVerifierTest'`; confirm failures.
- [ ] Implement streaming downloads and atomic finalization, bounded release discovery, provenance, hash comparison and actual APK inspection. Manifest schema is `{schemaVersion:1,releases:[{packageName,versionName,versionCode,url,sha256,format,abis,minSdk,signerSha256,prerelease,publishedAt}]}`; reject malformed/ambiguous release entries.
- [ ] Verify with `ApkUtils.verifiedSigningCertificates` on every supported SDK. Accept a pinned original certificate only through valid signing-lineage evidence. Use the active alias certificate for patched output. Reuse existing split inspection and all required device splits; verify archive provenance before reusing a merge.
- [ ] Add Ktor MockEngine as a test dependency matching 3.6.0. Run provider/verifier tests, with instrumentation for APK/SDK-dependent reads. Commit `feat: download and validate original APK updates`.

## Task 5: Add APKMirror and Uptodown acquisition

**Files:** Create `sources/ApkMirrorSourceProvider.kt`, `UptodownSourceProvider.kt`, `MirrorPageParser.kt`; tests `app/src/test/java/app/morphe/manager/domain/automation/MirrorSourceProviderTest.kt`; captured pages under `app/src/test/resources/automation/mirrors/`.
Modify `SourceProviderRegistry.kt`, `app/build.gradle.kts` and `gradle/libs.versions.toml`.

**Interfaces:** Implement Task 4's provider interface. `MirrorPageParser.parse(providerId: String, pageUrl: String, html: String, query: CandidateQuery): AutomationResult<MirrorPage>`; `MirrorPage` carries candidate releases or the next confirmed download step, never a fabricated locator.

- [ ] Capture representative original release/build/download pages for both sites, plus challenge, retired-build, redirected-latest and altered-markup fixtures. Tests assert the requested package/version/ABI, include the newest observed upstream release, and return `SOURCE_ATTENTION` for ambiguous or blocked pages.
```kotlin
@Test fun aChallengePageNeedsAttention() {
    val result = parser.parse("apkmirror", pageUrl, challengeHtml, query)
    assertEquals(BlockerCode.SOURCE_ATTENTION,
        (result as AutomationResult.Blocked).blocker.code)
}
```

- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*MirrorSourceProviderTest'`; confirm failure.
- [ ] Parse HTML with jsoup 1.23.2; enable NIO core desugaring with `com.android.tools:desugar_jdk_libs_nio:2.1.5` for Android compatibility. Use configured source pages and `DownloadUrlResolver` for requested versions; a Google-search fallback is attention, not a download.
- [ ] Bound page/redirect traversal to eight verified steps per acquisition. Reject login/CAPTCHA and expired-token cycles. Select required ABI/build variants, and send every download through Task 4's verifier. A fixed APK URL is labeled a single build, not a discovery feed.
- [ ] Verify one real supported download from each site without guessing links; if live access is challenged, record that limitation and retain a working manual attachment path. Run fixture tests plus an API-26 parser smoke check. Commit `feat: acquire originals from supported mirror pages`.

## Task 6: Reuse the patch engine with frozen inputs and complete evidence

**Files:** Create `AutomationPatchExecutor.kt`; modify `app/src/main/java/app/morphe/manager/patcher/runtime/Runtime.kt`, `CoroutineRuntime.kt`, `ProcessRuntime.kt`, `runtime/process/Parameters.kt`, `PatcherProcess.kt`, `app/src/main/java/app/morphe/manager/patcher/Session.kt`, `patcher/worker/PatcherWorker.kt`, `domain/worker/WorkerRepository.kt`, `domain/batch/BatchPatchCoordinator.kt`, and `ui/viewmodel/PatcherViewModel.kt`.
Modify `app/src/main/aidl/app/morphe/manager/patcher/runtime/process/IPatcherEvents.aidl`; tests `app/src/test/java/app/morphe/manager/domain/automation/PatchExecutionContractTest.kt`.

**Interfaces:** `AutomationPatchExecutor.execute(snapshot: PatchExecutionSnapshot, emit: suspend (PatchExecutionEvent) -> Unit): AutomationResult<PatchedArtifact>`. Both runtime constructors accept `bundleOverride: Map<Int, PatchBundle>? = null` and `workspaceOverride: File? = null`; existing callers default to current behavior. Add `void patchNodeSucceeded(String nodeId)` and `void patchNodeFailed(String nodeId, String message)` to AIDL while retaining existing display/progress callbacks.

- [ ] Write tests asserting bundle refresh cannot change an executing snapshot; same-named patches in different sources remain distinct; a dependency failure prevents a verified artifact; and restarted attempts clear previous completion evidence.
```kotlin
@Test fun dependencyFailureCannotProduceAnArtifact() = runBlocking {
    val result = executor.execute(snapshotWithFailingDependency) { events += it }
    assertTrue(result is AutomationResult.Blocked)
    assertTrue(events.any { it is PatchExecutionEvent.NodeFailed })
}
```

- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*PatchExecutionContractTest'`; confirm failure.
- [ ] Adapt the existing merge/patch/runtime/fallback/sign pipeline behind the executor. Pass pinned bundles through coroutine and remote-process parameters; do not re-read mutable repository bundles during automatic execution. Keep existing memory/OOM behavior.
- [ ] Observe selected patches and all dependencies, including unnamed dependency nodes. `Session` must detect exceptions before filtering out unselected nodes. Verify full completion before signing/output promotion; strict option application must throw on any omitted/invalid enrolled value.
- [ ] Acquire/release the global lease inside every execution worker; manual/batch launch checks must not replace or overlap an active automatic run. Release only after child-process stop is acknowledged. Keep per-run output/workspace cleanup isolated.
- [ ] Persist successful effective selections, every option/default, bundle identities and signer evidence for manual, batch and automatic builds. Enroll only these complete snapshots; bind tracking updates to frozen bundle versions rather than later refreshed versions.
- [ ] Run contract tests, existing patcher/option/split tests and a tiny real patch fixture in both runtimes. Commit `feat: execute frozen recipes through the existing patch engine`.

## Task 7: Run and recover durable automation jobs

**Files:** Create `AutomationRunner.kt`, `AutomationRecovery.kt`, `AutomationRetryPolicy.kt`, and `app/src/main/java/app/morphe/manager/worker/AutomationRunWorker.kt`; modify `app/src/main/java/app/morphe/manager/di/WorkerModule.kt` and manager/repository DI registration.
Tests: `app/src/test/java/app/morphe/manager/domain/automation/AutomationRecoveryTest.kt` and `app/src/androidTest/java/app/morphe/manager/domain/automation/AutomationWorkerTest.kt`.

**Interfaces:** suspend `AutomationRunner.advance(runId: String, workId: String): AutomationResult<AutomationRun>`, `retry(runId: String): AutomationResult<AutomationRun>`, and `AutomationRecovery.reconcile(runId: String): AutomationResult<AutomationRun>`; pure `AutomationRetryPolicy.nextDelay(failedAttempts: Int, serverRetryAfterMillis: Long?): Long?`. Worker input is only `runId`.

- [ ] Write tests for reload with an empty process-local input map, restart from verified original after interrupted patching, ignoring partial output, and reconciling committed/completed installer sessions without a second install.
- [ ] Assert first and second download failures schedule 30s and 60s minimum backoff, respectively; the third failure returns no automatic retry. Respect a longer server Retry-After. Platform quota/constraint stops do not count as download failures. Deterministic patch failures require changed inputs or explicit retry.

```kotlin
@Test fun threeFailuresExhaustAutomaticBudget() {
    assertEquals(30_000L, policy.nextDelay(1, null))
    assertEquals(60_000L, policy.nextDelay(2, null))
    assertNull(policy.nextDelay(3, null))
}
```
- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*AutomationRecoveryTest'`; confirm failures.
- [ ] Implement persisted transitions through the spec's stages and typed waiting reasons. Resolve unknown discovery build codes only after acquisition and final strict validation. Explicit retry revalidates the current recipe and resets the failed run's attempt budget transactionally; it does not create duplicate fingerprint rows. Cancellation propagates; opt-out stops current and queued work before further stage advancement.
- [ ] Record work IDs and stop reasons; reconcile WorkManager ownership before recovering leases. A clock shift cannot steal ownership. Restart patching from immutable inputs rather than resuming mutated resource/DEX output.
- [ ] Use foreground worker notifications for long work. On denied promotion or scheduling limits, persist a waiting/retry result; provide a resumable action through Task 10. Run worker instrumentation with constraint/cancel/process-restart simulations. Commit `feat: recover automatic update jobs across interruptions`.

## Task 8: Install validated artifacts without data-erasing fallback

**Files:** Create `AutomationInstaller.kt`, `AutomationInstallReceiver.kt`, `RecipeInstallGate.kt`; modify `app/src/main/java/app/morphe/manager/domain/installer/SessionInstaller.kt`, `ShizukuInstaller.kt`, `InstallerManager.kt`, `domain/repository/InstalledAppRepository.kt`, and `app/src/main/AndroidManifest.xml`.
Tests: `app/src/test/java/app/morphe/manager/domain/automation/AutomationInstallerTest.kt`, existing `app/src/test/java/app/morphe/manager/domain/installer/SilentUpdateEligibilityTest.kt`, and `app/src/androidTest/java/app/morphe/manager/domain/automation/InstallRecoveryTest.kt`.

**Interfaces:** `AutomationInstaller.install(runId: String): AutomationResult<InstallReceipt>`; `reconcile(runId: String): AutomationResult<InstallReceipt>`. `RecipeInstallGate.withRecipeLock(recipeId: String, block: suspend () -> T): T` serializes recipe edits with the final install commit check. Persist session ID before commit; durable receiver updates the run independently of activity lifetime.

- [ ] Test active-alias mismatch in a multi-alias keystore → `SIGNER_MISMATCH`; same signer and validated output permits update. Assert signature conflict, downgrade, unexpected package, disabled recipe or changed revision causes zero uninstall/commit calls.
- [ ] Test no Shizuku/lost permission → waiting; system pending-user-action survives recreation; session completion is reconciled once; completed evidence must match actual installed package/build/signer/artifact.
```kotlin
@Test fun changedRecipeCannotCommit() = runBlocking {
    repo.putRecipe(recipe.copy(revision = recipe.revision + 1))
    val result = installer.install(run.id)
    assertEquals(BlockerCode.RECIPE_CHANGED,
        (result as AutomationResult.Blocked).blocker.code)
    assertEquals(0, platform.commitCalls)
    assertEquals(0, platform.uninstallCalls)
}
```

- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*AutomationInstallerTest' --tests '*SilentUpdateEligibilityTest'`; confirm new cases fail.
- [ ] Correct ordinary silent eligibility: minimum target SDK 29 on API 31–32; 30 on 33; 31 on 34; 33 on 35; 34 on 36; 35 on 37. Require permission plus installer/update-owner/self-update rules; unknown later SDKs use confirmation.
- [ ] Implement prepare/notify and opt-in auto-install. Bypass auto-uninstall-on-conflict entirely in this path. Use non-exported durable install-result handling and notification-driven confirmation; do not store a callback solely in `onReturnToForeground`.
- [ ] Re-check revision/enabled state under the shared recipe gate immediately before commit; recipe edits cancel any uncommitted/pending session and reconcile committed ones. Mark installed only after validation; preserve the old installed evidence until success.
- [ ] Run JVM and installation-recovery instrumentation tests, then Shizuku tests with dedicated fixture packages. Commit `feat: install automation updates with durable confirmation and signer checks`.

## Task 9: Schedule checks independently of push and notification visibility

**Files:** Create `AutomationScheduler.kt`, `AutomationChecks.kt`, `app/src/main/java/app/morphe/manager/worker/AutomationCheckWorker.kt`; modify `app/src/main/java/app/morphe/manager/ManagerApplication.kt`, `di/WorkerModule.kt`, and `domain/manager/PreferencesManager.kt`.
Tests: `app/src/test/java/app/morphe/manager/domain/automation/AutomationScheduleTest.kt`, `app/src/androidTest/java/app/morphe/manager/domain/automation/AutomationConstraintsTest.kt`.

**Interfaces:** `AutomationScheduler.sync(recipes: List<AutomationRecipe>)`, `checkNow(recipeId: String)`, `cancel(recipeId: String)`; `AutomationChecks.check(recipeId: String): AutomationResult<AutomationRun?>`.

- [ ] Test six-hour default, hourly/daily/manual choices; multiple check events produce one fingerprinted run; silenced notifications and absent GMS do not disable checks. Opt-out cancels the named periodic/check/run work.
```kotlin
@Test fun theDefaultScheduleIsSixHours() {
    assertEquals(CheckInterval.SIX_HOURLY, AutomationPolicy().interval)
    assertEquals(360L, CheckInterval.SIX_HOURLY.minutes)
}
```

- [ ] Run `./gradlew :app:testDebugUnitTest --tests '*AutomationScheduleTest'`; confirm failures.
- [ ] Schedule unique `everforge_check_<recipeId>` periodic work with UPDATE policy, or no periodic work for manual. Enrollment and **Check now** enqueue immediate connected checks. Download/patch work uses configured unmetered, battery-not-low, storage-not-low and optional charging constraints.
- [ ] Refresh only enrolled, enabled, unblocked source identities through the existing source infrastructure; respect source auto-update configuration. Stable recipes must obtain stable patch snapshots even if a shared source currently has a prerelease loaded; require experimental opt-in on both recipe and source for experimental snapshots. Do not mutate another recipe's shared source settings. Pin bundles and compare app/patch fingerprints. Show newer observed unsupported releases even if the current app is otherwise fully patched.
- [ ] Register startup reconciliation and recipe-policy observation. Notification preferences affect presentation only. Run constrained-worker tests for metered network, low battery/storage, charging changes and fresh app startup. Commit `feat: schedule managed app checks and constrained rebuilds`.

## Task 10: Expose enrollment, progress and actionable waiting states

**Files:** Create `app/src/main/java/app/morphe/manager/ui/viewmodel/AutomationViewModel.kt`, `ui/screen/automation/AutomationScreen.kt`, `AutomationEnrollmentDialog.kt`, `AutomationRecipeDialog.kt`, `AutomationStatus.kt`, and `app/src/main/java/app/morphe/manager/domain/automation/AutomationNotifications.kt`.
Modify `ui/model/navigation/Nav.kt`, `MainActivity.kt`, `ui/screen/home/InstalledAppInfoDialog.kt`, `HomeAppCards.kt`, `ui/screen/SettingsScreen.kt`, DI viewmodel registration and strings.
Tests: `app/src/test/java/app/morphe/manager/domain/automation/AutomationPresentationTest.kt`.

**Interfaces:** Pure `presentAutomation(run: AutomationRun, latestUnsupportedVersion: String?): AutomationStatus` lives in `AutomationStatus.kt`; its result has `stage: RunStage`, `hasVerifiedInstall: Boolean` and `newerUnsupportedVersion: String?`. Viewmodel operations: `AutomationViewModel.enroll(outputPackage: String, source: SourceConfiguration, policy: AutomationPolicy)`, `checkNow(recipeId: String)`, `pause(recipeId: String)`, `retry(runId: String)`, `install(runId: String)`, `saveRecipe(recipe: AutomationRecipe)`; all UI state comes from repository observation.

- [ ] Write presentation tests: prepared APK is not labeled installed; newer unsupported release stays visible; missing asset identifies the option; denied notification visibility remains visible in settings. Run `./gradlew :app:testDebugUnitTest --tests '*AutomationPresentationTest'`; confirm failure.
```kotlin
@Test fun preparedOutputIsNotAnInstalledUpdate() {
    val status = presentAutomation(readyRun, "3.0")
    assertFalse(status.hasVerifiedInstall)
    assertEquals("3.0", status.newerUnsupportedVersion)
}
```

- [ ] Add **Keep updated** enrollment with source, recipe summary, channel and installation policy. Missing complete recipe routes to a manual rebuild; missing signer import routes to existing keystore import.
- [ ] Add queue/history, current stage, previous/new versions, last/next checks, patch-source versions and specific blocker actions: edit recipe, attach original, grant installer permission, retry, resume, or install. Preserve clone identity in actions.
- [ ] Add frequency/network/charging settings, prepare/auto-install selection, and explicit experimental opt-in. Notifications deep-link to durable run IDs; silence does not pause checks.
- [ ] Render UI on phone and split screen in dark/AMOLED modes, inspect contrast/insets/scrolling, and exercise one action for each waiting state. Run presentation tests and lint; commit `feat: manage EverForge automation from app cards and queue`.

## Task 11: Preserve automation assets through storage cleanup and imports

**Files:** Create `AutomationRetention.kt`; modify `app/src/main/java/app/morphe/manager/domain/repository/StorageStatsRepository.kt`, `data/platform/Filesystem.kt`, `ui/viewmodel/StorageManagementViewModel.kt`, and `ui/viewmodel/ImportExportViewModel.kt`.
Tests: `app/src/test/java/app/morphe/manager/domain/automation/AutomationRetentionTest.kt` and `AutomationImportTest.kt`.

**Interfaces:** `AutomationRetention.cleanup(): Long`; `exportRecipes(): String` and `importRecipes(json: String): AutomationResult<List<AutomationRecipe>>` on the enrollment/import adapter. Import initially disables execution until source/asset/active-signer verification and user confirmation.

- [ ] Test cleanup preserves active/pending outputs, latest successful artifact, enrolled assets/bundles and all signing keys. Imported source UID remapping preserves canonical identity; absent assets/keystore block reenrollment. Run the two JVM test classes and confirm failure.
```kotlin
@Test fun cleanupPreservesPendingOutputAndKeys() = runBlocking {
    retention.cleanup()
    assertTrue(pendingArtifact.exists())
    assertTrue(signingKey.exists())
}
```

- [ ] Integrate automation storage accounting. Keep newest successful output plus pending install artifacts; cap completed history at 50 runs per recipe and log files at 10 MiB each. Prune only unreferenced originals/bundles/assets. Cleanup uses the execution lease.
- [ ] Include recipe revisions, source configuration and asset hashes in versioned backup metadata. Continue using the existing separate keystore backup/import; never export private keys in recipe JSON. Never auto-enable imported recipes based on source UID or package-name coincidence.
- [ ] Run import/retention tests and existing import/export tests; commit `feat: retain and restore automation assets without losing signing continuity`.

## Task 12: Verify a complete update cycle and deliver a reviewable APK

**Files:** Create `app/src/androidTest/java/app/morphe/manager/domain/automation/AutomationEndToEndTest.kt`, reproducible fixture source/build instructions under `test-fixtures/automation/`, and `docs/everforge-automation.md`. Update CI for instrumented fixture coverage and save observed verification results under `docs/verification/`.

**Interfaces:** Consume all earlier contracts; no additional product interfaces.

- [ ] Write an end-to-end test using two original versions and two patch revisions. Assert identical enrolled options/selection survive, output/signing identity is stable, and installed evidence advances only after successful verification.
- [ ] Add an unsupported third original, a dependency failure, duplicate triggers, two clones, cancellation, recipe edit before commit and restart during download/patch/install. Assert correct waiting states and zero partial/stale installs.

```kotlin
assertEquals(recipe.outputPackageName, secondReceipt.packageName)
assertEquals(firstReceipt.signerSha256, secondReceipt.signerSha256)
assertEquals(enrolledOptions, secondArtifact.effectiveRecipe.options)
assertEquals(RunStage.INSTALLED, repository.getRun(secondRun.id)!!.stage)
```

These receipt/artifact field names are part of Task 2's model contract.
- [ ] Run `./gradlew :app:connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=app.morphe.manager.domain.automation.AutomationEndToEndTest`; confirm failing behavior before integration fixes. Fixtures must execute the real patch engine, not just a fake executor.
- [ ] Complete integration fixes within their owning tasks; run `./gradlew :app:testDebugUnitTest :app:lint :app:assembleDebug :app:assembleDebugAndroidTest` and instrumented migration/worker/installer/end-to-end tests. Verify both patch runtimes and API-26 compatibility.
- [ ] Run a supported real APK update cycle on the Pixel 9 Pro XL/Android 16, including app backgrounding and Shizuku loss/recovery. Use live authorized device capabilities at execution time. Keep dedicated QA packages separate from existing user apps; a missing device test remains an explicit unverified result.
- [ ] Verify manager install/update continuity under a persistent signing key using two successive artifacts, plus runtime patched-app keystore export/import continuity. Validate package, version, signer and APK SHA-256.
- [ ] Perform one independent whole-branch review, fix material findings, re-run affected checks, push `feat/everforge-automation`, and open a draft PR against main with the actual validation evidence. Do not merge or publish a release as part of this handoff.
- [ ] Deliver the verified APK link and concise test/remaining-blocker report. If release signing is unavailable, label the output a debug build; do not describe unrun device/mirror checks as passed. Commit `test: verify the complete EverForge update lifecycle`.

## Execution and Coverage Check

Execute tasks sequentially unless the user chooses the subagent workflow. At execution time, create an isolated feature worktree using the relevant skill; retain this spec and plan on the feature branch. Use a failing test before product implementation where a task lists a test cycle. Documentation/rebranding changes need the stated artifact checks, not additional tests that mirror labels.

| Approved requirement | Owning tasks |
| --- | --- |
| Independent identity, dark defaults, manager updater, signing/release isolation, Firebase removal | 1, 8, 12 |
| Durable records, exact enrollment/options/assets, clones, full compatibility/dependencies | 2, 3, 6 |
| Saved originals, GitHub/manifest/mirror discovery, validated downloads and split support | 4, 5 |
| Immutable source snapshots, full patch completion, single shared runtime ownership | 2, 6 |
| Restart/cancel/quota handling and bounded retry | 7, 9 |
| Installation/confirmation/revision checks, no data-erasing fallback, verified status | 8, 10 |
| Scheduled checking and separate notification visibility | 9, 10 |
| History, retention, backup/import and signing continuity | 11, 12 |
| JVM/lint/build plus actual device/mirror/engine verification | 4–12 |

The local planning environment has JDK 17 and no discovered Android SDK. At execution, use a suitable JDK 21/SDK 37 environment or the adapted GitHub Actions build. A dependency/download/toolchain failure is not a red test proving behavior; resolve that prerequisite or report the precise blocker before claiming validation.

Primary sources used for pinned additions: [AndroidX Test](https://developer.android.com/jetpack/androidx/releases/test), [jsoup downloads and Android requirements](https://jsoup.org/download), [desugar library changelog](https://github.com/google/desugar_jdk_libs/blob/master/CHANGELOG.md). Patcher interfaces were read from the public [Morphe patcher v1.15.1 source](https://github.com/MorpheApp/morphe-patcher/tree/v1.15.1).

