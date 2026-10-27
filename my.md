```text
+==================================================================+
|                                                                  |
|    B O O T   M E D I A   U T I L I T Y                           |
|                                                                  |
|    ATC-Gen2  /  LS1021A  /  NOR | USB | SD                       |
|                                                                  |
+==================================================================+
```

# Boot Media Utility

[![Platform](https://img.shields.io/badge/Platform-Linux-informational?style=flat-square)](#requirements)
[![Shell](https://img.shields.io/badge/Shell-bash-4EAA25?style=flat-square&logo=gnubash&logoColor=white)](#requirements)
[![Board](https://img.shields.io/badge/Board-ATC--Gen2%20%2F%20LS1021A-blue?style=flat-square)](#output-reference)
[![Distro](https://img.shields.io/badge/Distro-fsl--qoriq-orange?style=flat-square)](#requirements)
[![Media](https://img.shields.io/badge/Media-NOR%20%7C%20USB%20%7C%20SD-success?style=flat-square)](#what-it-produces)

> **TL;DR** — after a successful `bitbake` run:
> `cd BOOT_MEDIA_UTILITY && bash boot_image_utility.sh`

A single script that collects the artifacts produced by a Yocto build of the
`atc-gen2` machine and stages them into three ready-to-use folders — one per
boot medium.

`bitbake` writes its output with a full timestamp in every filename
(`atc-gen2-core-atc-gen2-20261024060846.rootfs.ubifs`) and keeps every older
build alongside it. `boot_image_utility.sh` picks the newest matching file,
renames it to the fixed names the boot flow expects, duplicates it where the
image set needs multiple copies, and skips anything that is not required for a
given medium.

---

## Table of contents

- [What it produces](#what-it-produces)
- [Requirements](#requirements)
- [Usage](#usage)
- [Output reference](#output-reference)
- [Reading the output](#reading-the-output)
- [How "latest" is chosen](#how-latest-is-chosen)
- [Troubleshooting](#troubleshooting)
- [Customizing](#customizing)
- [Limitations](#limitations)

---

## What it produces

| Folder | Boot medium |
| --- | --- |
| `NOR_IMAGES/` | NOR flash image set |
| `USB_IMAGES/` | USB recovery / update set |
| `SD_IMAGES/` | SD card image set |

At a glance:

|  |  |
| --- | --- |
| **Input** | `tmp/deploy/images/atc-gen2` + two `tmp/work` paths + `eth_conf/` |
| **Output** | `NOR_IMAGES/`, `USB_IMAGES/`, `SD_IMAGES/` |
| **Files staged** | 13 NOR · 10 USB · 4 SD |
| **Runtime** | seconds — plain file copies, no compression |
| **Exit code** | `0` all good · `1` something is missing |
| **Re-runnable** | yes — overwrites in place, never deletes |

```text
  SOURCES                               STAGED OUTPUT
  +----------------------------+        +----------------------+
  | tmp/deploy/images/atc-gen2 |        | NOR_IMAGES/   13     |
  | tmp/work/.../rcw           |   cp   | USB_IMAGES/   10     |
  | tmp/work/.../qe-ucode      | ---->  | SD_IMAGES/     4     |
  | eth_conf/                  |        +----------------------+
  +----------------------------+
```

The script copies files only. It does not flash anything, format media, or
mount a target device.

---

## Requirements

- **A completed Yocto build.** The deploy directory must already exist:
  `build_atc-gen2/tmp/deploy/images/atc-gen2`
- **bash** and standard coreutils (`cp`, `ls`, `mkdir`). No extra packages.
- **Linux.** Paths and file modes assume a Linux host or container.
- **No root required** when the source tree is owned by your user. Plain `cp`
  is used, so a permission error surfaces as a `[FAIL]` line.

The script must be run **from the `BOOT_MEDIA_UTILITY/` directory**. Every source
path inside it is relative to the current working directory.

---

## Usage

```bash
cd BOOT_MEDIA_UTILITY

bash boot_image_utility.sh
```

Or, if the file is executable:

```bash
./boot_image_utility.sh
```

Expected layout — the utility sits next to the build directory it reads from:

```
Oriux/
├── build_atc-gen2/                     # Yocto build directory
│   └── tmp/
│       ├── deploy/images/atc-gen2/     # bitbake output (read by the script)
│       └── work/                       # rcw and qe-ucode artifacts (read)
├── eth_conf/
│   └── ethernet_config.bin             # generated, not tracked in git
└── BOOT_MEDIA_UTILITY/
    ├── boot_image_utility.sh
    ├── NOR_IMAGES/                     # created on first run
    ├── USB_IMAGES/                     # created on first run
    └── SD_IMAGES/                      # created on first run
```

```bash
cd Oriux/BOOT_MEDIA_UTILITY
bash boot_image_utility.sh
```

Re-running is safe: output folders are created with `mkdir -p` if missing, and
existing files are overwritten with `cp -f`.

---

## Output reference

### `NOR_IMAGES/`

| Output file | Source |
| --- | --- |
| `rcw_1000.pbl` | `tmp/work/atc_gen2-fsl-linux-gnueabi/rcw/git-r0/git/ls1021atwr/RSR_PPS_70/rcw_1000.bin` |
| `QE.bin` | `tmp/work/cortexa7hf-vfp-neon-fsl-linux-gnueabi/qe-ucode/git-r0/deploy-qe-ucode/boot/iram_Type_A_LS1021a_r1.0.bin` |
| `ethernet_config.eth` | `eth_conf/ethernet_config.bin` |
| `atc-gen2-core-atc-gen2.ubifs1` | newest `atc-gen2-core-atc-gen2*.ubifs*` in the deploy dir |
| `atc-gen2-core-atc-gen2.ubifs2` | same file as `ubifs1` (second copy) |
| `core-image-minimal-atc-gen2-<timestamp>.rootfs.ubifs` | newest matching file, **original name kept** |
| `atc-gen2.dtbd` | `uImage-atc-gen2.dtb` |
| `atc-gen2.dtb1` | `uImage-atc-gen2.dtb` |
| `atc-gen2.dtb2` | `uImage-atc-gen2.dtb` |
| `u-boot-nor-1.0+fslgit-r0.btt` | `u-boot-nor-1.0+fslgit-r0.bin` |
| `zImage.krn` | `zImage` |
| `zImage.kr1` | `zImage` |
| `zImage.kr2` | `zImage` |

The device tree and the kernel are each written under three names; the two
`atc-gen2-core` UBIFS entries are one build copied twice. The
`core-image-minimal` UBIFS is copied under its timestamped name because that is
where its build ID lives.

### `USB_IMAGES/`

Everything from `NOR_IMAGES/` **except** three files:

| Skipped file | Reason printed by the script |
| --- | --- |
| `core-image-minimal-atc-gen2-*.rootfs.ubifs` | not required for USB |
| `atc-gen2.dtbd` | not required for USB |
| `zImage.krn` | not required for USB |

Result: `atc-gen2-core-atc-gen2.ubifs1`, `atc-gen2-core-atc-gen2.ubifs2`,
`atc-gen2.dtb1`, `atc-gen2.dtb2`, `ethernet_config.eth`, `QE.bin`,
`rcw_1000.pbl`, `u-boot-nor-1.0+fslgit-r0.btt`, `zImage.kr1`, `zImage.kr2`.

### `SD_IMAGES/`

| Output file | Source |
| --- | --- |
| `atc-gen2-core-atc-gen2-<timestamp>.rootfs.tar.gz` | newest matching file, name kept |
| `uImage-atc-gen2.dtb` | deploy dir, name kept |
| `u-boot-sdcard-ifc-1.0+fslgit-r0.bin` | deploy dir, name kept |
| `zImage` | deploy dir, name kept |

---

## Reading the output

Every step reports one of three tags:

| Tag | Meaning |
| :---: | --- |
| `[OK]` | File copied successfully |
| `[INFO]` | Deliberately skipped — not required for that medium |
| `[FAIL]` | Source missing, or the copy itself failed |

Each action prints one line:

```
[OK]   <source file> -> <destination path>
[INFO] Skipping <file>; it is not required for USB_IMAGES.
[FAIL] <what was missing or which copy failed>
```

At the end the script prints a summary and a listing of all three folders:

```
SUCCESS: All required images were prepared successfully.
```

…or, when at least one action failed:

```
FAILURE: <n> operation(s) failed.
Review the [FAIL] messages above, resolve the issues, and rerun the script.
```

| Exit code | Meaning |
| --- | --- |
| `0` | Every required image was staged |
| `1` | At least one source was missing or one copy failed |

A failing step does not stop the run — every input is attempted so a single
report shows all problems at once. Fix them and run again.

---

## How "latest" is chosen

Three inputs are selected from the deploy directory rather than named
explicitly:

- `atc-gen2-core-atc-gen2*.ubifs*`
- `core-image-minimal-atc-gen2*.rootfs.ubifs`
- `atc-gen2-core-atc-gen2-*.rootfs.tar.gz`

In each case the script takes the **most recently modified** match
(`ls -t … | head -n 1`), so the newest `bitbake` run wins. Two consequences:

- Copying or touching an old artifact makes it the newest one.
- A build that produced only `.jffs2`/`.ext2.gz` but not `.ubifs` will not be
  picked up — the image type has to exist in the deploy directory.

Everything else (`zImage`, `uImage-atc-gen2.dtb`, the U-Boot binaries, the RCW
and QE files) is a fixed path with no timestamp, so it is always the current
build's output.

---

## Troubleshooting

| Message | Cause | Fix |
| --- | --- | --- |
| `[FAIL] Required RCW binary rcw_1000.bin was not found` | RCW recipe not built | `bitbake rcw` in the build directory, then rerun |
| `[FAIL] Required QE microcode … was not found` | QE microcode recipe not built | `bitbake qe-ucode`, then rerun |
| `[FAIL] … ethernet_config.bin was not found in ../eth_conf/` | The binary is a generated artifact and is not tracked in git; only `eth_confs.c` and a README ship in that folder | Copy or generate `ethernet_config.bin` into `eth_conf/` |
| `[FAIL] No matching … .ubifs image was found` | Deploy dir has no image of that type | Build the image (`bitbake atc-gen2-core` / `bitbake core-image-minimal`), confirm `IMAGE_FSTYPES` includes `ubi` |
| `[FAIL] Required kernel image zImage was not found` | Kernel not built, or wrong `IMG_DIR` | `bitbake virtual/kernel`, then rerun |
| Everything fails with "not found" | Script was run from the wrong directory | `cd` into `BOOT_MEDIA_UTILITY/` first |
| `[FATAL] Failed to create one or more output directories` | No write permission in the utility folder | Run from a writable copy, or fix ownership |
| Nothing was picked up, only old timestamps | Stale `tmp/deploy` from an earlier build | Verify the build actually completed in this tree |

> The comment block at the top of the script mentions `NOR_utility/`. That is a
> historical name — the directory is `BOOT_MEDIA_UTILITY/`. The comment has no
> effect on behavior.

---

## Customizing

The four paths are variables at the top of the script:

```bash
DEST_DIR="NOR_IMAGES"
USB_DIR="USB_IMAGES"
SD_DIR="SD_IMAGES"
IMG_DIR="../build_atc-gen2/tmp/deploy/images/atc-gen2"
```

Point `IMG_DIR` somewhere else to stage from a second build tree without moving
anything. To stage one more file, add a call in the relevant section:

```bash
copy_to "../path/to/input.bin" "output-name.bin" "Message shown if the source is missing."
```

Available helpers:

| Function | Behavior |
| --- | --- |
| `copy_to <src> <dstname> <errmsg>` | Copy one file into `NOR_IMAGES/` under a new name |
| `copy_to_many <src> "<names>" <errmsg>` | Copy one file to several names |
| `copy_latest <dir> <pattern> <dstname> <errmsg>` | Newest match into `NOR_IMAGES/`; empty `dstname` keeps the original name |
| `copy_latest_to_dir <dir> <pattern> <dest_dir> <errmsg>` | Newest match into any folder, name kept |

---

## Limitations

- **Output folders are never cleared.** Files from earlier runs stay behind, so
  an old `core-image-minimal-*.rootfs.ubifs` can sit next to a fresh one. Check
  timestamps in the final listing if you are unsure. To start clean:

  ```bash
  rm -rf NOR_IMAGES USB_IMAGES SD_IMAGES
  ```

- **No integrity check.** Copies are not checksummed; a short read on failing
  media is not detected.
- **No flashing.** This prepares a folder, nothing more.
- **The USB set is derived from `NOR_IMAGES/`**, not from the deploy directory,
  so it inherits whatever that folder contains — including leftovers.

---

<div align="center">

**Boot Media Utility** · ATC-Gen2 (LS1021A) · Oriux BSP

*Documented from `boot_image_utility.sh`.*

</div>
