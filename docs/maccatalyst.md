# Mac Catalyst Build Notes

This repo includes a Mac Catalyst build preset and XCFramework packaging.
Use this checklist to carry the changes forward when you update to a newer
upstream version.

## Update workflow

1) Sync your fork with upstream:

```sh
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```

2) Update the Mac Catalyst branch:

```sh
git checkout maccatalyst
git rebase main
```

3) Resolve conflicts if any (these files are the usual hotspots):

- `CMakePresets.json`
- `scripts/build_apple_frameworks.sh`
- `scripts/create_frameworks.sh`
- `extension/apple/CMakeLists.txt`
- `tools/cmake/preset/apple_common.cmake`
- `CMakeLists.txt`
- `third-party/CMakeLists.txt`
- `Package.swift`

4) Build and package:

```sh
EXECUTORCH_PYTHON_EXECUTABLE=/path/to/.venv/bin/python EXECUTORCH_FLATC_EXECUTABLE=/opt/homebrew/bin/flatc24 ./scripts/build_apple_frameworks.sh --Release
```

## Cherry-pick alternative

If you prefer cherry-pick instead of rebase, pull the commits from the
`maccatalyst` branch:

```sh
git checkout -b maccatalyst-from-new main
git log --oneline fork/maccatalyst
git cherry-pick <hash1> <hash2> <hash3>
```

## Notes

- The `maccatalyst` preset uses Ninja to enable Swift builds on Mac Catalyst.
- The sentencepiece Mac Catalyst install fix is applied at configure time from
  `tools/cmake/preset/apple_common.cmake`, so no submodule forks are required.
