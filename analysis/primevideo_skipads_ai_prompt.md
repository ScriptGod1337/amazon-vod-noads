# AI Agent Prompt: Analyze and Patch Prime Video Skip-Ads

You are an AI coding/reverse-engineering agent working in a local workspace with APK files and a cloned ReVanced repository.

## Objective
Analyze and patch Amazon Prime Video APKs so stream ads are skipped safely, using ReVanced Prime Video skip-ads logic as the baseline reference.

## Reference Links (must be used)
- ReVanced patch catalog entry (Prime):
  - https://revanced.app/patches?s=amazon
- Sample patch source to map behavior from (Prime Video Skip Ads):
  - https://github.com/ReVanced/revanced-patches/blob/main/patches/src/main/kotlin/app/revanced/patches/primevideo/ads/SkipAdsPatch.kt
- Related fingerprints:
  - https://github.com/ReVanced/revanced-patches/blob/main/patches/src/main/kotlin/app/revanced/patches/primevideo/ads/Fingerprints.kt
- Related extension implementation:
  - https://github.com/ReVanced/revanced-patches/blob/main/extensions/primevideo/src/main/java/app/revanced/extension/primevideo/ads/SkipAdsPatch.java

## Prime Mapping (what this patch targets)
Map the patch to these Prime classes and behaviors:
- Target method hook:
  - `com.amazon.avod.media.ads.internal.state.ServerInsertedAdBreakState.enter(Lcom/amazon/avod/fsm/Trigger;)V`
- Required trigger API:
  - `AdBreakTrigger.getSeekStartPosition()`
  - `AdBreakTrigger.getSeekTarget()`
  - `AdBreakTrigger.getBreak()`
- Required ad-break API:
  - `AdBreak.getDurationExcludingAux()`
- Required player/time API:
  - `VideoPlayer.getCurrentPosition()`
  - `VideoPlayer.seekTo(long)`
  - `TimeSpan.getTotalMilliseconds()`
- State machine transition:
  - `new SimpleTrigger(AdEnabledPlayerTriggerType.NO_MORE_ADS_SKIP_TRANSITION)`
  - `StateBase.doTrigger(Trigger)`

## Required Workflow
1. Identify APK targets (old known-good + latest).
2. Decompile both APKs (smali-level is sufficient for patching).
3. Compare target method shape in both versions.
4. Decide if upstream fingerprint is still valid.
5. If incompatible, adapt patch anchor safely.
6. Implement patch pipeline script:
   - decompile
   - inject smali
   - rebuild
   - sign
7. Add strict fail-safe checks:
   - required symbol/class/method existence checks
   - anchor state classification: `clean`, `patched`, `incompatible`
   - stop on `patched` (prevent double patch)
   - stop on `incompatible`
8. Add post-injection verification checks.
9. Add runtime fail-open behavior in injected code:
   - if unexpected/null trigger data, jump back to original method flow
   - do NOT crash playback
10. Validate resulting APK:
   - zipalign/signature verified
   - patched block present in rebuilt smali
11. Provide exact commands and outputs summary.

## Safety Requirements
- Never silently patch if anchor does not match expected shape.
- Prefer preserving original logic if any ambiguity exists.
- Any detected incompatibility must hard-fail with explicit reason.
- Prevent accidental re-patching of already patched APK.

## Signing Requirements
- Support default debug signing.
- Support optional custom keystore signing:
  - `--keystore`
  - `--ks-alias`
  - `--ks-pass`
  - `--key-pass`

## Runtime/Test Notes
- Same package name cannot be installed side-by-side with official Prime app unless signatures match.
- For testing patched APK with same package name, uninstall official app first.
- If crash occurs, capture logcat and iterate on smallest failing assumption (anchor/register/null-path/state transition).

## Known Latest-Version Hint (3.0.438.2347)
- Symptom seen in latest patch attempts:
  - app can crash during playback after injection.
- Likely cause:
  - injected skip logic assumes trigger/break/timespan objects are always valid and forces the ad-skip path too early.
- Fix pattern that worked:
  - keep the same anchor (right after `getPrimaryPlayer()` `move-result-object`),
  - add fail-open guards before using objects:
    - `if-eqz p1` -> jump back to original method flow,
    - `if-eqz getSeekTarget()` -> jump back,
    - `if-eqz getBreak()` -> jump back,
    - `if-eqz getDurationExcludingAux()` -> jump back,
  - only execute `seekTo(...) + doTrigger(NO_MORE_ADS_SKIP_TRANSITION)` when guards pass,
  - otherwise continue original method logic unchanged.
- Implementation note:
  - use a dedicated fallback label (example: `:rvd_skip_ads_original`) that lands before original instructions (not at method end).

## Existing Local Script (use/update this first)
- `analysis/scripts/patch-primevideo-skipads.sh`

## Definition of Done
- Patch script produces a signed patched APK for target version.
- Playback no longer crashes.
- Ad-skip logic is injected and verified in smali.
- Script fails safely on incompatible versions.
- Summary includes exact file changes, commands used, and generated APK paths.
