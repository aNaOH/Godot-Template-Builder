# godot-export-templates

CI pipeline for building custom **Godot Engine export templates** from source using GitHub Actions. Each build is driven by a JSON configuration file — no workflow edits needed to add new targets.

---

## Features

- **Config-driven** — one JSON file per template, auto-discovered by the workflow.
- **Windows & Linux** builds (cross-compiled on Ubuntu via MinGW for Windows).
- **Per-config SCons cache** — drastically reduces incremental build times.
- **Optional encryption** — pass a GitHub Secret as the AES-256 key.
- **Custom build profiles** — strip unused modules with Godot's [compilation configuration editor](https://docs.godotengine.org/en/stable/tutorials/editor/using_engine_compilation_configuration_editor.html).
- **Release hardening** — `lto=full` + binary stripping applied automatically for `template_release` targets.
- **Automatic GitHub Releases** — pushes a tagged release with zipped templates when you create a git tag.

---

## Repository structure

```
.
├── .github/
│   └── workflows/
│       └── build-templates.yml   # Main CI workflow
├── configs/                      # ← one JSON per template build
│   ├── windows-release.json
│   ├── windows-release-encrypted.json
│   └── linux-debug.json
├── build_profile.gdbuild         # Optional: default build profile
├── build_profile_minimal.gdbuild # Optional: additional profiles
└── README.md
```

---

## Quickstart

### 1 — Fork / clone this repo

```bash
git clone https://github.com/your-org/godot-export-templates
cd godot-export-templates
```

### 2 — Add a config file

Create a file under `configs/` (name it anything, must end in `.json`):

```jsonc
// configs/windows-release.json
{
  "godot_version": "4.4.1-stable",   // Must match a Godot GitHub tag exactly
  "template_name": "windows-release", // Used for artifact name and cache key
  "target":        "template_release",
  "build_profile": "build_profile.gdbuild", // Path relative to repo root (optional)
  "output_dir":    "dist/windows",    // Informational — actual output goes to artifacts
  "platform":      "windows",         // "windows" | "linuxbsd"
  "encryption_key": "",               // Leave empty or use a secret reference (see below)
  "extra_scons_flags": ""             // Extra raw scons flags, space-separated
}
```

### 3 — Push and watch CI run

The workflow triggers automatically on every push that touches a `configs/**.json` file. Navigate to **Actions** in your repository to follow progress.

---

## Configuration reference

| Field | Required | Description |
|---|---|---|
| `godot_version` | ✅ | Godot version tag, e.g. `4.4.1-stable`. Must be an exact match to the GitHub release tag. |
| `template_name` | ✅ | Unique name for this build. Used as the artifact name, cache key, and output zip name. |
| `target` | ✅ | SCons build target. `template_release` or `template_debug`. |
| `platform` | ✅ | `windows` or `linuxbsd`. |
| `output_dir` | ✅ | Informational output path. The actual output is always uploaded as a GitHub artifact. |
| `build_profile` | ❌ | Path (relative to repo root) to a `.gdbuild` profile. Passed to SCons as `build_profile=<absolute path>`. |
| `encryption_key` | ❌ | AES-256 encryption key. Leave empty to skip. Use `${{ secrets.MY_SECRET }}` syntax to reference a GitHub Secret (see below). |
| `d3d12` | ❌ | `true` (default) or `false`. When `true`, installs the Direct3D 12 SDK via `misc/scripts/install_d3d12_sdk_windows.py` before compiling and enables D3D12 support. Set to `false` to skip the SDK install and pass `d3d12=no` to SCons (Vulkan and OpenGL remain available). Only relevant for `platform: windows`. |
| `extra_scons_flags` | ❌ | Additional raw SCons flags appended verbatim, e.g. `"use_lto=no precision=double"`. |

---

## Encryption keys

To pass an encryption key without exposing it in the config file:

1. Go to **Settings → Secrets and variables → Actions** in your GitHub repository.
2. Create a new secret, e.g. `GODOT_ENCRYPTION_KEY`.
3. In your JSON config, set:

```json
"encryption_key": "${{ secrets.GODOT_ENCRYPTION_KEY }}"
```

The test config uses 81c71fdce0cde1c2d42e8f2357e1c99a9a9efc8aed46be44ee000409f383d109 as the key, if you want to test.

The workflow evaluates this expression at runtime. The key is automatically masked in all logs.

> ⚠️ Remember to build your game project with the same key so the PCK can be decrypted at runtime.

---

## Build profiles

A [build profile](https://docs.godotengine.org/en/stable/tutorials/editor/using_engine_compilation_configuration_editor.html) lets you disable unused Godot modules to produce smaller, faster templates.

1. Use the Godot editor's **Engine Compilation Configuration Editor** (Project → Tools → Engine Compilation Configuration) to generate a `.gdbuild` file.
2. Add it to your repository (at the root or any subdirectory).
3. Reference it in your config: `"build_profile": "my_profile.gdbuild"`.

The workflow resolves the path to an absolute path and passes it directly to SCons as `build_profile=<abs_path>`, so there is no need to copy it into the Godot source tree.

Example `build_profile.gdbuild` (JSON format used by the editor):

```json
{
  "profile": {
    "modules": {
      "mono": false,
      "text_server_adv": false,
      "navigation": false
    }
  }
}
```

---

## Release flags (template_release)

When `target` is `template_release`, the workflow automatically:

| Flag | Effect |
|---|---|
| `lto=full` | Link-time optimisation — smaller, faster binary |
| `strip` | Removes debug symbols from the final binary |

These flags are **not** applied to `template_debug` builds.

---

## SCons cache

Each config gets its own cache bucket keyed on:

```
scons-{template_name}-{godot_version}-{hash(config + build_profile)}
```

This means:
- Changing only the build profile invalidates only its own cache.
- Multiple configs for the same Godot version don't share (and corrupt) each other's caches.
- Cold builds are cached between runs — only changed translation units are recompiled.

The cache is stored in `~/.scons_cache/{template_name}` and capped at **2 GB per config** via `cache_limit=2`.

---

## Triggering builds

### Automatic

The workflow triggers on:
- Any push to `main`/`master` that changes a `configs/**.json` file or the workflow itself.
- Any pull request touching those paths.

### Manual (single or filtered)

Go to **Actions → Build Godot Export Templates → Run workflow**.

Use the `config_filter` input to build a subset of configs:

| Input | Effect |
|---|---|
| *(empty)* | Build all configs |
| `windows-*` | Build only configs whose filename starts with `windows-` |
| `linux-debug` | Build only `configs/linux-debug.json` |

### On git tags (releases)

Push any tag to create a GitHub Release with all templates zipped:

```bash
git tag v1.0.0
git push origin v1.0.0
```

---

## Adding a new platform

Currently supported platforms are `windows` and `linuxbsd`. To add more (e.g. `macos`, `android`, `web`):

1. Add the appropriate system dependencies in the workflow's **Install build dependencies** step.
2. Extend the platform-specific SCons flag blocks in the **Build export template** step.
3. Add the new platform value to the validation list in the **discover** job.

---

## Local build (equivalent command)

If you want to reproduce a build locally, the SCons command for a release build looks like:

```bash
# Inside the Godot source directory
scons platform=windows target=template_release build_profile=/path/to/build_profile.gdbuild lto=full -j$(nproc)

# Then strip
x86_64-w64-mingw32-strip --strip-unneeded bin/godot.windows.template_release.x86_64.exe
```

---

## License

This repository contains only CI configuration and build profiles. Godot Engine itself is licensed under the [MIT License](https://github.com/godotengine/godot/blob/master/LICENSE.txt).
