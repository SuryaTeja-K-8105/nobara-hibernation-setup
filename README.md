# Nobara Hibernate Setup

Sleep-then-hibernate configuration for Dell Tiger Lake laptops running Nobara Linux with KDE Plasma 6.

## Problem

- System enters s2idle sleep but cannot wake up (requires hard reset)
- Hibernate not triggering on lid close or idle timeout
- PowerDevil defaults to plain suspend instead of suspend-then-hibernate

## Files

### `sleep.conf` → `/etc/systemd/sleep.conf`

```ini
[Sleep]
HibernateDelaySec=10min
HibernateMode=shutdown
```

- `HibernateDelaySec=10min`: After entering sleep, auto-hibernate after 10 minutes
- `HibernateMode=shutdown`: Most reliable hibernate mode for Dell hardware

### `powerdevilrc` → `~/.config/powerdevilrc`

```ini
[AC][Performance]
PowerProfile=performance

[AC][SuspendAndShutdown]
LidAction=1
SleepMode=3
...
```

- `LidAction=1`: Trigger sleep action on lid close
- `SleepMode=3`: Use standbyThenHibernate (not hybrid=2 or plain suspend=1)

**Important:** The PowerDevil SleepMode enum values are:
| Value | Mode |
|-------|------|
| 0 | SuspendToRam (plain suspend) |
| 1 | Standby |
| 2 | HybridSuspend |
| 3 | StandbyThenHibernate |

### `zram-generator.conf` → `/etc/systemd/zram-generator.conf`

```ini
[zram0]
zram-size = min(ram, 8192)
swap-priority = 50
```

Lowers zram swap priority from 100 to 50 so disk swap is preferred for hibernation.

### GRUB kernel parameter (manual step)

Add `acpi_osi=Linux` to fix s2idle wake issues on Dell Tiger Lake:

```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="acpi_osi=Linux /' /etc/default/grub
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
sudo reboot
```

## Installation

```bash
# Copy sleep.conf
sudo cp sleep.conf /etc/systemd/sleep.conf

# Copy powerdevilrc
cp powerdevilrc ~/.config/powerdevilrc

# Copy zram config
sudo cp zram-generator.conf /etc/systemd/zram-generator.conf

# Add GRUB kernel param
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="acpi_osi=Linux /' /etc/default/grub
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg

# Reload PowerDevil
dbus-send --session --type=method_call --dest=org.kde.Solid.PowerManagement /org/kde/Solid/PowerManagement org.kde.Solid.PowerManagement.reparseConfiguration

# Reboot
sudo reboot
```

## How it works

1. **Lid close / idle timeout** → PowerDevil triggers `suspend-then-hibernate` (SleepMode=3)
2. **System suspends** (s2idle, fast wake)
3. **After 10 minutes** → systemd auto-hibernates (RAM written to disk via shutdown mode)
4. **Next boot** → Kernel resumes from hibernate image on swap partition

## Hardware

- Dell laptop, Intel Tiger Lake, Iris Xe
- Nobara Linux (Fedora-based), KDE Plasma 6
- systemd 259, PowerDevil 6.7.2
