# Fedora Workstation, Development, and Security Toolkit

> **Operational note:** Review security-testing tools and service packages before installing them on production systems. Use them only on systems and networks you are authorized to assess.

## 1. Update the system

```bash
sudo dnf upgrade --refresh
sudo reboot
```

## 2. Update Flatpak applications

```bash
flatpak update
```

## 3. Enable RPM Fusion

Free repository
```bash
sudo dnf install \
https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
```

Nonfree repository
```bash
sudo dnf install \
https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Refresh the system after enabling both repositories:
```bash
sudo dnf upgrade --refresh
```

## 4. Update firmware

```bash
sudo fwupdmgr refresh
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

## 5. Optimize DNF

Open the DNF configuration file:
```bash
sudo nano /etc/dnf/dnf.conf
```

Add the following settings:
```bash
fastestmirror=True
max_parallel_downloads=10
defaultyes=True
keepcache=True
```

## 6. Core command-line, administration, and diagnostic tools

```bash
sudo dnf install -y \
  git curl wget rsync openssh-clients \
  jq yq tree ripgrep fd-find fzf \
  vim-enhanced nano tmux bash-completion \
  unzip zip 7zip zstd xz \
  htop btop fastfetch \
  lsof strace file which man-pages \
  pciutils usbutils ethtool bind-utils \
  lm_sensors smartmontools nvme-cli \
  iotop ncdu fdupes \
  flatpak fwupd \
  python3 python3-pip
```

## 7. Network analysis, security assessment, and host-security tools

```bash
sudo dnf install -y \
  wireshark wireshark-cli \
  nmap nmap-ncat tcpdump socat \
  traceroute mtr whois \
  etherape ettercap \
  yara hashcat medusa \
  clamav clamav-update \
  openscap-scanner scap-security-guide \
  audit aide policycoreutils-python-utils setools-console \
  fail2ban
```

## 8. Compatibility and supporting libraries

```bash
sudo dnf install -y \
  libxcrypt-compat \
  libxc libxcb libxchange libxcvt libxcrypt
```

These names are preserved exactly as supplied. Some may not exist in every Fedora release or enabled repository. Validate unavailable packages with:

```bash
dnf search '<package-name>'
dnf info '<package-name>'
```

## 9. Fonts and multilingual coverage

```bash
sudo dnf install -y \
  google-noto-fonts-all-vf \
  google-noto-color-emoji-fonts \
  google-noto-sans-cjk-vf-fonts \
  google-noto-serif-cjk-vf-fonts \
  liberation-fonts-all \
  fira-code-fonts \
  jetbrains-mono-fonts-all
```

Rebuild the font cache after installation:

```bash
fc-cache -fv
```

## 10. Development toolchain

Install the explicitly requested language runtimes and build tools:

```bash
sudo dnf install -y \
  gcc cmake clang \
  cargo nodejs
```

## 11. System maintenance, desktop utilities, remote access, and services

```bash
sudo dnf install -y \
  dnf-automatic tuned \
  bleachbit baobab \
  caddy remmina \
  cockpit cockpit-system \
  gnome-tweaks
```

### Optional service activation

Enable only the services required for the target system:

```bash
# Cockpit web administration interface
sudo systemctl enable --now cockpit.socket

# Fail2ban intrusion-prevention service
sudo systemctl enable --now fail2ban

# Caddy web server
sudo systemctl enable --now caddy
```

## 13. Suggested post-install checks

```bash
# Confirm that the package database is consistent.
sudo dnf check

# Refresh ClamAV signatures, if freshclam is available and configured.
sudo freshclam

# Detect hardware-monitoring sensors interactively.
sudo sensors-detect
```
