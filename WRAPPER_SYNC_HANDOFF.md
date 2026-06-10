# Wrapper-Sync Automation — Flutter Handoff Doc

**Status as of:** 2026-06-10 — **validated end-to-end on the fork.** A full both-platform sync (Android `8.1.0→8.3.0` + iOS `7.6.0→7.7.1`) ran green and opened a structured PR with re-implemented APIs, version bumps, and a CHANGELOG entry. Polish fixes (clean PR body + cost/changelog metadata, gitignored build noise) and the changelog **recall pass** are shipped.
**Owner:** @piyush-kukadiya
**Audience:** future Claude Code sessions starting fresh in this repo, and new developers picking up the Flutter wrapper-sync.

---

## Read this first if you're new (Claude or human)

This is the single source of truth for "where the Flutter wrapper-sync is right now." Fresh Claude session: read top to bottom (~10 min). New developer: read **Overview** + **Workflow structure**, then **What's DONE / PENDING**.

Related, reusable doc: **`CleverTap/clevertap-wrapper-tooling/.claude/skills/onboard-wrapper-sync/SKILL.md`** — the generic playbook for adding a *new* hybrid wrapper (Cordova is next). This Flutter doc is the wrapper-specific companion to that generic skill.

---

## Overview — what this is

When CleverTap ships new native SDKs (`clevertap-android-sdk`, `CleverTap-iOS-SDK`), the Flutter wrapper must catch up: bump version pins, surface new public APIs through the MethodChannel bridge, propagate minSdk/permission changes. This automates it: a maintainer clicks **one button** in the Flutter repo's Actions UI, picks the target native versions, and Claude runs headless in CI — diffs the native SDKs, implements the new wrappers across Dart/Android/iOS, bumps versions, writes the CHANGELOG, builds the example app, and opens a PR. **The only human step is reviewing the PR.**

Why "click a button" not fully auto: native releases are sometimes Android-only or iOS-only; the maintainer decides when to sync, and one dispatch handles "Android only", "iOS only", or "both".

The **local `/update-sdk` command** (the interactive, multi-agent version under `.claude/`) is **untouched** — it still works on a laptop with its human confirmation gates. The CI path is a separate, headless door into the same knowledge (the 5 skills).

---

## Workflow structure

The reusable workflow is a **thin conductor** that calls **composite actions**. Wrapper-specific bits (the build) live in their own files; everything else is shared.

```
setup (composite)            mint App token, checkout wrapper @ base_ref, branch, install Claude CLI
   ↓
build/flutter (pre-sync)     flutter pub get + flutter build apk / build ios  (on unchanged checkout)
   ↓  pre-sync build gate → if a build failed, ABORT here ($0 Claude spent)
claude-sync (android)        claude -p sync-orchestrator-flutter.md  (Claude edits files)
claude-sync (ios)            claude -p sync-orchestrator-flutter.md
   ↓  if a Sync step fails → no commit/PR; Slack ping
build/flutter (post-sync)    rebuild on Claude's edits  (failure does NOT block the PR — Option A)
   ↓
open-pr (composite)          Claude writes the PR body, applies labels, gh pr create --base <base_ref>
   ↓
cost report  +  Slack-on-failure  +  upload sync logs / APK / .app artifacts
```

### Dispatch inputs
- **`android_module` / `android_version`**, **`ios_module` / `ios_version`** — what to sync. Empty version skips that platform.
- **`skip_sync`** (bool, default false) — skip Claude + post-sync builds; pre-sync builds still run. Iterate on the build pipeline at **$0 Claude cost**.
- **`model`** (sonnet/opus/haiku, default sonnet).
- **`release_name`** (optional) — branch suffix; defaults to today's date. **Use a unique value per run** (the workflow bails if `task/release_<name>` already exists).
- **`base_ref`** (default `develop`) — which wrapper branch to sync from and open the PR against. **Point it at a test baseline branch for testing** (see Testing).

### Option A (post-sync build failure)
If Claude's edits compile-fail in the post-sync build, the PR **opens anyway** with a `build-failed` label; the run is marked failed (Slack pings). Reviewer pulls the branch, fixes, pushes.

