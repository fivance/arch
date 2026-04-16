# Linux Performance Tweaks

## 1. Install CachyOS Repository

```bash
curl https://mirror.cachyos.org/cachyos-repo.tar.xz -o cachyos-repo.tar.xz
tar xvzf cachyos-repo.tar.xz && cd cachyos-repo
sudo ./cachyos-repo.sh
```

---

## 2. Kernel Parameters

Create `/etc/sysctl.d/99-cachyos.conf`:

```ini
# Memory
vm.swappiness = 10
vm.vfs_cache_pressure = 50
vm.dirty_bytes = 268435456
vm.dirty_background_bytes = 67108864
vm.dirty_writeback_centisecs = 1500
vm.page-cluster = 0

# Stability
kernel.nmi_watchdog = 0
kernel.unprivileged_userns_clone = 1

# Network
net.core.netdev_max_backlog = 16384

# File handles
fs.file-max = 2097152
```

Then apply:

```bash
sudo sysctl --system
```

---

## 3. I/O Schedulers

Create `/etc/udev/rules.d/60-ioschedulers.rules`:

```udev
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"
```

---

## 4. ZRAM (Compressed Swap)

```bash
sudo pacman -S zram-generator
```

Create `/etc/systemd/zram-generator.conf`:

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
```

---

## 5. CPU Frequency Scaling (AMD)

```bash
# Check current driver
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
```

Add to `GRUB_CMDLINE_LINUX` in `/etc/default/grub`:

```
amd_pstate=active
```

Set performance EPP:

```bash
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference
```

---

## 6. sched-ext Scheduler (SCX)

```bash
sudo pacman -S scx-scheds   # from CachyOS repos or AUR
sudo systemctl enable --now scx
```

> For gaming, `scx_lavd` or `scx_rusty` are the recommended options.  
> Edit `/etc/default/scx` to set your preferred scheduler.

---

## 7. Audio Power Save

Create `/etc/modprobe.d/audio-powersave.conf`:

```
options snd-hda-intel power_save=0
```

---

## 8. SATA Link Power Management

Create `/etc/udev/rules.d/60-sata-power.rules`:

```udev
ACTION=="add", SUBSYSTEM=="scsi_host", KERNEL=="host*", \
ATTR{link_power_management_policy}="max_performance"
```

---

## 9. NVIDIA Parameters

Create `/etc/modprobe.d/nvidia.conf`:

```
options nvidia NVreg_UsePageAttributeTable=1
options nvidia NVreg_InitializeSystemMemoryAllocations=0
```

> `NVreg_UsePageAttributeTable=1` enables CPU PAT for better performance.  
> `NVreg_InitializeSystemMemoryAllocations=0` skips zeroing system memory on allocation.
