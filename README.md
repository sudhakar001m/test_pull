# Oriux — ATC-Gen2 Yocto Build Guide

Steps to build the Yocto binaries for the **ATC-Gen2** board.

---

## Step 1 — Clone the source branch

```bash
git clone https://github.com/FirstView/Oriux.git -b <branch-name>
```

Replace `<branch-name>` with the branch you want to build, for example:

```bash
git clone https://github.com/FirstView/Oriux.git -b source
```

---

## Step 2 — Set up the build environment

Access the path `<build-path>/Oriux`, then source the environment script.

```bash
cd <build-path>/Oriux
```

```bash
source atc-gen2-env -m <build_dir_name>
```
For example:
```bash
source atc-gen2-env -m atc-gen2
```

The build output directory is `build_atc-gen2`.

---

## Step 3 — Build the images

In `build_atc-gen2`, run `bitbake` to build the required images.

**Main BSP image:**

```bash
bitbake atc-gen2-core
```

**Default rootfs:**

```bash
bitbake core-image-minimal
```

Build artifacts are deployed to `build_atc-gen2/tmp/deploy/images/atc-gen2/`.

---

## Quick reference

| Step | Command |
| --- | --- |
| Clone | `git clone https://github.com/FirstView/Oriux.git -b <branch-name>` |
| Environment | `source atc-gen2-env -m atc-gen2` |
| Main image | `bitbake atc-gen2-core` |
| Default rootfs | `bitbake core-image-minimal` |
