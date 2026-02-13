# amazon-vod-noads

Patch Amazon Prime Video APKs to skip server-inserted stream ads using ReVanced CLI, while keeping the patch logic reproducible for new APK versions.

## What’s In Here
- `analysis/revanced-patches` (git submodule): upstream ReVanced patches source.
- `patches/revanced-primevideo-skipads.patch`: local diff applied on top of upstream to make the Prime Video skip-ads patch more robust across app versions.
- `analysis/scripts/patch-primevideo-skipads.sh`: end-to-end build + patch script.
- `docs/AI_RUNBOOK.md`: what was changed and how to adapt when a new APK version breaks matching.

## Usage
```bash
analysis/scripts/patch-primevideo-skipads.sh <input.apk> [output.apk]
```

Alternative (no ReVanced CLI; smali injection):
```bash
analysis/scripts/patch-primevideo-skipads-smali.sh <input.apk> [output.apk]
```

Prereqs the script handles (installs to `/tmp`):
- JDK 21 (for Gradle)
- Android SDK (platform 34 + build-tools 34.0.0)

You must provide GitHub Packages credentials for `maven.pkg.github.com/revanced/registry`:
- Create `/tmp/gradle-home-revanced-patches/gradle.properties` containing:
  - `githubPackagesUsername=...`
  - `githubPackagesPassword=...`

Token requirements:
- `githubPackagesPassword` should be a GitHub Personal Access Token (PAT).
- Required scope: `read:packages`.
- Your GitHub account/token must have access to the `revanced/registry` GitHub Packages feed (a random GitHub account may not be sufficient if the packages are restricted).

## License

This project is under an [MIT license](LICENSE.txt).
