# EverForge automatic update and repatching design

Date: 2026-10-11 (America/Los_Angeles)
Status: Approved by the user on 2026-10-11; implementation awaits plan review.
Repository: https://github.com/mekromn/everForge-manager
Reviewed base: `da8545c92c94b83e23adf7b573b46b282fa2b062` (Morphe Manager 1.35.0)

## Purpose and success

The user wants EverForge, a Morphe fork that automatically updates and patches APKs so their modified apps stay up to date.

A managed app has a saved patch recipe. EverForge watches both the app's original APK releases and its patch sources, obtains an appropriate original, reapplies the recipe, signs the result with the same key, and updates the installed app when its configured installer permits it.

The promise is continuous maintenance of the newest version that can satisfy the complete recipe. An upstream release can precede patch support. EverForge must show that gap and keep watching; it must not call an older installed build current simply because no compatible update exists.

Initial assumptions, open to correction during this review:

- All download and patch work runs on the Android device. There is no mandatory account or hosted build service.
- Android 8 and newer remain supported; the main acceptance device is an unrooted Pixel 9 Pro XL on Android 16.
- Automatic checking and rebuilding become active after a user enrolls an app and confirms its recipe and APK source. Installation is configurable per app.
- Silent installation uses the existing Shizuku path when enabled and available. Android's session installer is also supported where the platform allows it; otherwise the result waits for an install tap.
- Stable app and patch releases are the default. Experimental releases require a separate per-app choice.
- Existing Morphe features remain available. Root mount continues as a manual flow; unattended root mount is outside the first release.
- The working interface keeps the existing theme system; EverForge's default is dark with an AMOLED option and Pixel blue accent.

## Repository findings

| Existing foundation | What EverForge needs |
| --- | --- |
| `UpdateCheckWorker.kt` checks manager and patch releases and posts notifications. | A separate managed-app scheduler that works regardless of push messaging or notification preferences. |
| `BatchPatchCoordinator.kt` runs apps sequentially and can install through Shizuku. Its state is in memory. | Durable run state and a shared execution lease covering automatic, manual, and batch patching. |
| `WorkerRepository.kt` passes arguments through a process-local map; `PatcherWorker.kt` rejects retries. | Automatic workers that load persisted job input by ID and recover after process death. |
| `BatchPlanResolver.kt` validates selections, adds new default patches, and removes unreadable option paths. | Strict unattended planning: preserve the enrolled recipe or block with a precise reason. |
| `AppVersionCatalog.kt` provides versions advertised across sources. | Compatibility evaluation against every selected patch and its dependency closure. A union of advertised versions is insufficient. |
| `DownloadUrlResolver.kt` finds a web page; helpers use an interactive activity contract. | Native source providers that fetch files without an activity, plus explicit manual fallback. |
| Installed-app records preserve original/output package identities, selections, and clone tracking. | Distinct automation recipes for distinct installs, with provenance, revision, and verification evidence. |
| Room database is version 17; signing, split preparation, and source repositories already exist. | Additive schema migrations and integration with those facilities. |
| Manager package, release feeds, branding, and Firebase configuration still identify Morphe. | Independent EverForge identity, release validation, and scheduling. |

Relevant source paths are under `app/src/main/java/app/morphe/manager/`. Internal Kotlin package names can remain during the first release to keep upstream integration manageable.

## Alternatives considered

| Approach | Benefit | Limitation |
| --- | --- | --- |
| **Device-native update loop — recommended** | Uses current patch engine, local signing keys, and Shizuku; users can configure independent APK sources. | Must handle device scheduling, storage, source challenges, and recovery. |
| Notification and batch shortcut only | Small change using existing flows. | Users still download APKs and start runs; does not deliver the requested automation. |
| Hosted builders and a signed APK feed | Moves expensive work off the phone. | Adds hosting, key custody, configuration synchronization, and a new service to maintain. |

Build the device-native loop as one end-to-end subsystem, including sources, strict planning, persistent execution, and installation. Adding a scheduler alone is not an EverForge first release.

## Product flow

1. Patch an app normally, or import existing settings and the original signing keystore.
2. Choose **Keep updated** on that tracked app.
3. Confirm the saved recipe, original APK source, release channel, and installation mode.
4. EverForge checks immediately and on the chosen schedule.
5. A source or app release changes; the planner selects a compatible original and creates one durable run.
6. The run downloads, validates, patches, signs, validates its output, and installs or waits for an install tap.
7. The app card reports the verified result, the next check, or the exact blocker.

