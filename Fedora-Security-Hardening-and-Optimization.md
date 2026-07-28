# Fedora Security Hardening and Optimization

> **Operational warning:** Review each section before applying it. Several commands are hardware-, workload-, or environment-specific and may disable services or features required by virtualization, storage, identity management, printing, discovery, or enterprise authentication.

## 1. Recommended security checklist

- [ ] Keep SELinux enabled
- [ ] Keep Firewalld enabled
- [ ] Enable automatic updates
- [ ] Use Secure Boot
- [ ] Use Full Disk Encryption (LUKS)
- [ ] Use strong passwords
- [ ] Install firmware updates
- [ ] Audit the system with Lynis
- [ ] Review open ports
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

## 5. Log suspicious IPv4 packets

Open the system-wide sysctl configuration:

```bash
sudo nano /etc/sysctl.conf
```

Add:

```ini
net.ipv4.conf.all.log_martians = 1
```

## 6. Kernel hardening and performance-tuning script

Save the following as `kernel-tuning-hardening.sh`.

```bash
#!/usr/bin/env bash

# Review every section before running.
# Run with: sudo bash kernel-tuning-hardening.sh
# A reboot is required for the boot-parameter changes to take effect.

set -euo pipefail

echo "== 1/4 Kernel boot parameters (grubby - BLS safe, no grub.cfg surgery needed) =="

grubby --update-kernel=ALL --args="intel_iommu=on iommu=pt page_alloc.shuffle=1"

# intel_iommu=on iommu=pt:
#   Turns on DMA remapping and isolation in passthrough mode. This helps protect
#   against malicious DMA over USB-C, Thunderbolt, or PCIe while preserving
#   native performance for a non-virtualization laptop.
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

echo "== 2/4 Sysctl security hardening =="

cat > /etc/sysctl.d/99-security-hardening.conf <<'EOF'
# Kernel information-leak and exploitation hardening
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
kernel.kexec_load_disabled = 1

# Filesystem protections
fs.protected_fifos = 2
fs.protected_regular = 2

# Network hardening
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
EOF

echo "== 3/4 Performance tuning for zram + NVMe + 24 GB RAM =="

cat > /etc/sysctl.d/98-perf-tuning.conf <<'EOF'
# Use zram more aggressively than the default swappiness setting.
vm.swappiness = 100

# Use single-page reads instead of clustered swap readahead.
vm.page-cluster = 0

# Retain dentry and inode caches longer.
vm.vfs_cache_pressure = 50

# Use fixed writeback thresholds for more predictable I/O latency.
vm.dirty_bytes = 268435456
vm.dirty_background_bytes = 134217728
EOF

sysctl --system

echo "== 4/4 Power management daemon =="

# Use power-profiles-daemon with GNOME/Fedora Workstation instead of TLP.
dnf install -y power-profiles-daemon
systemctl enable --now power-profiles-daemon

echo
echo "Done. Reboot for the kernel boot-parameter changes to take effect."
echo "Verify after reboot with:"
echo "  cat /proc/cmdline"
echo "  sysctl kernel.kptr_restrict kernel.yama.ptrace_scope vm.swappiness"
echo "  powerprofilesctl list"
```

Run the script:

```bash
sudo bash kernel-tuning-hardening.sh
```

## 7. Fail2ban operational commands

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip <IP>
```

- `sudo fail2ban-client status` shows the active jails.
- `sudo fail2ban-client status sshd` shows the status of the SSH jail and currently banned addresses.
- `sudo fail2ban-client set sshd unbanip <IP>` removes an address from the SSH jail.
- With SSH disabled, the `sshd` jail may continue to show zero banned addresses. This is expected.

## 8. ClamAV scan

Scan the Downloads directory recursively and display only infected files:

```bash
clamscan -r -i ~/Downloads
```

## 9. Disable the Wi-Fi Direct P2P pseudo-device

### Scope

This configuration is intended for a system using the `mt7921e` driver where NetworkManager automatically creates the `p2p-dev-wlo1` pseudo-device.

NetworkManager starts `wpa_supplicant` and creates the P2P pseudo-device when the driver reports P2P capability. Editing `/etc/wpa_supplicant/wpa_supplicant.conf` does not affect NetworkManager-managed active connections.

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