---

## System architecture at a glance

```
Native SDK repos (release on their own schedule)
  • CleverTap/clevertap-android-sdk   (tags: corev8.3.0, …)
  • CleverTap/clevertap-ios-sdk       (tags: 7.7.1, …)
        │
        ▼  maintainer clicks Run on:  CleverTap/clevertap-flutter → Actions → native-release-sync
        │  (workflow_dispatch: android/ios module+version, base_ref, skip_sync, model)
        ▼
Reusable workflow  CleverTap/clevertap-wrapper-tooling/.github/workflows/sync.yml@v1
  setup → build/flutter(pre) → claude-sync(android, ios) → build/flutter(post) → open-pr → cost/slack/artifacts
        │   (composite actions under .github/actions/; Flutter prompt prompts/sync-orchestrator-flutter.md)
        ▼
PR opens on the wrapper repo against <base_ref>
  • branch task/release_<name> • author clevertap-wrapper-sync[bot]
  • body = structured sync log (surfaced / skipped / deferred / flagged-for-review / cost / native changelog)
```

---

## The two repos

### 1. `CleverTap/clevertap-flutter` (this repo)
- `.github/workflows/native-release-sync.yml` — the dispatch button (routes to the reusable workflow; `wrapper: flutter`).
- `.claude/skills/` — the **5 domain skills** Claude reuses headlessly (see below). Also `.claude/commands/update-sdk.md` + agents = the **local interactive** path (untouched by CI).
- Bridge code the automation reads/edits: `lib/clevertap_plugin.dart` (Dart), `android/src/main/java/com/clevertap/clevertap_plugin/DartToNativePlatformCommunicator.kt` (Android), `ios/Classes/CleverTapPlugin.m` (iOS), `example/lib/main.dart` (demos).
- `WRAPPER_SYNC_HANDOFF.md` (this file).

### 2. `CleverTap/clevertap-wrapper-tooling` (org-owned, public)
Shared CI logic so every wrapper doesn't keep its own copy.
```
.github/workflows/sync.yml                  reusable conductor (branches on inputs.wrapper)
.github/actions/setup/                       shared: token, checkout, branch, Claude CLI
.github/actions/build/flutter/               Flutter build (pre/post) + google-services stub
.github/actions/build/react-native/          RN build (npm, gradle, xcodebuild, Podfile patch)
.github/actions/claude-sync/                 shared: headless `claude -p` for one platform
.github/actions/open-pr/                      shared: PR body + labels + gh pr create
prompts/sync-orchestrator-flutter.md          Flutter auto-pilot prompt (reuses the 5 skills)
prompts/sync-orchestrator.md                  RN auto-pilot prompt
prompts/pr-description.md                     shared PR-body prompt (renders flagged_for_review)
tools/diff_native_api.py                      the native SDK differ (Python, stdlib only)
scripts/{open-combined-pr,compute-cost,slack-notify}.sh
.claude/skills/onboard-wrapper-sync/          the generic "add a new wrapper" playbook
```
**Versioning:** `@v1` is a moving tag; wrapper dispatches pin to it. `uses:` does NOT follow org redirects — after the org move, every wrapper's dispatch had to switch to `CleverTap/...`.

---

## How Claude is driven for Flutter

CI runs a **headless auto-pilot prompt** (`prompts/sync-orchestrator-flutter.md`) — NOT the local `/update-sdk` command. The prompt:
- Tells Claude there's no human; any skill instruction to "wait for the user / Approved-Hold" → proceed and record.
- Reads the **5 existing Flutter skills by path** for the real conventions:
  - `version-detection` — the **7 version locations** + the `libVersion` integer (`4.2.0`→`40200`)
  - `api-wrapper-patterns` — Dart/Android/iOS wrapper patterns + type mapping
  - `native-sdk-changelog-analysis` — change categorization + decision tree + type checks
  - `example-app-patterns` — demo buttons in `example/lib/main.dart`
  - `changelog-generation` — strict CHANGELOG format + version anchors