Enrolling from an existing build captures its last successful effective selection and options, not unrelated current global preferences. If a complete recipe cannot be recovered, the app remains unenrolled until one successful configured build supplies it.

Recipe edits create a new revision. An older run cannot install after that revision has changed.

## Architecture and ownership

| Component | Responsibility |
| --- | --- |
| `AutomationRepository` | Room-backed recipes, run snapshots, attempts, events, artifacts, and installation evidence. |
| `ApkSourceProvider` | Discover release candidates and download the requested original; return typed failures and provenance. |
| `AutomationPlanner` | Compare installed evidence, releases, selected patches, dependency requirements, options, and device compatibility. |
| `AutomationScheduler` | Unique periodic checks and explicit **Check now** requests; deduplicate targets and honor policy. |
| `AutomationRunWorker` | Load a run by ID and advance persisted stages, including recovery and cancellation. |
| Shared patch execution adapter | Reuse Morphe unpack/merge/patch/sign behavior with immutable job inputs and observable progress. |
| `AutomationInstaller` | Install only a validated artifact under the recipe's current installation policy; reconcile session results. |
| Automation presentation | Home status, queue/history, settings, actions, and notifications derived from persisted state. |

Use existing Koin, Room, WorkManager, Ktor, split tools, signing, and installers. Do not create a second patch engine or make a ViewModel own the automatic queue.

A global execution lease serializes the patch runtime and its shared workspaces across all entry points. Acquiring it is atomic. Worker identity and WorkManager state govern recovery; a wall-clock timeout alone must not steal a live worker's lease.

Each run snapshots the recipe, input identity, patch bundle files and hashes, option assets, patcher version, and signing certificate. Source refresh cannot swap bundles underneath an executing run. An explicit pause/cancel stops active work as well as new dispatch; staging waits for cancellation acknowledgement before another run acquires the lease.

## Durable records

Add Room entities with explicit migration from version 17:

- **AutomationRecipe:** ID, revision, original package, expected output package, clone identity, selected source/patch names, encoded effective options, managed file assets, original certificate allowlist, signing certificate, source configuration, channel, enabled state, installation mode, and scheduling constraints.
- **AutomationRun:** recipe revision, target original version/build, immutable source/bundle/asset hashes, deduplication fingerprint, stage, work ID, attempt count, next retry, output path/hash, installation session ID, error category, and timestamps.
- **AutomationEvent:** stage/progress changes and actionable error details. Keep logs per run and cap retention.

Store large payloads and APKs in private files; workers receive only durable IDs through WorkManager Data. Options referencing images or other files are copied into managed assets at enrollment. If copying or decoding fails, enrollment or the run blocks; an option is never silently omitted.

Distinct clone/output installs have distinct recipes even when they share an original package. Shared originals may be reused by verified content hash.

## APK source acquisition

First-release providers:

| Provider | Selection and behavior |
| --- | --- |
| Verified saved original | Reuse the exact target build already on disk, particularly for patch-only updates. Never substitute an old original for a newer target. |
| GitHub releases | User-configured owner/repository and asset matcher, stable/pre-release filtering, bounded candidate discovery, then APK identity inspection. |
| HTTPS JSON release manifest | User-configured feed of package, version name, long version code, artifact URL, SHA-256, format, ABI/minimum SDK, and certificate metadata. |
| APKMirror and Uptodown | Resolve the requested release from the existing download-page resolver, parse supported download pages, follow bounded download steps, then verify the actual artifact. |

Mirror adapters use captured page fixtures for selection and parsing tests. CAPTCHA, login, rate limits, expired links, or changed markup become **Source needs attention**, not guessed links or an endless retry loop. Existing helper/manual selection remains a recovery path. Its activity contract must not be treated as a background download service.

A fixed direct APK URL can acquire one build but is not a discovery feed. The interface makes that distinction explicit.

All providers return provenance and the exact inspected package/version/build/ABI. Downloads stream into a per-run partial file; commit the file only after complete validation. Resume only when the server's range response and validators prove the same object; otherwise restart that download.

