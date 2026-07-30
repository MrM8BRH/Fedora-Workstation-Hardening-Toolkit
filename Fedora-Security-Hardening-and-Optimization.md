# Fedora Security Hardening and Optimization

> **Operational warning:** Review each section before applying it. Several commands are hardware-, workload-, or environment-specific and may disable services or features required by virtualization, storage, identity management, printing, discovery, wireless networking, or enterprise authentication.

```bash
### The operational recommendation is CIS Workstation L1 plus a tailored set of selected Level 2 controls, rather than applying Level 2 wholesale.
# Run the Fedora Workstation Level 1 assessment
sudo oscap xccdf eval \
  --profile cis_workstation_l1 \
  --results-arf fedora44-cis-workstation-l1-arf.xml \
  --report fedora44-cis-workstation-l1-report.html \
  /usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml

# Run the Fedora Workstation Level 2 assessment
sudo oscap xccdf eval \
  --profile cis_workstation_l2 \
  --results-arf fedora44-cis-workstation-l2-arf.xml \
  --report fedora44-cis-workstation-l2-report.html \
  /usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml

# Open the report
xdg-open fedora44-cis-workstation-l1-report.html
xdg-open fedora44-cis-workstation-l2-report.html
```

## 1. Recommended security checklist

- [ ] Keep SELinux enabled (enforcing)
- [ ] Keep Firewalld enabled
- [ ] Enable automatic updates
- [ ] Use Secure Boot
- [ ] Use Full Disk Encryption (LUKS)
- [ ] Use strong passwords
- [ ] Install firmware updates
- [ ] Audit the system with Lynis
- [ ] Review open ports
- [ ] Harden file/directory permissions on sensitive config files
- [ ] Harden SSH configuration
- [ ] Enable sudo command logging
- [ ] Back up important data

## 2. Baseline security verification

### Check SELinux status

```bash
getenforce
```

### Check Firewalld

```bash
sudo systemctl status firewalld
sudo firewall-cmd --list-all
```

### Review open ports

```bash
ss -tulpn
```

### Review enabled services

```bash
systemctl list-unit-files --state=enabled
```

### Review installed packages

```bash
# Lists installed packages with install date, most recent first.
rpm -qa --last
```

## 3. Firewalld hardening

### Remove the broad high-port ranges from the Fedora Workstation zone

```bash
sudo firewall-cmd --permanent \
  --zone=FedoraWorkstation \
  --remove-port=1025-65535/tcp \
  --remove-port=1025-65535/udp

sudo firewall-cmd --reload
```

### Log denied traffic

```bash
sudo firewall-cmd --set-log-denied=all
sudo firewall-cmd --reload
```

## 4. Disable unused services

> Disable only services that are not required by the system. The following list includes virtualization guest agents, storage monitoring, iSCSI, identity services, live-environment services, and crash-reporting components.

```bash
sudo systemctl disable \
  vboxservice.service \
  vgauthd.service \
  vmtoolsd.service \
  qemu-guest-agent.service \
  virtqemud.service \
  lvm2-monitor.service \
  mdmonitor.service \
  iscsi-onboot.service \
  iscsi-starter.service \
  sssd.service \
  livesys.service \
  livesys-late.service \
  abrt-vmcore.service
```

### Disable Avahi and CUPS immediately

```bash
sudo systemctl disable --now avahi-daemon cups
```

## 5. File and configuration permission hardening

Restrict ownership and permissions on cron directories and sensitive account/authentication files so only `root` can read or modify them. This is a standard CIS-benchmark-style hardening step.

```bash
# Cron directories: root ownership only
sudo chown root:root /etc/crontab /etc/cron.hourly /etc/cron.daily \
  /etc/cron.weekly /etc/cron.monthly /etc/cron.d
sudo chmod og-rwx /etc/crontab /etc/cron.hourly /etc/cron.daily \
  /etc/cron.weekly /etc/cron.monthly /etc/cron.d

# Backup copies of account/auth files: root ownership, no group/other access
sudo chown root:root /etc/passwd- /etc/group-
sudo chown root:root /etc/shadow-
sudo chmod u-x,go-rwx /etc/passwd- /etc/shadow- /etc/group-

# SSH daemon config: root only
sudo chown root:root /etc/ssh/sshd_config
sudo chmod og-rwx /etc/ssh/sshd_config
```