- Uses the **diff tool (`diff_native_api.py`) as ground truth** (`api_diff` + `build_manifest` + verbatim `changelog`), then runs a **step-3b changelog recall pass**: any public API the changelog names that the regex diff missed is **source-verified** against the new native header and acted on (implement / remove / keep-deprecated / update-signature); behavior-only or unconfirmable items go to **`flagged_for_review`**, rendered at the top of the PR.

---

## Flutter-specific gotchas (hard-won — don't relitigate)

- **7 version locations** must move together (see the `version-detection` skill): `pubspec.yaml`, `android/build.gradle` (top `version` **and** the `clevertap-android-sdk` pin), `ios/clevertap_plugin.podspec` (top `s.version` **and** the `CleverTap-iOS-SDK` pin), `lib/clevertap_plugin.dart` (`libVersion` integer), `README.md`.
- **google-services.json is gitignored** → absent on a fresh checkout → the Google Services plugin fails the Android build. The `build/flutter` composite **writes a CI stub** with `package_name: com.example.clevertap_plugin_example` (must match the example's `applicationId`).
- **No Podfile patch needed** (unlike RN). The Flutter example's `ios/Podfile` already has `use_modular_headers!` globally.
- **iOS pin change ⇒ delete `example/ios/Podfile.lock`.** A rolled-back/changed podspec conflicts with the committed lock (`CocoaPods specs too out-of-date`). Post-sync deletes it automatically; for a hand-made baseline you must delete it yourself.
- **Ephemeral build files** (`example/ios/Flutter/ephemeral/`) are gitignored so they don't pollute the PR.

---

## What's DONE ✅
- Reusable workflow refactored into composite actions; `wrapper`-aware conductor; RN moved into `build/react-native` (regression-tested, green).
- Flutter dispatch + `build/flutter` composite + `sync-orchestrator-flutter.md` (reuses the 5 skills).
- `base_ref` knob for testing against baselines; PR base parameterized.
- **Validated:** `skip_sync` pipeline run (green), Android-only full run → PR #1, **both-platform full run → PR #2** (re-implemented `fetchInbox`/`fetchInboxWithCallback`/`pushDisplayUnitElementClickedEvent` across Dart+Android+iOS, bumped 7 locations `4.1.0→4.2.0`, wrote CHANGELOG; ~$3.09, ~27 min).
- Polish: clean PR body (no preamble) + cost/token/native-changelog metadata (sync logs inlined into the PR-body prompt); ephemeral files gitignored.
- **Changelog recall pass + `flagged_for_review`** added (and ported to RN).
- App + 4 secrets installed on the fork.

## What's PENDING 🟡
- **Production rollout:** add `native-release-sync.yml` to the real `CleverTap/clevertap-flutter` (via PR); install the App + 4 secrets on the real repo; confirm `develop` exists.
- **Real cost cap:** today the `$3` cap is **informational only** (a post-run PR comment, nothing aborts). A true guardrail would be a between-platform hard stop + `--max-turns` on the sync calls.
- **Run the recall pass live** — PR #1/#2 predate it; a fresh run will exercise step-3b + `flagged_for_review`.
- **Cosmetic:** a couple of generated files (`example/ios/Flutter/AppFrameworkInfo.plist`, `gradle.properties` migrator flags) still appear in PRs; gitignore the plist if undesired.

---

## Key decisions and why
| Decision | Rationale |
|---|---|
| Headless auto-pilot prompt (not the `/update-sdk` command) in CI | Command has human-approval gates + spawns 7 sub-agents; a single headless prompt reusing the same 5 skills is simpler, proven (RN), and emits structured JSON. Local command stays intact. |
| Composite actions (not one monolith) | Shared steps written once; only the per-wrapper build lives in its own file. `uses:` can't take expressions, so wrapper selection is `if:`-guarded steps. |
| Diff tool as ground truth + changelog recall pass | Structural diff is precise but ~80% (regex) — the recall pass + source-verify recovers misses without inventing APIs. |
| `base_ref` knob | Test any old→new gap against a baseline branch without touching `develop`. |
| Old release tags NOT used as test baselines | They don't build on today's toolchain (Flutter "Built-in Kotlin" migration). Use `develop` + rolled-back pins. |
| Soft cost cap, informational | Don't fail mid-PR-creation; report and let humans act. |

---

## File inventory
| Path | What |
|---|---|
| `.github/workflows/native-release-sync.yml` | dispatch button |
| `.claude/skills/{version-detection,api-wrapper-patterns,native-sdk-changelog-analysis,example-app-patterns,changelog-generation}/` | the 5 domain skills (reused by CI prompt + local command) |
| `.claude/commands/update-sdk.md` + `.claude/agents/*` | local interactive path (untouched by CI) |
| `lib/clevertap_plugin.dart` / `…/DartToNativePlatformCommunicator.kt` / `ios/Classes/CleverTapPlugin.m` / `example/lib/main.dart` | bridge layers the automation edits |
| (tooling repo) `prompts/sync-orchestrator-flutter.md`, `.github/actions/build/flutter/`, `tools/diff_native_api.py` | Flutter prompt, build composite, diff tool |

---

## How to pick up where we left off

### Fresh Claude session
1. Read this doc, then the generic skill `clevertap-wrapper-tooling/.claude/skills/onboard-wrapper-sync/SKILL.md`.
2. Check current state:
   ```bash
   gh secret list --repo piyush-kukadiya/clevertap-flutter
   gh run list --repo piyush-kukadiya/clevertap-flutter --workflow native-release-sync.yml --limit 5
   gh api repos/CleverTap/clevertap-wrapper-tooling/git/refs/tags/v1 --jq '.object.sha'
   ```
3. Ask the user what's next (run the recall pass live / production rollout / Cordova).

### New developer
1. Read this doc + the onboarding skill.
2. Get access to `CleverTap/clevertap-flutter` (read) and the fork (full); get the Anthropic API key from the CleverTap-org owner.
3. Walk a fork test (`skip_sync=true` first) to get hands-on.

### Useful commands
```bash
# Pipeline-only (no Claude cost)
gh workflow run native-release-sync.yml --repo piyush-kukadiya/clevertap-flutter \
  --ref task/setup-sync-automation \
  -f base_ref=test/baseline-impl -f android_module=core -f android_version=8.3.0 \
  -f ios_module=none -f skip_sync=true -f release_name=smoke-$(date -u +%H%M%S)

# Full sync (Claude does the work, opens a PR)
gh workflow run native-release-sync.yml --repo piyush-kukadiya/clevertap-flutter \
  --ref task/setup-sync-automation \
  -f base_ref=test/baseline-impl -f android_module=core -f android_version=8.3.0 \
  -f ios_module=core -f ios_version=7.7.1 -f release_name=full-$(date -u +%H%M%S)

# gh run watch is flaky on slow networks; a resilient poll is more reliable:
RID=<run-id>; REPO=piyush-kukadiya/clevertap-flutter
for i in $(seq 1 70); do s=$(gh run view $RID --repo $REPO --json status,conclusion \
  --jq '.status+"|"+(.conclusion//"")' 2>/dev/null); echo "$s"; \
  [ "${s%%|*}" = "completed" ] && break; sleep 30; done

# Run the diff tool locally
python3 .../clevertap-wrapper-tooling/tools/diff_native_api.py \
  --platform android --module core --old-version 8.1.0 --new-version 8.3.0
```

---

## Known issues / lessons learned
- **`uses:` doesn't follow org redirects.** After moving the tooling repo to the CleverTap org, every wrapper dispatch had to change `uses:` to `CleverTap/...@v1` (a redirect 422s the dispatch).
- **Old release tags don't build on current toolchain** → use a `develop`-based baseline with rolled-back pins for testing.
- **PR-body Claude can't read files outside its cwd** — so the sync logs are inlined into the PR-body prompt (don't revert to path-passing, or cost/native-changelog metadata goes missing).
- **`gh run watch` dies on transient network blips** — prefer the resilient poll loop above.
- **Unique `release_name` per run** or the branch-exists check bails.
- **Cross-run shape variance** — Claude may produce slightly different (still-correct) wrapper shapes across independent runs (e.g. merging an overload vs. keeping two methods). Normal; review accordingly.

---

_If this doc is stale by the time you read it, update it as part of your change. It's meant to reflect "now."_