Validate package identity, device support, archive format, declared hash when supplied, and original signing certificate. Establish the certificate allowlist from bundle declarations or a user-confirmed, verified enrollment original; do not silently trust a new signer. Reject HTML returned as an APK.

Split archives use Morphe's existing inspector and preparation path. Preserve required device splits and configuration instead of selecting only a base APK. A previously merged cache is reusable only when its build, device configuration, and original archive provenance still match.

## Strict recipe replay

For each candidate:

1. Resolve every enrolled source and selected patch by its exact source identity and patch name.
2. Resolve and validate dependency closure and all patch option definitions.
3. Intersect the selected patches' version, build, SDK, architecture, and experimental constraints.
4. Decode the saved option values using the existing typed option codec.
5. Verify all managed file assets and the enrolled output-package settings.
6. Record precisely what will execute.

A missing/renamed patch, removed source, invalid option type, changed output package, or missing asset blocks the automatic run. Offer a recipe-edit action showing the mismatch. Fuzzy name matching, automatically adding newly default-selected patches, dropping unreadable paths, and global compatibility-check bypasses are prohibited in unattended mode.

Universal patches can run on newer versions when their actual constraints permit it. Constrained patches must all accept the candidate. A runtime fingerprint failure still blocks the result even when metadata advertised compatibility.

A changed patch bundle can trigger rebuilding the current app version when no newer compatible original is available. Fingerprints distinguish such a patch update from an app-version update and prevent duplicate work.

Automatic replay is not automatic invention or repair of patch code after an incompatible upstream change. That future feature would need separate requirements and validation.

## Queue, scheduling, and recovery

The persisted flow is **Queued → Resolving → Downloading → Validating original → Patching → Signing → Validating output → Ready to install → Installing → Installed**.

Waiting states distinguish constraints, patch compatibility, source attention, installer permission, and user confirmation. Failed and canceled runs preserve their reason and history. The UI distinguishes **Patched APK ready**, **Installed**, and **Waiting for compatible patches**.

Defaults after enrollment:

- Check every six hours; offer hourly, six-hourly, daily, and manual schedules.
- Unmetered downloads by default; explicit mobile-data opt-in.
- Require battery-not-low and storage-not-low for automatic patch work; charging-only is configurable.
- Process one patch job at a time. **Check now** requests an immediate check subject to Android scheduling.
- Retry transient download/network failures with exponential backoff, capped at three automatic attempts for an unchanged fingerprint. Source/patch/recipe changes or an explicit retry permit another attempt.
- Do not repeat the same deterministic patch failure until relevant input changes or the user requests retry.
- Keep the latest successful artifact and any pending install. Older files follow storage retention; active files and signing keys are never removed by cache cleanup.

Periodic work is independent of whether notifications are silenced and independent of Google Play services. Stop automatic work on opt-out, and re-check enabled/revision state before installing.

On process death, inspect persisted stages and WorkManager/session state. Downloads resume or restart safely. An interrupted patch restarts from its verified original and immutable bundle snapshot; it cannot resume halfway through mutated DEX/resources. Ignore incomplete output files. Reconcile an existing install session before starting another; if installation already succeeded, verify and finish without installing twice.

Android 16 can limit long-running WorkManager execution even with a foreground service. Persist progress, record stop reasons, yield/retry under system policy, and offer **Resume now**. Do not promise exact execution times or a permanent wake lock.

## Output and installation contract

Before making an artifact installable:

- All selected patches and dependencies must complete successfully; maintain completion evidence.
- Reopen the resulting APK, verify expected output package/version/build, and confirm valid signing under the enrolled certificate.
- Verify artifact hash and recipe revision, and commit the artifact record atomically.
- Keep the previous verified installed evidence until the new installation is confirmed.

Automatic modes are **Prepare and notify** and **Install automatically when available**. The second supports Shizuku and eligible Android session updates. First installs, unavailable permission, or a platform confirmation requirement wait visibly.

Re-check the installed signing certificate and version before committing an update. A signature conflict never invokes Morphe's existing auto-uninstall-on-conflict option in this unattended path. Data-erasing uninstall, downgrade, and replacement of an unexpected output package require the separate manual flow.

On Android 16, ordinary silent session updates also depend on target SDK, installer/update ownership, and permission; the current upstream eligibility helper must be corrected rather than assumed sufficient. Handle pending user action as a durable waiting state.

