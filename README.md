# avd-kernelsu-ci

GitHub Actions builds of **KernelSU Next for the Android Studio emulator (AVD)**.

## Why

Android Studio emulator images ship a frozen, non-GKI-style kernel built from the AOSP
virtual-device config. For `system-images;android-34;google_apis_playstore;arm64-v8a`
the kernel is:

```
Linux version 6.1.23-android14-4-00257-g7e35917775b8-ab9964412
KMI: android14-6.1      ABI: arm64-v8a
```

KernelSU's published prebuilt LKM modules are built against the newest kernel of each KMI
generation (6.1.166 at the time of writing). KernelSU's driver patches the arm64 syscall
dispatcher and registers hooks whose layout is version specific, so on this emulator kernel
the prebuilt module **loads** but never initialises its hooks:

* the module is resident (`/sys/module/kernelsu`)
* no `welcome to KernelSU` kernel message
* `/data/adb/ksud` is never served by the driver
* the KernelSU Next manager reports `Not installed`

Reproduced with both KernelSU Next `v3.4.0` and upstream KernelSU `v3.3.0` prebuilt LKMs.

## What this workflow produces

| Mode | Artifact | Use |
|---|---|---|
| `module` | `kernelsu.ko` | drop into a patched AVD ramdisk (`/kernelsu.ko` + `ksuinit` as `/init`), stock kernel untouched — LKM mode |
| `builtin` | `Image` + all built `*.ko` | boot with `emulator -kernel Image` (GKI mode), replacing the stock ramdisk's `/lib/modules/*.ko` if the version string differs |

Both are built from the emulator's exact kernel commit (`7e35917775b8`), so the compiled
hook layout matches the running kernel.

In `builtin` mode the build forces the exact stock version string
(`CONFIG_LOCALVERSION="-android14-4-00257-g7e35917775b8-ab9964412"`, `LOCALVERSION_AUTO=n`)
so the stock ramdisk's virtio modules keep loading.

## Usage

Actions → *Build KernelSU Next for Android Studio AVD* → Run workflow, or:

```sh
gh workflow run build.yml \
  -f ksu_ref=v3.4.0 \
  -f kernel_commit=7e35917775b8 \
  -f kernel_branch=common-android14-6.1 \
  -f ksu_mode=module
```

## Result

Artifacts: `kernelsu-avd-<ref>-<mode>`, retained 14 days.