### Remove the pre-login MOTD (Cockpit message)

```bash
sudo ln -sfn /dev/null /etc/motd.d/cockpit
```

## 6. Legal notice / login banner

Set a warning banner shown before local and remote (SSH) logins.

```bash
echo "Authorized uses only. All activity may be monitored and reported." | sudo tee /etc/issue
echo "Authorized uses only. All activity may be monitored and reported." | sudo tee /etc/issue.net
```

## 7. Kernel module blacklisting (uncommon filesystems and USB storage)

> **Warning:** These modules are blacklisted by disabling their ability to load (`install <module> /bin/true`), then unloaded if currently active. This is a common CIS/DISA hardening control to reduce attack surface from rarely-used filesystem parsers and removable storage, but it has real functional impact — read each item before applying.

```bash
# USB mass-storage driver.
# WARNING: this disables ALL USB flash drives, USB external hard disks, and
# USB card readers that expose a block device. Keyboards/mice (HID) are unaffected.
echo "install usb-storage /bin/true" | sudo tee /etc/modprobe.d/usb_storage.conf
sudo rmmod usb-storage 2>/dev/null || true

# Legacy Veritas filesystem — rarely used, low functional risk.
echo "install freevxfs /bin/true" | sudo tee /etc/modprobe.d/freevxfs.conf
sudo rmmod freevxfs 2>/dev/null || true

# JFFS2 (flash filesystem, embedded devices) — rarely used on a workstation.
echo "install jffs2 /bin/true" | sudo tee /etc/modprobe.d/jffs2.conf
sudo rmmod jffs2 2>/dev/null || true

# Apple HFS — WARNING: disabling this prevents mounting classic Mac HFS-formatted media.
echo "install hfs /bin/true" | sudo tee /etc/modprobe.d/hfs.conf
sudo rmmod hfs 2>/dev/null || true

# Apple HFS+ — WARNING: disabling this prevents mounting modern Mac-formatted
# external drives and Time Machine disks.
echo "install hfsplus /bin/true" | sudo tee /etc/modprobe.d/hfsplus.conf
sudo rmmod hfsplus 2>/dev/null || true

# SquashFS — WARNING: this is used by live/installer ISOs, Snap packages, and
# some AppImage/Flatpak runtime layers. Disabling it can break Snap (if installed)
# and mounting of squashfs-based images.
echo "install squashfs /bin/true" | sudo tee /etc/modprobe.d/squashfs.conf
sudo rmmod squashfs 2>/dev/null || true

# UDF — WARNING: this prevents reading UDF-formatted optical media (most
# burned/pressed DVDs) and some USB sticks formatted as UDF.
echo "install udf /bin/true" | sudo tee /etc/modprobe.d/udf.conf
sudo rmmod udf 2>/dev/null || true
```

## 8. World-writable directory remediation (sticky bit)

Set the sticky bit on world-writable directories that are missing it, preventing users from deleting or renaming files they don't own inside shared directories (the same protection `/tmp` already has).

```bash
# Note: this walks every local filesystem and can take a long time on large disks.
sudo df --local -P | awk '{if (NR!=1) print $6}' | \
  xargs -I '{}' find '{}' -xdev -type d \( -perm -0002 -a ! -perm -1000 \) 2>/dev/null | \
  xargs -I '{}' sudo chmod a+t '{}'
```

## 9. SSH hardening

Edit `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Modify these existing directives:

```
MaxAuthTries 4
PermitRootLogin no
ClientAliveInterval 300
ClientAliveCountMax 0
LoginGraceTime 60
```

Add this directive to restrict MACs to modern, encrypt-then-MAC algorithms:

```
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
```

Apply the change:

```bash
sudo systemctl restart sshd
```

## 10. sudo command logging

Log every sudo invocation to a dedicated file for auditing.

```bash
sudo visudo
```

Add:

```
Defaults logfile="/var/log/sudo.log"
```

## 11. Persistent sysctl hardening and performance tuning

Every `sysctl` setting in this toolkit (kernel/exploit hardening, filesystem protections, network hardening, network privacy, network/TCP performance tuning, and VM/memory tuning) lives in **one** persistent drop-in file instead of being applied ad hoc with `sysctl -w` (which only affects the running kernel and is lost on reboot).

> **Note:** some entries below serve both security and performance goals (e.g. `rp_filter`, `tcp_syncookies`). Where a security-relevant default and a performance-tuning default could conflict, the security-relevant value is used.

```bash
sudo tee /etc/sysctl.d/99-workstation-hardening.conf <<'EOF'
# =============================================================================
# Persistent sysctl configuration — security hardening + performance tuning
# Applied at boot by systemd-sysctl, and immediately via `sysctl --system`.
# =============================================================================

