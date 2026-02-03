# Boot Sequence & Runlevels

## Linux Boot Process Overview

```
┌─────────────────────────────────────────────────────────────┐
│  1. BIOS/UEFI                                               │
│     └── Power On Self Test (POST)                           │
│     └── Find bootable device                                │
├─────────────────────────────────────────────────────────────┤
│  2. Boot Loader (GRUB2)                                     │
│     └── Load kernel into memory                             │
│     └── Pass control to kernel                              │
├─────────────────────────────────────────────────────────────┤
│  3. Kernel Initialization                                   │
│     └── Decompress kernel                                   │
│     └── Initialize hardware                                 │
│     └── Mount root filesystem (read-only)                   │
│     └── Start init process (PID 1)                          │
├─────────────────────────────────────────────────────────────┤
│  4. Init System (systemd)                                   │
│     └── Mount filesystems                                   │
│     └── Start services                                      │
│     └── Reach target/runlevel                               │
├─────────────────────────────────────────────────────────────┤
│  5. Login Prompt / GUI                                      │
└─────────────────────────────────────────────────────────────┘
```

## 1. BIOS vs UEFI

| Feature | BIOS | UEFI |
|---------|------|------|
| Full Name | Basic Input/Output System | Unified Extensible Firmware Interface |
| Age | Legacy (1980s) | Modern (2000s) |
| Boot Mode | MBR | GPT |
| Max Disk Size | 2 TB | 9.4 ZB |
| Boot Speed | Slower | Faster |
| Security | Limited | Secure Boot |
| Interface | Text-based | Graphical |

### Check Boot Mode

```bash
# Check if UEFI or BIOS
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"

# Or check:
ls /sys/firmware/efi
```

## 2. Boot Loader - GRUB2

GRUB (Grand Unified Bootloader) loads the kernel into memory.

### GRUB Configuration

```bash
# Main config file (DO NOT EDIT DIRECTLY)
/boot/grub/grub.cfg      # Debian/Ubuntu
/boot/grub2/grub.cfg     # RHEL/CentOS

# Edit settings here instead:
/etc/default/grub

# Custom entries:
/etc/grub.d/

# Regenerate GRUB config after changes:
sudo update-grub          # Debian/Ubuntu
sudo grub2-mkconfig -o /boot/grub2/grub.cfg  # RHEL/CentOS
```

### Common GRUB Settings

```bash
# /etc/default/grub

GRUB_TIMEOUT=5                    # Seconds to wait
GRUB_DEFAULT=0                    # Default boot entry
GRUB_CMDLINE_LINUX=""             # Kernel parameters
GRUB_DISABLE_RECOVERY="false"     # Show recovery mode
```

### Boot into Single User Mode (Recovery)

1. Restart system
2. Press `Shift` or `Esc` at GRUB menu
3. Press `e` to edit
4. Add `single` or `init=/bin/bash` to kernel line
5. Press `Ctrl+X` or `F10` to boot

## 3. Kernel Initialization

```bash
# Kernel location
/boot/vmlinuz-<version>

# Initial RAM disk (drivers for boot)
/boot/initrd.img-<version>    # Debian/Ubuntu
/boot/initramfs-<version>.img # RHEL/CentOS

# View kernel messages
dmesg

# Follow kernel messages in real-time
dmesg -w

# View boot messages
journalctl -b

# View previous boot
journalctl -b -1
```

## 4. Init Systems

### Init System Comparison

| Feature | SysVinit | Upstart | systemd |
|---------|----------|---------|---------|
| Used By | Old distros | Ubuntu (old) | Modern distros |
| Config | /etc/init.d/ | /etc/init/ | /etc/systemd/ |
| Parallel | No | Yes | Yes |
| PID 1 | init | init | systemd |

### Check Init System

```bash
# Check PID 1
ps -p 1

# Or
stat /sbin/init
```

## 5. Runlevels (SysVinit) vs Targets (systemd)

### Runlevel to Target Mapping

| Runlevel | Target | Description |
|----------|--------|-------------|
| 0 | poweroff.target | Halt/Shutdown |
| 1 | rescue.target | Single user mode |
| 2 | multi-user.target | Multi-user (no network) |
| 3 | multi-user.target | Multi-user with network (CLI) |
| 4 | multi-user.target | Custom/Unused |
| 5 | graphical.target | Multi-user with GUI |
| 6 | reboot.target | Reboot |

### Managing Targets (systemd)

```bash
# Check current target
systemctl get-default

# List all targets
systemctl list-units --type=target

# Change default target
sudo systemctl set-default multi-user.target   # CLI only
sudo systemctl set-default graphical.target    # With GUI

# Switch target immediately
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target

# Emergency mode (minimal)
sudo systemctl isolate emergency.target

# Rescue mode (single user)
sudo systemctl isolate rescue.target
```

### Managing Runlevels (SysVinit - Legacy)

```bash
# Check current runlevel
runlevel

# Change runlevel
init 3    # CLI mode
init 5    # GUI mode

# Default runlevel config
/etc/inittab
```

## Boot Analysis

```bash
# Analyze boot time
systemd-analyze

# Blame - show slow services
systemd-analyze blame

# Critical chain
systemd-analyze critical-chain

# Plot boot sequence (creates SVG)
systemd-analyze plot > boot.svg
```

## Important Boot Files

| File/Directory | Purpose |
|----------------|---------|
| `/boot/` | Kernel and boot files |
| `/boot/grub/` | GRUB configuration |
| `/etc/default/grub` | GRUB settings |
| `/etc/fstab` | Filesystem mount table |
| `/etc/systemd/system/` | Custom systemd units |

## Common Boot Issues

| Issue | Solution |
|-------|----------|
| GRUB not found | Reinstall GRUB |
| Kernel panic | Boot older kernel, fix initramfs |
| fsck required | Run filesystem check |
| Service fails | Check `journalctl -xe` |
| Slow boot | Check `systemd-analyze blame` |

## Quick Commands

```bash
# Reboot
sudo reboot
sudo systemctl reboot

# Shutdown
sudo shutdown -h now
sudo systemctl poweroff

# Schedule shutdown
sudo shutdown -h +10     # In 10 minutes
sudo shutdown -h 22:00   # At 10 PM

# Cancel shutdown
sudo shutdown -c
```

---

**Previous:** [02-kernel-hardware.md](02-kernel-hardware.md) | **Next:** [04-file-types-hierarchy.md](04-file-types-hierarchy.md)
