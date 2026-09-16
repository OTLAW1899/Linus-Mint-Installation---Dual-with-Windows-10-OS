Linux Mint on an External HDD - Dual-Boot with Windows 10


Overview:

This repo documents installing Linux Mint 22.3 onto an external USB HDD (Toshiba, USB 3.0), while keeping the existing Windows 10 installation intact on the internal drive (Samsung NVMe/SSD) of a Lenovo ThinkPad.

The goal: run Mint entirely off the external drive, dual-boot with Windows, and be able to unplug the external drive and boot straight into Windows when Mint isn't needed.

Setup:

- Laptop: Lenovo ThinkPad T460s
- Internal drive: 256GB, Windows 10
- External drive: Toshiba External HDD, USB 3.0 - Linux Mint 22.3 installed here
- Firmware mode: Legacy/CSM enabled (`UEFI/Legacy Boot Priority: Legacy First`, `CSM Support: Yes`)

Initial Installation Process:

1. Created a bootable Mint USB using [balenaEtcher](https://etcher.) on another PC, writing the Linux Mint 22.3 ISO to a spare USB flash drive.
2. Connected the target Toshiba External HDD to the ThinkPad T460s via USB 3.0.
3. Booted from the Mint installer USB by tapping `F12` at startup and selecting the USB drive from the boot menu.
4. Selected "Try or Install Linux Mint" from the boot menu (or "Start Linux Mint"), launching the live environment.
5. Opened the installer from the desktop and proceeded through the initial setup (language, keyboard layout, updates/third-party software prompt).
6. At the installation type screen, chose "Something else" (manual partitioning) rather than the default "Install alongside" or "Erase disk" options - this was necessary to explicitly target the external Toshiba drive instead of the internal Samsung SSD.
7. Selected the Toshiba external drive (appearing as `/dev/sdb`) as the install target, created the necessary partitions on it (root `/`, swap, and an EFI system partition if using UEFI/GPT), and set the "Device for boot loader installation" dropdown - this is the step that likely caused the later problems, since it's easy to leave this set to the internal drive (`/dev/sda`, the Samsung SSD) by default instead of pointing it to the external drive itself.
8. Completed the installation, removed the USB installer, and rebooted.

This is also where the root of Issues #2 and #3 traces back to: if the "boot loader installation" device wasn't explicitly changed to the external Toshiba drive during this step, GRUB installs onto the internal SSD by default setting up the exact dependency problem described below.
                                                                      
Issues encountered:

Issue #1: System defaulted to booting Mint instead of Windows

After installation, the laptop booted directly into Mint with no menu, even though BIOS boot order listed Windows Boot Manager first.

Diagnosis: Legacy/CSM boot mode was overriding the UEFI boot order list - the firmware appeared to boot the external HDD via a legacy path regardless of the listed UEFI priority.

Workaround used: Use `F12` at startup for a one-time boot menu to explicitly choose Windows or the Toshiba drive, rather than relying on default boot order.

Issue #2: `grub rescue>` when the external drive was unplugged

Removing the Toshiba drive (to boot straight into Windows without a menu) resulted in:


error: unknown filesystem.
Entering rescue mode...
grub rescue>


Root cause: During Mint's install, GRUB had been installed to the internal Samsung drive, not the external one. With the external drive absent, GRUB (still triggered by the internal drive's boot sector) couldn't find its root filesystem which lived only on the now-unplugged Toshiba drive.

Fix attempt: Boot-Repair:

Booted into Mint (external drive plugged in) and used the [Boot-Repair](https://) tool:

bash
sudo add-apt-repository ppa:yannubuntu/boot-repair
sudo apt update
sudo apt install -y boot-repair
boot-repair


In Advanced options → GRUB location:
- OS to boot by default: `sdb3` (Linux Mint, on the Toshiba drive)
- Place GRUB into: `/dev/sdb` (the Toshiba drive itself, not the internal disk)

This successfully reinstalled GRUB onto the external drive's own boot sector, separating it from the internal disk.

Issue #3: `grub rescue>` persisted on the internal drive

Even after Boot-Repair, selecting the internal Samsung drive directly from the F12 boot menu (or letting the system default-boot without the Toshiba plugged in) still dropped to `grub rescue>`.

Diagnosis:
The internal SSD's own boot sector (MBR) still contained the original GRUB stage-1 code from the initial Mint install. Boot-Repair fixed GRUB on the external drive, but never touched and couldn't safely touch he leftover bootloader remnant on the internal disk.

Resolution in progress: Rebuilding the Windows boot sector

To fully separate the two drives so the internal disk boots Windows independently:

1. Create a Windows 10 installation USB via the Microsoft Media Creation Tool.
2. Boot from it → Repair your computer → Troubleshoot → Advanced options → Command Prompt.
3. Run:
   ```
   bootrec /fixmbr
   bootrec /fixboot
   bootrec /scanos
   bootrec /rebuildbcd
   ```

This overwrites the internal drive's boot sector with a clean Windows-only MBR, removing the leftover GRUB code, while leaving the Toshiba drive's independent (and already working) GRUB installation untouched.

Windows 11 Upgrade - Considered, but Ruled Out

While troubleshooting, briefly considered upgrading the Windows side from 10 to 11. Ruled out after checking hardware compatibility:

- CPU:ThinkPad T460s ships with 6th-generation ("Skylake-U") Intel chips - i5-6200U, i5-6300U, or i7-6600U. Microsoft's official Windows 11 supported CPU list starts at 8th-generation Intel processors, so this CPU falls outside supported hardware regardless of other settings.
- TPM: system currently reports TPM 1.2. Whether this can be switched to 2.0 depends on the specific physical TPM chip and isn't something confirmed for this exact model but it's moot anyway, since the CPU generation is a separate, independent blocker.
- Decision: stayed on Windows 10 rather than pursue an unsupported-hardware install workaround, to keep the dual-boot setup on fully supported footing.

Current Status

Scenario & Result
Toshiba plugged in, boot - Dual-boot menu (Windows + Mint) works 
Toshiba unplugged, boot - `grub rescue>` -internal drive still has stale GRUB (fix pending) 



Lessons Learned:

- Always check where GRUB installs during setup. Most installers default to installing the bootloader on the "first" or system disk rather than the disk actually hosting the OS this is exactly what caused the internal/external split issue here.
- Legacy/CSM BIOS modes can override UEFI boot order settings, making the listed boot priority unreliable as the sole method of control. A one-time boot menu (F12) is more predictable in mixed Legacy/UEFI setups.
- Boot-Repair fixes the target OS's bootloader, not other drives' leftover boot code. If an install went wrong on one disk, expect to also need OS-native tools (like Windows' `bootrec`) to clean up the other disk.
- A full power-off (not just restart) can sometimes clear stale UEFI boot-entry caching - this briefly made Windows boot manager respond again, which was a useful (if temporary and slightly misleading) diagnostic signal.

Tools Used: 

- [Linux Mint 22.3](https://linuxmint.com/)
- [Boot-Repair](https://) (via `yannubuntu/boot-repair` PPA)
- Windows 10 Media Creation Tool / recovery environment (`bootrec`)


Author: 

Olatunji Lawal

Cybersecurity and GRC Analyst

Published: September 16, 2026