# --- Kernel information-leak and exploitation hardening ---
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
kernel.kexec_load_disabled = 1

# --- Filesystem protections ---
fs.protected_fifos = 2
fs.protected_regular = 2
# Raise max open files system-wide (useful for high-fd-count workloads).
fs.file-max = 1048576

# --- Network hardening ---
# Disable IPv4 forwarding. Only set to 1 on hosts acting as a router/NAT gateway.
net.ipv4.ip_forward = 0
# Enable strict reverse-path filtering (anti IP-spoofing).
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
# Reject ICMP redirects (anti route-injection).
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
# Reject source-routed packets.
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
# Log packets with impossible/spoofed source addresses ("martians").
net.ipv4.conf.all.log_martians = 1
# Enable SYN cookies to mitigate SYN flood attacks.
net.ipv4.tcp_syncookies = 1

# --- Network privacy tuning ---
# Disable TCP timestamps to reduce information leakage (uptime fingerprinting).
net.ipv4.tcp_timestamps = 0
# Disable TCP metrics caching to avoid leaking previous connection RTT info.
net.ipv4.tcp_no_metrics_save = 1

# --- Network/TCP performance tuning ---
# Reuse TIME-WAIT sockets for new connections (use with caution behind NAT).
net.ipv4.tcp_tw_reuse = 1
# Reduce FIN_WAIT timeout for faster socket cleanup.
net.ipv4.tcp_fin_timeout = 15
# Keepalive time for idle connections.
net.ipv4.tcp_keepalive_time = 600
# Socket buffer sizes, tuned for high-throughput workloads.
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.rmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_default = 262144
net.core.wmem_max = 16777216
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_sack = 1
# Backlog/queue limits for performance under load.
net.core.netdev_max_backlog = 10000
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
# Ephemeral port range.
net.ipv4.ip_local_port_range = 1024 65535
# Reduce SYN retries for faster failure detection.
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_synack_retries = 3
# Orphan socket limit (raise cautiously, depending on available memory).
net.ipv4.tcp_max_orphans = 32768

# --- VM/memory tuning ---
# Zram-aware tuning
vm.swappiness = 100
vm.page-cluster = 0
# Retain dentry and inode caches longer.
vm.vfs_cache_pressure = 100
# RAM-relative writeback thresholds
vm.dirty_background_ratio = 10
vm.dirty_ratio = 20
# Raise virtual memory map limits (useful for large applications, e.g. Elasticsearch).
vm.max_map_count = 1048576
EOF

sudo sysctl --system
```

## 12. Kernel boot parameter hardening

Save the following as `kernel-boot-hardening.sh`. This only covers boot-time kernel parameters (via `grubby`); all `sysctl` settings live in the single persistent file from Section 11 above.

```bash
#!/usr/bin/env bash

# Review before running.
# Run with: sudo bash kernel-boot-hardening.sh
# A reboot is required for these changes to take effect.

set -euo pipefail

grubby --update-kernel=ALL --args="intel_iommu=on iommu=pt page_alloc.shuffle=1"

