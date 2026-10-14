# Headless KVM Host on Rocky Linux, Managed from a MacBook

A complete guide to turning a Lenovo ThinkCentre into a headless KVM virtualization host running Rocky Linux, and managing it and its VMs entirely from macOS.

---

## Table of Contents

1. [Overview](#overview)
2. [Placeholders Used](#placeholders-used)
3. [Phase 1: Preparation (Mac)](#phase-1-preparation-mac)
4. [Phase 2: Install Rocky Linux (ThinkCentre)](#phase-2-install-rocky-linux-thinkcentre)
5. [Phase 3: SSH Access (Mac)](#phase-3-ssh-access-mac)
6. [Phase 4: Configure the Host (over SSH)](#phase-4-configure-the-host-over-ssh)
7. [Phase 5: Management Tools (Mac)](#phase-5-management-tools-mac)
8. [Phase 6: Create Your First VM](#phase-6-create-your-first-vm)
9. [Phase 7: Daily Cheat Sheet](#phase-7-daily-cheat-sheet)
10. [Optional: Access from Anywhere (Tailscale)](#optional-access-from-anywhere-tailscale)
11. [Performance Notes](#performance-notes)
12. [Troubleshooting](#troubleshooting)

---

## Overview

| Component | Role |
|---|---|
| **ThinkCentre** | Headless host running Rocky Linux + KVM/QEMU + libvirt |
| **MacBook** | Management workstation (SSH, `virsh`, Cockpit, optional virt-manager) |
| **VMs** | Run on the ThinkCentre, accessed from the Mac via SSH |

**Ways to manage the host from the Mac:**

- **SSH + `virsh`**: full control from the terminal
- **Cockpit**: web dashboard at `https://<host-ip>:9090`, including VM consoles in the browser
- **virt-manager (optional)**: GUI app installed on the Mac via Homebrew, connects over SSH

> **Note:** virt-manager is deprecated and **not available on Rocky Linux 10 hosts**; Cockpit is the official replacement. Running virt-manager *on your Mac* still works, because it only talks to libvirt over SSH.

---

## Placeholders Used

Replace these with your own values throughout the guide:

| Item | Example |
|---|---|
| ThinkCentre IP | `192.168.1.50` |
| Router / gateway | `192.168.1.1` |
| Linux username on host | `you` |
| Username inside VM | `user` |
| First VM name | `vm1` |

---

## Phase 1: Preparation (Mac)

### 1.1 Check the ThinkCentre's CPU

Rocky Linux 10 requires an **x86-64-v3** CPU (Intel: roughly 4th-gen Core / Haswell or newer).

- If your CPU is older, use **Rocky Linux 9** instead (supported until 2032, and still includes virt-manager on the host). The rest of this guide is nearly identical.
- If a Linux system is already on the machine, you can check with:

```bash
/lib64/ld-linux-x86-64.so.2 --help | grep supported
```

If `x86-64-v3 (supported, searched)` appears, Rocky 10 will work.

### 1.2 Download the ISO

From <https://rockylinux.org/download>, download the **x86_64 Minimal ISO**.

> Make sure it's **x86_64**, not ARM/aarch64, even though you're downloading on a Mac.

### 1.3 Create a Bootable USB

**Option A: balenaEtcher (easiest)**

```bash
brew install --cask balenaetcher
```

Open Etcher → select the ISO → select the USB drive → **Flash**.

**Option B: `dd` (careful, the wrong disk number erases that disk)**

```bash
diskutil list                          # find the USB, e.g. /dev/disk4
diskutil unmountDisk /dev/disk4
sudo dd if=~/Downloads/Rocky-10*-minimal.iso of=/dev/rdisk4 bs=4m status=progress
```

---

## Phase 2: Install Rocky Linux (ThinkCentre)

> A monitor and keyboard are needed **only for this phase**.

### 2.1 BIOS Settings

1. Power on and press **F1** to enter the BIOS (most ThinkCentres).
2. Enable **Intel Virtualization Technology (VT-x)** and **VT-d**, usually under *Advanced → CPU Setup*.
3. Set the USB as a boot device, save, and reboot from the USB.

### 2.2 Installer Settings

| Section | Setting |
|---|---|
| **Software Selection** | Minimal Install |
| **Network & Hostname** | Enable Ethernet; set hostname, e.g. `thinkcentre` |
| **User Creation** | Create `you`; tick **"Make this user administrator"** (root can stay locked) |
| **Installation Destination** | See below (important) |

**Disk layout (important):** The default automatic layout gives `/` only ~70 GB and puts the rest into `/home`. VM disks live in `/var/lib/libvirt/images` (on `/`), so you'd run out of space quickly.

1. Choose **Custom**.
2. Click **"Click here to create them automatically."**
3. **Delete `/home`.**
4. Set `/` to use all remaining space.

Click **Begin Installation**, then reboot when it finishes.

### 2.3 Give the Host a Fixed IP

**Option A (recommended):** In your router, create a **DHCP reservation** for the ThinkCentre's MAC address.

**Option B:** Set a static IP on the host (do this at the console, not over SSH):

```bash
nmcli con show                         # note the connection name, e.g. "eno1"
sudo nmcli con mod eno1 ipv4.method manual \
  ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns 192.168.1.1
sudo nmcli con up eno1
ip a                                   # confirm the IP
```

You can now disconnect the monitor and keyboard.

---

## Phase 3: SSH Access (Mac)

### 3.1 Create and Copy an SSH Key

```bash
ssh-keygen -t ed25519                  # skip if you already have a key
ssh-copy-id you@192.168.1.50
```

### 3.2 Add the Host to `~/.ssh/config`

```
Host thinkcentre
    HostName 192.168.1.50
    User you
```

Test it:

```bash
ssh thinkcentre
```

---

## Phase 4: Configure the Host (over SSH)

Run all of the following after `ssh thinkcentre`.

### 4.1 Update and Install Packages

```bash
sudo dnf update -y
sudo dnf install -y qemu-kvm libvirt virt-install virt-viewer \
  cockpit cockpit-machines libguestfs-tools tmux
sudo reboot
```

### 4.2 Enable libvirt and Cockpit

Rocky uses libvirt's modular daemons:

```bash
sudo systemctl enable --now virtqemud.socket virtnetworkd.socket \
  virtstoraged.socket virtnodedevd.socket virtproxyd.socket
sudo systemctl enable --now cockpit.socket
sudo usermod -aG libvirt you
```

Log out and back in for the group change to take effect.

### 4.3 Start the Default VM Network

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

("Network is already active" is fine.)

### 4.4 Validate the Host

```bash
virt-host-validate qemu
```

All lines should show **PASS**. An IOMMU warning is fine unless you plan to use PCI passthrough.

### 4.5 Firewall

SSH and Cockpit are usually allowed by default. To be sure:

```bash
sudo firewall-cmd --add-service=cockpit --permanent
sudo firewall-cmd --reload
```

### 4.6 Harden SSH

> ⚠️ Only do this **after** confirming key-based login works.

```bash
echo -e "PasswordAuthentication no\nPermitRootLogin no" | \
  sudo tee /etc/ssh/sshd_config.d/99-hardening.conf
sudo systemctl restart sshd
```

Open a **new** terminal on the Mac and run `ssh thinkcentre` to confirm access **before** closing your current session.

---

## Phase 5: Management Tools (Mac)

### 5.1 libvirt Client (`virsh` on the Mac)

```bash
brew install libvirt
echo 'export LIBVIRT_DEFAULT_URI="qemu+ssh://thinkcentre/system"' >> ~/.zshrc
source ~/.zshrc
virsh list --all                       # runs against the ThinkCentre
```

### 5.2 virt-manager (Optional GUI)

```bash
brew install virt-manager
virt-manager -c "qemu+ssh://thinkcentre/system?socket=/var/run/libvirt/libvirt-sock" --no-fork
```

Tips:

- **SSH keys are required.** The Mac build can't display SSH password prompts properly.
- The `?socket=...` parameter and `--no-fork` avoid common macOS connection errors.
- The Mac build is community-maintained; if an update breaks it, use Cockpit.

### 5.3 Cockpit (Web Dashboard)

1. Open `https://192.168.1.50:9090` in your browser.
2. Accept the self-signed certificate warning.
3. Log in as `you` and click **"Turn on administrative access."**
4. Use the **Virtual Machines** tab to create, edit, and view VMs (with a graphical console in the browser).

---

## Phase 6: Create Your First VM

### 6.1 Download an Installer ISO on the Host

Downloading directly on the host is faster than copying from the Mac. Run this on the ThinkCentre:

```bash
cd /var/lib/libvirt/images
sudo curl -LO https://download.rockylinux.org/pub/rocky/10/isos/x86_64/Rocky-10-latest-x86_64-minimal.iso
```

### 6.2 Create the VM (Text-Mode Installer in Your Terminal)

```bash
sudo virt-install \
  --name vm1 \
  --memory 2048 --vcpus 2 \
  --cpu host-passthrough \
  --disk size=20,bus=virtio \
  --osinfo rocky10 \
  --network network=default,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --location /var/lib/libvirt/images/Rocky-10-latest-x86_64-minimal.iso \
  --extra-args 'console=ttyS0,115200n8 inst.text'
```

- If `rocky10` isn't recognized, check with `osinfo-query os | grep -i rocky`, or use `linux2022`.
- During installation, **enable networking** and **create a user**.
- **GUI alternative:** In Cockpit, click **Create VM** and install via the browser console.

### 6.3 Autostart the VM with the Host

```bash
virsh autostart vm1
```

### 6.4 Find the VM's IP

```bash
virsh domifaddr vm1                    # e.g. 192.168.122.10
```

### 6.5 Add the VM to the Mac's `~/.ssh/config`

VMs on the default NAT network (`192.168.122.x`) are only reachable through the host, so use `ProxyJump`:

```
Host vm1
    HostName 192.168.122.10
    User user
    ProxyJump thinkcentre
```

Copy your key and connect:

```bash
ssh-copy-id vm1
ssh vm1
```

### 6.6 Serial Console (Emergency Access)

If the VM's network or SSH breaks:

```bash
virsh console vm1                      # exit with Ctrl + ]
```

If the console is blank, enable it once inside the VM:

```bash
sudo systemctl enable --now serial-getty@ttyS0
```

---

## Phase 7: Daily Cheat Sheet

All commands run from the Mac (thanks to `LIBVIRT_DEFAULT_URI`):

| Task | Command |
|---|---|
| List all VMs | `virsh list --all` |
| Start a VM | `virsh start vm1` |
| Graceful shutdown | `virsh shutdown vm1` |
| Force power-off | `virsh destroy vm1` |
| Reboot | `virsh reboot vm1` |
| Serial console | `virsh console vm1` (exit: `Ctrl + ]`) |
| VM info | `virsh dominfo vm1` |
| Find VM IP | `virsh domifaddr vm1` |
| Create snapshot | `virsh snapshot-create-as vm1 clean-install` |
| List snapshots | `virsh snapshot-list vm1` |
| Revert to snapshot | `virsh snapshot-revert vm1 clean-install` |
| Autostart on boot | `virsh autostart vm1` |
| Delete VM + disk | `virsh undefine vm1 --remove-all-storage` |
| SSH into VM | `ssh vm1` |
| Update the host | `ssh thinkcentre 'sudo dnf update -y'` |

---

## Optional: Access from Anywhere (Tailscale)

**On the host:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**On the Mac:** Install Tailscale from the App Store and sign in with the same account.

Then update `~/.ssh/config` so `HostName` for `thinkcentre` points to its Tailscale name or IP. All SSH, `virsh`, Cockpit, and `ProxyJump` VM access will work from anywhere, with no router port forwarding.

---

## Performance Notes

- **VMs run entirely on the ThinkCentre's hardware.** With KVM, guests typically get ~95%+ of bare-metal CPU performance. The Mac only sends keystrokes and receives text.
- **SSH on a home LAN** uses a few KB/s at ~1 ms latency, so typing feels instant, even over Wi-Fi. The extra `ProxyJump` hop adds no noticeable delay.
- **For best VM performance:**
  - CPU model: `host-passthrough`
  - Disk and network: **virtio**
  - Don't overcommit RAM; leave 1–2 CPU cores and some RAM for the host
  - An **SSD** in the ThinkCentre is the single biggest speed upgrade

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `virsh`: permission denied | Make sure `you` is in the `libvirt` group (`groups`), then log out and back in |
| virt-manager on Mac hangs when connecting | Use SSH keys (no passwords); include `?socket=...` and `--no-fork` |
| Cockpit page won't load | `sudo systemctl status cockpit.socket` and check the firewall (Phase 4.5) |
| `virsh domifaddr` shows nothing | Wait for the VM to fully boot; try `virsh domifaddr vm1 --source arp` |
| `virsh console` is blank | Press Enter; if still blank, enable `serial-getty@ttyS0` inside the VM |
| Rocky 10 VM won't boot | Use `--cpu host-passthrough` (Rocky 10 guests need x86-64-v3) |
| `virt-host-validate` fails on hardware virtualization | Enable VT-x in the BIOS (Phase 2.1) |
| Running out of disk space | Check `df -h /`; see the partitioning note in Phase 2.2 |
