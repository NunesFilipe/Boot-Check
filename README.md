<div align="center">

<h1>🖥️ Boot-Check</h1>

<p><strong>Test bootable USB drives directly from your terminal — no VirtualBox, no GUI, no rebooting.</strong></p>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Open Source](https://img.shields.io/badge/Open%20Source-%E2%9D%A4-red)](https://github.com/NunesFilipe/Boot-Check)
[![Shell: Bash](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnubash&logoColor=white)](https://www.gnu.org/software/bash/)
[![Powered by QEMU](https://img.shields.io/badge/Powered%20by-QEMU-orange)](https://www.qemu.org/)
[![Platform: Linux](https://img.shields.io/badge/Platform-Linux-blue?logo=linux)](https://kernel.org/)

</div>

---

## Overview

**Boot-Check** is a lightweight, open-source Bash TUI tool that lets you boot and test any physical USB drive using **QEMU** — directly from your terminal, in seconds.

It automatically detects connected USB drives, lets you choose between **Legacy BIOS** (MBR) and **UEFI** (GPT/EFI) boot modes, and launches a QEMU window simulating a real boot from that device.

> No virtual machine setup. No rebooting. Just plug and test.

---

## Features

| Feature | Description |
|---|---|
| 🔍 Auto-detection | Automatically lists all connected USB block devices |
| 🖥️ Dual boot mode | Supports Legacy BIOS (MBR) and UEFI (OVMF/EFI) |
| ⚡ KVM acceleration | Near-native boot speed with hardware virtualization |
| 🎨 Colorful TUI | Intuitive terminal interface with keyboard navigation |
| 🔄 Rescan | Rescan USB devices without restarting the tool |
| 🛡️ Safe | Read-only raw disk access — no data is written |
| 📦 Minimal deps | Only requires QEMU and standard Linux tools |

---

## Requirements

| Dependency | Arch Linux | Debian / Ubuntu | Fedora |
|---|---|---|---|
| `qemu-system-x86_64` | `qemu-full` | `qemu-system-x86` | `qemu-system-x86` |
| `lsblk` | `util-linux` *(pre-installed)* | `util-linux` *(pre-installed)* | `util-linux` *(pre-installed)* |
| OVMF *(UEFI only)* | `edk2-ovmf` | `ovmf` | `edk2-ovmf` |

> **KVM** is strongly recommended. Your CPU must support Intel VT-x or AMD-V, and the `kvm` kernel module must be loaded.

---

## Installation

### 1. Install dependencies

**Arch Linux / Manjaro:**
```bash
sudo pacman -S qemu-full edk2-ovmf
```

**Debian / Ubuntu / Mint:**
```bash
sudo apt install qemu-system-x86 ovmf
```

**Fedora:**
```bash
sudo dnf install qemu-system-x86 edk2-ovmf
```

### 2. Clone the repository

```bash
git clone https://github.com/NunesFilipe/Boot-Check.git
cd Boot-Check
```

### 3. Install globally

```bash
sudo cp checkusb /usr/local/bin/checkusb
sudo chmod +x /usr/local/bin/checkusb
```

### 4. (Recommended) Grant USB access without sudo

```bash
sudo usermod -aG disk $USER
```

> Log out and back in for the group change to take effect.

---

## Usage

```bash
checkusb
```

### Key bindings

| Key | Action |
|---|---|
| `1` – `N` | Select a USB drive |
| `L` | Boot in Legacy BIOS mode (MBR) |
| `U` | Boot in UEFI mode (GPT/EFI) |
| `R` | Rescan connected USB devices |
| `V` | Go back to the previous menu |
| `Q` | Quit |
| `Ctrl+C` | Force quit at any time |

To stop a boot session, close the QEMU window or press `Ctrl+Alt+Q` inside it.

---

## How It Works

1. **Detection** — Uses `lsblk -d -P -o NAME,SIZE,TRAN` to list USB block devices (whole disks only, no partitions). Fields are parsed with Bash's `=~` operator and `BASH_REMATCH` — no external subprocesses needed.
2. **Model info** — Reads `/sys/block/<device>/device/model` for the human-readable drive name. The name is sanitized by `sanitize_model_name` before display.
3. **Validation** — Before launching, `validate_launch_params` checks the device path format, RAM range, and OVMF firmware availability.
4. **QEMU launch** — `build_qemu_args` constructs the argument array based on the selected boot mode:
   - **Legacy BIOS**: `-machine type=pc` with KVM acceleration. Drive is mounted read-only.
   - **UEFI**: `-machine type=q35` with `-bios /path/to/OVMF.fd`. Drive is mounted read-only.
5. **Permissions** — If the device isn't readable by the current user, `sudo` is invoked automatically.

---

## OVMF Firmware Path

By default, Boot-Check looks for OVMF at:

```
/usr/share/edk2/x64/OVMF.4m.fd
```

If your distro places it elsewhere, update `OVMF_PATH` at the top of the `checkusb` script:

| Distro | Path |
|---|---|
| Arch Linux | `/usr/share/edk2/x64/OVMF.4m.fd` |
| Debian / Ubuntu | `/usr/share/OVMF/OVMF_CODE.fd` |
| Fedora | `/usr/share/edk2/ovmf/OVMF_CODE.fd` |

```bash
# Example for Debian/Ubuntu — edit at the top of the checkusb script:
OVMF_PATH="/usr/share/OVMF/OVMF_CODE.fd"
```

---

## Troubleshooting

<details>
<summary><strong>Permission denied on /dev/sdX</strong></summary>

Add your user to the `disk` group:

```bash
sudo usermod -aG disk $USER
```

Then log out and back in. Verify with:

```bash
groups $USER
```

</details>

<details>
<summary><strong>QEMU window doesn't open</strong></summary>

Make sure your display server (X11 or XWayland) is running and QEMU was compiled with SDL support:

```bash
qemu-system-x86_64 -display help
```

</details>

<details>
<summary><strong>KVM not available</strong></summary>

Check if the module is loaded:

```bash
lsmod | grep kvm
```

Load it manually if needed:

```bash
sudo modprobe kvm_intel   # Intel
sudo modprobe kvm_amd     # AMD
```

</details>

<details>
<summary><strong>UEFI firmware not found</strong></summary>

Install the OVMF package for your distro and update `OVMF_PATH` at the top of the `checkusb` script. See the [OVMF Firmware Path](#ovmf-firmware-path) section for per-distro paths.

</details>

<details>
<summary><strong>Script aborts unexpectedly</strong></summary>

Boot-Check uses `set -euo pipefail` for safety — any unexpected error or unset variable will cause the script to exit immediately. If this happens:

1. Run in debug mode to see what went wrong:
   ```bash
   bash -x checkusb
   ```
2. Check that all dependencies are installed and the device is connected.
3. Open an issue on [GitHub](https://github.com/NunesFilipe/Boot-Check/issues) with the debug output.

</details>

---

## Project Structure

```
Boot-Check/
├── checkusb    # Main executable script (single-file)
└── README.md   # Documentation
```

### Script Internals

The `checkusb` script is organized into clearly separated sections:

| Section | Purpose |
|---|---|
| `ANSI COLOR CODES` | Terminal color constants |
| `CONFIGURATION` | User-editable settings (OVMF path, RAM, etc.) |
| `TERMINAL HELPERS` | `clear_screen`, `restore_cursor`, exit trap |
| `TUI DRAWING` | `draw_header`, `draw_section`, `draw_footer` |
| `LOG HELPERS` | `msg_info`, `msg_ok`, `msg_warn`, `msg_error` |
| `USB DETECTION` | `detect_usb_drives`, `get_usb_model`, `sanitize_model_name` |
| `SESSION STATE` | Global variables shared between menus |
| `MENUS` | `menu_select_device`, `menu_select_mode` |
| `QEMU LAUNCH` | `validate_launch_params`, `build_qemu_args`, `launch_qemu` |
| `STARTUP` | `check_deps`, `main` |

---

## Contributing

Boot-Check is **open source** and contributions are welcome!

Whether it's a bug fix, a new feature, or a UI improvement — feel free to fork the repository, make your changes, and open a pull request.

**Ideas for contributions:**
- ARM / RISC-V architecture support
- Custom RAM and CPU configuration via flags
- Distribution-specific install scripts
- UI improvements and accessibility

If you fork or redistribute this project, please keep the original authorship notice in the script header:

```
Originally created by Filipe Nunes — https://github.com/NunesFilipe/Boot-Check
```

---

## License

Distributed under the **MIT License**. See [`LICENSE`](./LICENSE) for more information.