Mark **Installed/current** only after PackageManager and install-session evidence match the validated artifact and run. A newer known incompatible upstream version remains visible as **Waiting for patches**, even if the installed app is the newest supported build.

Retaining a previous APK helps recovery but does not guarantee data-compatible downgrade. Offer manual recovery and do not automate rollback.

## Fork identity and release behavior

- Display name: **EverForge**.
- Release application ID: `app.everforge.manager`; debug suffix remains separate. Morphe can coexist.
- Update provider authorities, exported EverForge action names, labels, notification text, and launcher branding. Keep legacy helper/deep-link compatibility deliberately where needed.
- Preserve GPL licensing, copyright notices, Morphe attribution, and the existing NOTICE. Identify the modified program clearly.
- EverForge manager updates point only to this repository. Replace inherited release metadata and validate downloaded manager package and signer so the updater cannot install a Morphe APK.
- Remove the inherited Firebase push setup and its Google Services build requirement; use the independent scheduler instead. Retain normal Android notifications. Notification permission affects visibility, not whether checking is enabled; record denied visibility in settings and continue only work the platform actually permits.
- Separate the manager APK's release-signing identity from its runtime keystore for patched apps. Preserve both across updates; do not generate a new manager signing key on every CI run.
- Importing Morphe's settings does not make its private keystore available automatically. Reuse requires the existing backup/import process; mismatches block installation.
- Keep Morphe patch feeds and library coordinates unless explicitly changed. Upstream patch authors remain responsible for their patch releases.

## Verification and first-release acceptance

Meaningful tests cover:

- Two original app releases and two patch bundle revisions: identify, download, replay identical selections/options, sign, and update the expected package.
- Unsupported latest release: keep the working app, show the newer version and compatibility gap, and retry when a compatible patch arrives.
- Removed patch, renamed patch, unreadable asset, changed option type, and an incompatible member of a multi-source recipe: block rather than reduce the recipe.
- Duplicate scheduler triggers, separate clones, recipe changes during a run, and simultaneous manual/batch work: no duplicate or stale installation.
- Process death during download, patch execution, and installation: restart safely, reconcile sessions, and avoid double installs.
- APKMirror/Uptodown fixtures, source challenges, hash/certificate mismatch, wrong package/build, split selection, truncated APKs, and HTML response bodies.
- Missing Shizuku, lost permission, system pending user action, rejected signatures, storage pressure, cancellation, and network loss.
- Room migration preserves selections, options, tracked apps, and signing references.
- Fork identity, independent manager-update feed, and sequential APK updates under a stable signing certificate.

Run existing JVM tests, added automation tests, lint, and APK assembly. Validate real on-device background and Shizuku behavior on Android 16 before describing the release as fully verified. Run at least one complete real update cycle for a supported app; fixture tests alone do not prove mirror availability or background installation.

Build evidence at design time: the local workspace has JDK 17 and no Android SDK discovered under /opt. The upstream CI uses JDK 21, compile SDK 37, and authenticated GitHub Packages for Morphe libraries. Its pull-request workflow targets a nonexistent `dev` branch in this fork and assumes release secrets. Adapt CI to the fork's `main` branch, run tests/lint, and provide a debug artifact when release signing is not provisioned. Publish a manager release only after its independent stable signing setup is verified.

## Reference material

Repository paths above are verified against the reviewed base commit. Primary platform guidance:

- [WorkManager long-running workers](https://developer.android.com/develop/background-work/background-tasks/persistent/how-to/long-running): foreground execution, persisted work, and Android 16 quota constraints.
- [PackageInstaller.SessionParams](https://developer.android.com/reference/android/content/pm/PackageInstaller.SessionParams): silent-update requirements and pending-user-action handling.
- Existing Morphe docs: [updating patched apps](../../updating-patched-apps.md), [batch patching](../../batch-patching.md), [APK download helpers](../../apk-download-helper.md), and [installers](../../installers.md).

## Review outcome

The user approved this architecture on 2026-10-11. The [implementation plan](../plans/2026-10-11-everforge-automation.md) carries the next review and execution-method handoff. No claim is made that mirror adapters, recovery workers, or automatic updates have been built or device-tested.
