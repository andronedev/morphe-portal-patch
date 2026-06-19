# morphe-portal-patch

Custom [Morphe](https://morphe.software/) patches for Meta Portal apps. Builds a `.mpp` patch
bundle that a Morphe build pipeline loads **alongside** the official `MorpheApp/morphe-patches`
bundle, via a `MY_PATCHES_URL` slot. The patched + signed APKs are installed by
[OpenPortal](https://github.com/andronedev/openportal).

This repo is Portal-scoped: it only holds patches for apps that run on Meta Portal.

## Patches

| Patch | Apps | What it does |
|---|---|---|
| **Custom DPI** | YouTube, YouTube Music | Forces a higher display density for the app only (default 240 dpi, ~1.5x), so the UI scales up on Portal's 160-dpi (mdpi) screen **without** changing the system density. Opt-in (`use = false`); enabled at build time with `-e "Custom DPI"` and tuned with `-O dpi=<value>`. |

### How "Custom DPI" works

Android density is read at runtime in many places, and forcing it on the `Application` alone does
not propagate to Activities (each Activity has its own `Resources` via `ResourcesManager`). So the
patch:

1. Bundles an **extension** (`extensions/youtube`, merged into the APK as `extensions/youtube.mpe`)
   whose `app.morphe.extension.dpi.DensityPatch` registers `ActivityLifecycleCallbacks` and forces
   the density per-Activity in `onActivityPreCreated` (available on API 29 = Portal, runs before any
   view inflation). It also re-applies on configuration changes.
2. The bytecode patch (`patches/.../youtube/dpi`) injects one `invoke-static` into the app's
   `Application.onCreate`, passing the application instance and the `dpi` option value to
   `DensityPatch.init(Application, int)`.

## Layout

```
settings.gradle.kts            plugin app.morphe.patches 1.2.0 (registry MorpheApp/registry) + modules
gradle/libs.versions.toml      morphe-patcher 1.3.3, smali, agp, ...
patches/                       the patch bundle (Kotlin)
  build.gradle.kts             patches { about { ... } } + tasks
  stub/                        Android API stubs (compileOnly)
  src/main/kotlin/app/morphe/
    util/                      reusable bytecode/resource helpers
    patches/youtube/dpi/       Custom DPI: CustomDpiPatch.kt, Constants.kt, Fingerprints.kt
extensions/youtube/            extension module -> extensions/youtube.mpe (DensityPatch.kt)
.github/workflows/release.yml  build -> publish my-patches.mpp on the stable 'latest' release
```

## Build

```bash
./gradlew build        # produces patches/build/libs/patches-<version>.mpp (bundling youtube.mpe)
```

The `app.morphe.patches` plugin and `morphe-patcher` resolve from the GitHub Packages registry
`MorpheApp/registry`, which needs read auth. CI passes `GITHUB_TOKEN` (with `GITHUB_ACTOR`). If that
token cannot read the cross-org registry, set repo secrets `GPR_USER` + `GPR_KEY` (a PAT with
`read:packages`); the release workflow forwards them as `-Pgpr.user`/`-Pgpr.key`.

## Release (stable URL for `MY_PATCHES_URL`)

`.github/workflows/release.yml` builds on push to `main` (or manual dispatch), renames the bundle to
`my-patches.mpp`, and publishes it to a single rolling **`latest`** release. The asset URL is
therefore stable across builds:

```
https://github.com/andronedev/morphe-portal-patch/releases/download/latest/my-patches.mpp
```

## Consuming the bundle

1. Set the `MY_PATCHES_URL` variable in your Morphe build pipeline to the release URL above.
2. The builder downloads it as an extra `-p` bundle and enables this patch with
   `-O dpi=240 -e "Custom DPI"`.
3. `morphe-cli` applies both bundles; the "Custom DPI" patch only matches YouTube / YT Music and is
   ignored for any other app.

## Validation checklist (must run against the real APK + toolchain)

The patch depends on obfuscated runtime structure and on the exact 1.3.3 / 1.2.0 toolchain APIs, so
verify these the first time:

- **Toolchain API**: confirm against the real `morphe-patcher` 1.3.3 sources that `intOption`,
  `Compatibility`/`AppTarget`/`ApkFileType`, `extendWith`, and `findFreeRegister` have the signatures
  used here. If `intOption` differs, fall back to `stringOption` + `toInt`.
- **Extension wiring**: `./gradlew :extensions:youtube:syncExtension` must produce `youtube.mpe`, and
  `:patches:build` must bundle it so `extendWith("extensions/youtube.mpe")` resolves (else the CLI
  throws "Extension not found"). If `:extensions:youtube` is auto-discovered by the plugin, the
  explicit `include(":extensions:youtube")` in `settings.gradle.kts` may need removing.
- **Application hook**: `ApplicationOnCreateFingerprint` matches a class whose superclass is
  `android.app.Application` with `public onCreate()V`. Decompile the base APKs (jadx) to confirm the
  manifest's `<application android:name>` resolves to exactly one such class; if several match (e.g.
  a `:`-process Application), narrow the fingerprint or read the manifest's application name instead.
  If init must run earlier, move the `invoke-static` into `attachBaseContext`.
- **On device (Portal, API 29)**: install the built APK, confirm UI is ~1.5x larger, `adb logcat |
  grep MorpheDpi` shows `init` + per-Activity calls, and `adb shell wm density` still reports 160
  (system density untouched, launcher unchanged). Test `-O dpi=200/320` and an out-of-range value
  (clamps to 240).
- **Rollback**: the patch is `use = false`; dropping `-e "Custom DPI"` from the build restores the
  previous modded APK with zero blast radius.