# intel_iommu=on iommu=pt:
#   Turns on DMA remapping and isolation in passthrough mode. This helps protect
#   against malicious DMA over USB-C, Thunderbolt, or PCIe while preserving
#   native performance for a non-virtualization laptop.
#   NOTE: intel_iommu is Intel-CPU-specific. On AMD systems use amd_iommu=on instead.
#
# page_alloc.shuffle=1:
#   Randomizes the page allocator freelist. This is a KSPP hardening measure
#   that makes heap-spray and heap-grooming exploitation more difficult, with
#   negligible expected cost.
#
# NOT added and NOT recommended:
#   mitigations=off or disabling Spectre-class mitigations. Doing so would
#   create a security regression.
#
# Optional hardening with real tradeoffs:
#
# init_on_free=1
#   Zeroes freed memory and closes a use-after-free information-leak window.
#   Expected cost: approximately 1-3% on allocation-heavy workloads.
#
# debugfs=off
#   Reduces attack surface but breaks powertop and some driver diagnostic tools.
#
# nowatchdog
#   Provides a small power or performance gain but removes hard-lockup detection.
#
# grubby --update-kernel=ALL --args="init_on_free=1 nowatchdog"
```

Run the script:

```bash
sudo bash kernel-boot-hardening.sh
```

## 13. Disable SMT / Hyper-Threading (optional)

Disables simultaneous multithreading to mitigate cross-thread CPU side-channel attacks (e.g. MDS/L1TF-class issues) at the cost of CPU throughput.

> **Warning — performance impact:** this can measurably reduce multi-threaded workload performance (often cited in the 10-30% range depending on workload). Only apply on threat models where cross-tenant/cross-process side-channel isolation matters (e.g. running untrusted code, VMs, or browser sandboxes on shared cores).
>
> **Note — hardware-specific:** the sysfs path and terminology apply to Intel Hyper-Threading and AMD SMT alike, but availability depends on the specific CPU model.

```bash
# Check current state
cat /sys/devices/system/cpu/smt/active

# Disable (does not persist across reboot; add to a boot script/unit for persistence)
echo off | sudo tee /sys/devices/system/cpu/smt/control
```

## 14. Automatic security updates

```bash
sudo install -d -m 0755 /etc/dnf

sudo tee /etc/dnf/automatic.conf >/dev/null <<'EOF'
[commands]
apply_updates = yes
upgrade_type = default
reboot = never
EOF
```
```bash
sudo systemctl enable --now dnf5-automatic.timer
```

## 15. SELinux enforcing mode

```bash
# Set enforcing for the current session
sudo setenforce 1
```

Make it permanent by editing `/etc/selinux/config`:

```
SELINUX=enforcing
```

## 16. Fail2ban operational commands

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip <IP>
```

- `sudo fail2ban-client status` shows the active jails.
- `sudo fail2ban-client status sshd` shows the status of the SSH jail and currently banned addresses.
- `sudo fail2ban-client set sshd unbanip <IP>` removes an address from the SSH jail.
- With SSH disabled, the `sshd` jail may continue to show zero banned addresses. This is expected.

## 17. ClamAV scan

Scan the Downloads directory recursively and display only infected files:

```bash
clamscan -r -i ~/Downloads
```

## 18. Disable the Wi-Fi Direct P2P pseudo-device

### Scope

This configuration is intended for a system using the `mt7921e` driver where NetworkManager automatically creates the `p2p-dev-wlo1` pseudo-device.

NetworkManager starts `wpa_supplicant` and creates the P2P pseudo-device when the driver reports P2P capability. Editing `/etc/wpa_supplicant/wpa_supplicant.conf` does not affect NetworkManager-managed active connections.

> **Note — hardware/vendor-specific:** this section applies to the MediaTek `mt7921e` Wi-Fi chip/driver specifically. Interface names (`wlo1`, `p2p-dev-wlo1`) and the P2P-capability quirk may differ on other Wi-Fi chipsets (Intel `iwlwifi`, Realtek, Broadcom, etc.).

### Configuration

```bash
sudo mkdir -p /etc/NetworkManager/conf.d

sudo tee /etc/NetworkManager/conf.d/99-disable-p2p.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:p2p-dev-wlo1
EOF

sudo systemctl restart NetworkManager
```

### Verification

```bash
nmcli device status
```

The `p2p-dev-wlo1` interface should appear as `unmanaged` instead of `disconnected`.

### Driver note

The `mt7921e` driver may intermittently fail while initializing the P2P device during startup. This is generally cosmetic when the primary Wi-Fi interface, such as `wlo1`, remains connected and functional.

The NetworkManager configuration suppresses management of the P2P pseudo-device but does not fix the underlying driver or firmware behavior. Leaving it unmanaged is appropriate when Wi-Fi Direct or P2P-based screen casting is not required.

## 19. Disable all radios (optional, high impact)

```bash
nmcli radio all off
```

> **WARNING — this disables Wi-Fi, Bluetooth, and WWAN system-wide.** Unlike section 19 (which only unmanages the P2P pseudo-interface), this command turns off the actual Wi-Fi radio and all other wireless radios via NetworkManager, immediately dropping any wireless network connection. Only use this for air-gapped/kiosk configurations or when wired Ethernet is the sole intended network path. To re-enable: `nmcli radio all on`.
