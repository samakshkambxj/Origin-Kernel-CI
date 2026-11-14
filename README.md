# Origin Kernel CI

Build workflow for [Origin-Kernel](https://github.com/samakshkambxj/Origin-Kernel), the unified
MT6878 kernel for CMF Phone 1 (`Tetris`), CMF Phone 2 Pro (`Galaga`) and Nothing Phone 3a Lite
(`Galaxian`).

Two images are produced per run, both `arm64` and both with full Clang LTO:

| Artifact | Source branch | Config |
|---|---|---|
| `Origin-Reforged-<release>` | `main` | `-Origin-Reforged`, OriginSU + SUSFS + KPM + Baseband-Guard |
| `Origin-Vanilla-<release>` | `vanilla` | `-Origin-Vanilla`, no root solution |

Each artifact contains `Image` (raw arm64 kernel) and its `.sha256`. The image is what
AnyKernel3 packages for flashing; the build does not produce a boot image.

## Triggers

- **Manual** — *Actions → Build Origin Kernel → Run workflow*. Leave `kernel_ref` empty to build
  `main` and `vanilla`, or set it to any branch/tag (for example a release branch) to build that
  ref for both variants.
- **Tag** — pushing a `v*` tag creates a GitHub release on this repository with both images
  attached.

## Toolchain

Neutron Clang, pinned by `NEUTRON_TAG` in the workflow (`06092026` at the time of writing).
The tarball is cached by tag, so bumping the tag is the only change needed to move it:

```yaml
env:
  NEUTRON_TAG: '06092026'
```

Tags come from
[Neutron-Toolchains/clang-build-catalogue](https://github.com/Neutron-Toolchains/clang-build-catalogue/releases)
and are `DDMMYYYY`. The archive extracts to the toolchain root (`bin/clang`, `bin/ld.lld`,
`bin/llvm-*`), so it is unpacked into `$RUNNER_TEMP/toolchain` and `bin` is added to `PATH`.

## Build

Equivalent to the commands in the kernel README:

```sh
make O=out ARCH=arm64 LLVM=1 LLVM_IAS=1 gki_defconfig
make O=out ARCH=arm64 LLVM=1 LLVM_IAS=1 -j"$(nproc)" Image
```

`LLVM=1` is not optional: `CONFIG_LTO_CLANG_FULL=y` is silently dropped when Kconfig is evaluated
without it, and the workflow fails if the resulting `out/.config` does not contain it.

Submodules (`KernelSU`, `susfs4ksu`, `Baseband-guard`) are checked out recursively; `drivers/kernelsu`
and `security/baseband-guard` are tracked symlinks into them.

Full LTO links the whole kernel in one pass, so the workflow adds an 8 GB swap file on the 16 GB
runner and allows up to four hours. A cold build takes roughly an hour; the toolchain cache keeps
repeat runs shorter.

## Local use

The workflow is plain shell and can be reproduced locally by exporting the toolchain path and
running the two `make` invocations above. Nothing in it is GitHub-specific apart from the artifact
upload and the cache.
