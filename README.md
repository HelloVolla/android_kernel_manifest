# Volla Phone Plinius & Plinius Plus (ansuz) - Kernel Manifest

## About

This repository contains the kernel manifest for **Volla Phone Plinius & Plinius Plus (codename: ansuz)**, enabling developers to build and customize the Android kernel for this device.

## Required Repositories

The following repositories are required for building the Volla Phone Plinius kernel, as defined in `ansuz.xml`:

### Kernel Build System
- **[android_kernel_build_kernel](https://github.com/HelloVolla/android_kernel_build_kernel)** - Kernel build and scripts
  - Path: `build/kernel`

- **[android_kernel_build_bazel_mgk_rules](https://github.com/HelloVolla/android_kernel_build_bazel_mgk_rules)** - Bazel build rules for kernel modules
  - Path: `build/bazel_mgk_rules`

### Kernel Sources
- **[android_kernel_volla_mt6878](https://github.com/HelloVolla/android_kernel_volla_mt6878)** - Main kernel source (Linux 6.1)
  - Path: `kernel-6.1`

- **[android_kernel_device_modules_volla_mt6878](https://github.com/HelloVolla/android_kernel_device_modules_volla_mt6878)** - Device-specific kernel modules
  - Path: `kernel_device_modules-6.1`

- **[android_kernel_modules_volla_mt6878](https://github.com/HelloVolla/android_kernel_modules_volla_mt6878)** - MediaTek vendor kernel modules
  - Path: `vendor/mediatek/kernel_modules`

## Getting Started

### Prerequisites

You'll need to be familiar with [Android Source Control Tools](https://source.android.com/setup/develop) and have the following installed:
- Git
- Repo tool
- Build dependencies for kernel compilation

### Initialize Repository

1. Initialize your local repository using the Google kernel manifest:
```bash
repo init -u https://android.googlesource.com/kernel/manifest.git -b common-android14-6.1
```

2. Clone this local manifest repository:
```bash
git clone git@github.com:HelloVolla/android_kernel_manifest.git -b volla-15.0-ansuz .repo/local_manifests
```

3. Sync all repositories:
```bash
repo sync
```

### Building the Kernel

The Volla Phone Plinius kernel uses **Kleaf** (Kernel Build System with Bazel) for building.

To build the GKI kernel image and device-specific kernel modules:
```bash
kernel_device_modules-6.1/build.sh
```

Build artifacts will be available in:
```bash
out/
```

## Project Structure

```
.
├── build/
│   ├── kernel/              # Kernel build scripts and infrastructure
│   └── bazel_mgk_rules/     # Bazel build rules
├── kernel-6.1/              # Main kernel source tree
├── kernel_device_modules-6.1/ # Device-specific modules and build script
├── vendor/
│   └── mediatek/
│       └── kernel_modules/  # MediaTek vendor modules
└── out/                     # Build output directory
```

## Contributing

Contributions are welcome! Please follow the standard Android kernel development practices and submit pull requests to the appropriate repository.

## License

This project is licensed under the Apache License 2.0. See the [LICENSE](LICENSE) file for details.

## Copyright

Copyright (c) 2026 Volla Systeme GmbH

## Support

For issues, questions, or contributions:
- Open an issue on the respective repository
- Contact Maintainer: aryan.sinha@volla.online
- Visit: [volla](https://volla.online)

---

**Note**: This is a local manifest repository. It modifies the base Android kernel manifest to include Volla-specific kernel sources and modules.
