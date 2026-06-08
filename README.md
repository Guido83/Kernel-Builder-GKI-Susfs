# Custom GKI CI/CD Runner

An automated GitHub Actions workflow for building custom Android Generic Kernel Images (GKI). While explicitly tested on the Pixel 6 and Pixel 9/10 Pro XL, this pipeline is designed to be highly compatible with **any device that supports standard GKI architectures**. 

This runner caters to advanced kernel modifications, supporting automated pulling, patching, and repacking of boot images. It is modular by design, allowing developers to inject root solutions, mask modifications, and compile network/performance modules seamlessly.

## Core Features

* **Automated Source Syncing:** Fetches pure AOSP/Pixel kernel branches directly from Google's manifest.
* **Multi-Variant Root Integration & SUSFS:** Automated cloning, branch verification, and patching for kernel-level masking. Supports KernelSU, KernelSU-Next, and SukiSU-Ultra out of the box.
* **CI Symmetry Enforced:** Dynamically calculates upstream divergence points to lock manager compilation version strings to exact commits, maintaining parity with official apps.
* **Rejection Resolution:** Built-in hooks to automatically resolve known SUSFS patch rejections in the common tree.
* **Environment Sanitizing:** Neutralizes ABI protected exports and uses the official Google commit hash and Unix timestamp to facilitate build environment and kernel string integrity.
* **WireGuard Support:** Optional native WireGuard integration with ARM64 hardware crypto acceleration and Android Netd routing hooks.
* **Targeted Payload Extraction:** Scans the remote OTA URL and streams only the `boot` partition directly from the source via `payload_dumper`, completely bypassing the need to download the massive multi-gigabyte OTA ZIP.
* **Automated Packaging:** Hot-swaps the compiled kernel `Image` into the stock boot image using `magiskboot`, or automatically generates an AnyKernel3 flashable zip if no OTA is provided.

## Workflow Inputs

Trigger the workflow manually via the **Actions** tab. The pipeline accepts the following variables:

| Input | Default | Description |
| :--- | :--- | :--- |
| `build_name` | `gki-custom` | Identifier for the build artifact (e.g., `komodo-cp1a`). |
| `root_manager` | `KernelSU-Next` | Select the Root Environment to integrate (`KernelSU`, `KernelSU-Next`, `SukiSU-Ultra`). |
| `enable_ksu_susfs` | `true` | Toggles root manager and SUSFS integrations. Uncheck for a pure stock build. |
| `ota_url` | `""` | Direct link to the official OTA `.zip`. Required if you want the runner to automatically repack the kernel into a flashable `boot.img`. |
| `manifest_branch` | `common-android14-6.1-2025-09` | The kernel manifest branch (e.g., use `common-android15-6.6-2025-10` for the P10 series). |
| `susfs_branch` | `gki-android14-6.1-dev` | The specific SUSFS branch to pull. |
| `build_wg` | `false` | Injects custom Kconfig integrations (Bazel fragments or legacy Make) to compile WireGuard natively into the kernel. |

> **Note on Repacking:** If an `ota_url` is omitted or invalid, or the ZIP does not contain a standard `payload.bin` at its root, the runner will automatically degrade gracefully to generating an AnyKernel3 (AK3) flashable zip containing your compiled kernel `Image`, completely skipping the stock boot image repacking phase.

## Manager APK Locator Instructions

Because this pipeline dynamically locks the CI version strings to the exact upstream commit prior to custom modifications, you **must** use the official Manager APK compiled from that specific commit to ensure version symmetry. 

During the build process, check the GitHub Actions logs (click ✅ build), under the **"Integrate Root Manager"** step. The runner will output a Manager APK Locator link:

1. Navigate to the provided URL in the logs (e.g., `https://github.com/<Upstream-Repo>/commit/<hash>`).
2. Click the **green checkmark (✅)** next to the commit hash.
3. Click **Details** on the build action to obtain the run number next to the # at the end.
4. Click Actions, click on the corresponding run and scroll down to **Artifacts** to download the perfectly matched Manager APK for your build.

## Repository Structure

The workflow delegates tasks to specialized scripts to keep the YAML clean and modular:

* `scripts/build_kernel.sh`: Wraps the Kleaf/Bazel build process (with a legacy 5.10 Hermetic Make fallback), injects Google identity cloaking variables, and outputs a configuration validation report.
* `scripts/configure_kconfigs.sh`: Neutralizes ABI protected exports and handles the generation/injection of Kconfig fragments for WireGuard and Netd hooks.
* `scripts/inject_ksu_variant.sh`: Handles the cloning and integration of your chosen Root Manager variant, calculates upstream divergence for version locking, and establishes necessary symlinks.
* `scripts/integrate_susfs_next.sh`: Clones the designated SUSFS variant and handles common-side file transfers and patching.
* `scripts/custom_patches.sh`: A blank canvas executed prior to compilation. Use this to apply standard kernel tweaks, such as custom CPU governors or scheduler modifications. 
* `scripts/fix_susfs_rejections.sh`: A targeted `sed` routine to force-inject SUSFS headers into `exec.c`, `base.c`, and `namespace.c` if standard patching fails.
* `scripts/validate_ota.py` & `scripts/ota_pull.py`: Evaluates the OTA URL and executes the payload extraction.
* `scripts/boot_swap.sh`: Wraps Magiskboot to unpack the stock image, swap the core kernel, and repack the final artifact.
