# Cesium for Unreal custom fork maintenance

This branch (`custom/terrain-deform`) is the place for terrain depression / water-region changes.
Keep the fork's `main` branch tracking upstream `CesiumGS/cesium-unreal` without custom commits.

## Baseline

- Started from upstream stable release `v2.31.0`, commit `7792b53600262735361532a67f8fb8f3b3bb0f16`.
- The `extern/cesium-native` URL in this branch's `.gitmodules` is absolute because the upstream relative path would resolve against the personal fork's owner and can fail during submodule initialization.
- No terrain deformation has been implemented yet.

## Initial checkout (Windows PowerShell)

```powershell
git clone --branch custom/terrain-deform https://github.com/wyd111/cesium-unreal.git
cd cesium-unreal
git remote add upstream https://github.com/CesiumGS/cesium-unreal.git

git submodule sync --recursive
git submodule update --init --recursive
```

## Sync upstream release

Fetch upstream releases then merge a **tested stable release tag**, not arbitrary unreleased commits, into the custom branch.

```powershell
git fetch upstream --tags

git switch custom/terrain-deform
git merge v2.32.0  # Example: run only AFTER this version is officially released

git submodule sync --recursive
git submodule update --init --recursive

# Rebuild cesium-native for each required UE target platform if its submodule SHA changed.
# Rebuild the plugin; verify Windows Editor / Android against the actual UE version.

git push origin custom/terrain-deform
```

If the official merge changed `.gitmodules`, **retain the absolute native URL** for the personal fork.
If an official merge changes Cesium internals near deformation hooks, resolve conflicts manually and verify functionality.

## Optional: fast-forward fork main to upstream main

```powershell
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main
```

Do NOT put terrain deformation changes in `main`; do not force-push the `custom/terrain-deform` branch.

## Consuming from OceanView

Pin the fork as a Git submodule in OceanView at a tested commit, rather than automatically using the newest custom branch HEAD. Do not keep two active copies of the `CesiumForUnreal` plugin in both your project and engine installation.

Official source: https://github.com/CesiumGS/cesium-unreal
Windows development instructions: https://cesium.com/learn/cesium-unreal/ref-doc/developer-setup-windows.html
