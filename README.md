# Headless KVM Host on Rocky Linux, Managed from a MacBook

Step-by-step guide to turning a Lenovo ThinkCentre into a headless KVM virtualization host on Rocky Linux, fully managed from a MacBook via SSH, virsh, Cockpit, and virt-manager. Includes setup for office and shared networks without router access (mDNS, Tailscale, or the open-source alternative NetBird), and instructions for both **fish** and **zsh** shells on macOS.

---

## Table of Contents

1. [Overview](#overview)
2. [Placeholders Used](#placeholders-used)
3. [Phase 1: Preparation (Mac)](#phase-1-preparation-mac)
4. [Phase 2: Install Rocky Linux (ThinkCentre)](#phase-2-install-rocky-linux-thinkcentre)
   - [2.4 No Router Access (Office / Shared Networks)](#24-no-router-access-office--shared-networks)
5. [Phase 3: SSH Access (Mac)](#phase-3-ssh-access-mac)
6. [Phase 4: Configure the Host (over SSH)](#phase-4-configure-the-host-over-ssh)
7. [Phase 5: Management Tools (Mac)](#phase-5-management-tools-mac)
8. [Phase 6: Create Your First VM](#phase-6-create-your-first-vm)
9. [Phase 7: Daily Cheat Sheet](#phase-7-daily-cheat-sheet)
10. [Optional: Access from Anywhere (Tailscale or NetBird)](#optional-access-from-anywhere-tailscale-or-netbird)
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
- **Cockpit**: web dashboard at `https://<host>:9090`, including VM consoles in the browser
- **virt-manager (optional)**: GUI app installed on the Mac via Homebrew, connects over SSH

> **Note:** virt-manager is deprecated and **not available on Rocky Linux 10 hosts**; Cockpit is the official replacement. Running virt-manager *on your Mac* still works, because it only talks to libvirt over SSH.

> **Shells:** Mac commands in this guide work in both **fish** and **zsh**. Where they differ, both versions are shown. Commands run on the ThinkCentre or inside VMs use the Linux default shell (bash).

---

## Placeholders Used

Replace these with your own values throughout the guide:

| Item | Example |
|---|---|
| ThinkCentre IP | `192.168.1.50` |
| ThinkCentre hostname | `thinkcentre` |
| Router / gateway | `192.168.1.1` |
| Linux username on host | `you` |
| Username inside VM | `user` |
| First VM name | `vm1` |

> **On an office or shared network without router access?** Wherever this guide uses `192.168.1.50`, use `thinkcentre.local`, the Tailscale name (`thinkcentre`), or the NetBird name (`thinkcentre.netbird.cloud`) instead. See [2.4 No Router Access](#24-no-router-access-office--shared-networks).

---

## Phase 1: Preparation (Mac)

### 1.1 Mac Prerequisites (Homebrew + Shell)

Install Homebrew if you don't have it:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Homebrew's installer only prints PATH setup instructions for zsh/bash. Add Homebrew to your shell:

**fish** (add to `~/.config/fish/config.fish`):

```fish
# Apple Silicon
/opt/homebrew/bin/brew shellenv | source
# Intel Macs: use /usr/local/bin/brew instead
```

**zsh** (add to `~/.zprofile`):

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Open a new terminal and confirm:

```bash
brew --version
```

### 1.2 Check the ThinkCentre's CPU

Rocky Linux 10 requires an **x86-64-v3** CPU (Intel: roughly 4th-gen Core / Haswell or newer).

> **Check the exact CPU model, not just the generation.** Many ThinkCentre Tiny models ship with **Pentium or Celeron** chips, which lack AVX even in recent generations and are therefore *not* x86-64-v3.

- If your CPU is older (or a Pentium/Celeron), use **Rocky Linux 9** instead (supported until 2032, and still includes virt-manager on the host). The rest of this guide is nearly identical.
- If a Linux system is already on the machine, you can check with:

```bash
/lib64/ld-linux-x86-64.so.2 --help | grep supported
```

If `x86-64-v3 (supported, searched)` appears, Rocky 10 will work.

### 1.3 Download the ISO

From <https://rockylinux.org/download>, download the **x86_64 Minimal ISO**.

> Make sure it's **x86_64**, not ARM/aarch64, even though you're downloading on a Mac.

### 1.4 Create a Bootable USB

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
diskutil eject /dev/disk4
```

> **fish users:** fish treats a wildcard that matches nothing as an error ("No matches for wildcard") instead of passing it through. If you see this, run `ls ~/Downloads` and type the ISO filename out in full.

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
| **Network & Hostname** | Enable Ethernet; set hostname to `thinkcentre` |
| **User Creation** | Create `you`; tick **"Make this user administrator"** (root can stay locked) |
| **Installation Destination** | See below (important) |

**Disk layout (important):** The default automatic layout gives `/` only ~70 GB and puts the rest into `/home`. VM disks live in `/var/lib/libvirt/images` (on `/`), so you'd run out of space quickly. The default filesystem (XFS) can't be shrunk later, so fix this now.

1. Choose **Custom**.
2. Click **"Click here to create them automatically."**
3. **Delete `/home`.**
4. Set `/` to use all remaining space.

Click **Begin Installation**, then reboot when it finishes.

### 2.3 Give the Host a Fixed IP (Home / Router Access)

> **No router access?** Skip this step and go to [2.4](#24-no-router-access-office--shared-networks).

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

### 2.4 No Router Access (Office / Shared Networks)

If you can't access the router (common in offices), **don't rely on a fixed IP**. Reach the host by **name** instead, so it doesn't matter if its IP changes.

#### Step 1: Check Connectivity

Office networks often separate Wi-Fi from wired devices or block device-to-device traffic. At the ThinkCentre console, find its current IP:

```bash
ip a                                   # e.g. 10.20.5.37
```

Then from the Mac:

```bash
ping 10.20.5.37
ssh you@10.20.5.37
```

| Result | Next Step |
|---|---|
| Ping and SSH work | Use **Option A** (mDNS) |
| Timeout / no response | The network isolates devices; use **Option B** (mesh VPN: Tailscale or NetBird) |

#### Option A: Hostname via mDNS (`thinkcentre.local`)

macOS resolves `.local` names natively, so you only need to set this up on the host:

```bash
sudo hostnamectl set-hostname thinkcentre
sudo dnf install -y avahi
sudo systemctl enable --now avahi-daemon
sudo firewall-cmd --add-service=mdns --permanent
sudo firewall-cmd --reload
```

Test from the Mac:

```bash
ping thinkcentre.local
```

Use `thinkcentre.local` as the `HostName` in [Phase 3](#phase-3-ssh-access-mac). Everything else in this guide works unchanged:

- `ssh thinkcentre`
- `virsh` (via `LIBVIRT_DEFAULT_URI`)
- `ssh vm1` (via `ProxyJump`)
- Cockpit at `https://thinkcentre.local:9090`

> **Limitation:** mDNS only works within the same network segment. If the Mac is on office Wi-Fi and the ThinkCentre is wired, they may be on different segments. In that case `.local` won't resolve even if ping by IP works, so use Option B.

#### Option B: Mesh VPN (Tailscale or NetBird; Works on Any Network Layout)

A mesh VPN creates a private connection between your devices. It works across Wi-Fi/wired separation and IP changes, and from outside the office. Choose one:

| | Tailscale | NetBird |
|---|---|---|
| Cost | Free Personal plan | Free hosted tier |
| License | Open-source clients, proprietary coordination server | Fully open source; can be self-hosted later |
| Host name to use | `thinkcentre` (MagicDNS) | `thinkcentre.netbird.cloud` |

1. Follow the setup in [Optional: Access from Anywhere](#optional-access-from-anywhere-tailscale-or-netbird). For the first connection, use the host's current IP from Step 1, or run the host commands at the console.
2. Use the VPN name or its `100.x.x.x` IP as the `HostName` in [Phase 3](#phase-3-ssh-access-mac).

> **Note:** Some companies don't allow VPN-style tools. Check your IT policy first.

#### Things to Avoid on an Office Network

| Avoid | Why | Instead |
|---|---|---|
| Setting a static IP yourself | It may conflict with an address the DHCP server gives to someone else | Ask IT for a DHCP reservation or an assigned static IP |
| Bridged VM networking | Puts each VM directly on the office network, using office IPs | Keep VMs on the default **NAT** network (hidden behind the host) |
| Running a server without telling anyone | Security tools may flag a new machine running SSH | Let IT know about the setup |

---

## Phase 3: SSH Access (Mac)

### 3.1 Create and Copy an SSH Key

```bash
ssh-keygen -t ed25519                  # skip if you already have a key
ssh-copy-id you@192.168.1.50           # or you@thinkcentre.local
```

### 3.2 Add the Host to `~/.ssh/config`

Keep exactly one `HostName` line, matching your network. (Comments go on their own lines; older OpenSSH versions reject comments at the end of a setting.)

```
Host thinkcentre
    # Home with fixed IP:
    HostName 192.168.1.50
    # Office, mDNS (2.4 Option A):
    # HostName thinkcentre.local
    # Tailscale (2.4 Option B):
    # HostName thinkcentre
    # NetBird (2.4 Option B):
    # HostName thinkcentre.netbird.cloud
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

Make plain `virsh` on the host target the system VMs. Without this, `virsh` run as a normal user connects to `qemu:///session` and won't see VMs created with `sudo`:

```bash
echo 'export LIBVIRT_DEFAULT_URI=qemu:///system' >> ~/.bashrc
```

Log out and back in for the group change and variable to take effect.

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

sshd uses the **first** value it reads for each setting, and files in `sshd_config.d/` load in alphabetical order. The file is named `00-...` so it wins over installer-created files such as `01-permitrootlogin.conf`.

```bash
printf "PasswordAuthentication no\nKbdInteractiveAuthentication no\nPermitRootLogin no\n" | \
  sudo tee /etc/ssh/sshd_config.d/00-hardening.conf
sudo sshd -t && sudo systemctl restart sshd
sudo sshd -T | grep -Ei 'passwordauth|kbdinteractive|permitrootlogin'
```

All three settings should show `no`. Open a **new** terminal on the Mac and run `ssh thinkcentre` to confirm access **before** closing your current session.

---

## Phase 5: Management Tools (Mac)

### 5.1 libvirt Client (`virsh` on the Mac)

```bash
brew install libvirt
```

Set the default connection so `virsh` always targets the ThinkCentre:

**fish** (universal variable, persists across sessions and restarts):

```fish
set -Ux LIBVIRT_DEFAULT_URI qemu+ssh://thinkcentre/system
```

Or, to keep it in your config file, add this to `~/.config/fish/config.fish`:

```fish
set -gx LIBVIRT_DEFAULT_URI qemu+ssh://thinkcentre/system
```

**zsh:**

```bash
echo 'export LIBVIRT_DEFAULT_URI="qemu+ssh://thinkcentre/system"' >> ~/.zshrc
source ~/.zshrc
```

Test it:

```bash
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
- If you see `nc: command not found`, install netcat on the host: `sudo dnf install -y nmap-ncat`.
- The Mac build is community-maintained; if an update breaks it, use Cockpit.

### 5.3 Cockpit (Web Dashboard)

1. Open `https://192.168.1.50:9090` (or `https://thinkcentre.local:9090`, `https://thinkcentre.netbird.cloud:9090`, etc.) in your browser.
2. Accept the self-signed certificate warning.
3. Log in as `you` and click **"Turn on administrative access."**
4. Use the **Virtual Machines** tab to create, edit, and view VMs (with a graphical console in the browser).

---

## Phase 6: Create Your First VM

> Steps 6.1–6.2 run **on the ThinkCentre** (`ssh thinkcentre`). From 6.3 onward, `virsh` commands work from the Mac (via `LIBVIRT_DEFAULT_URI`) or the host (via the variable set in 4.2).

### 6.1 Download an Installer ISO on the Host

Downloading directly on the host is faster than copying from the Mac:

```bash
cd /var/lib/libvirt/images
sudo curl -LO https://download.rockylinux.org/pub/rocky/10/isos/x86_64/Rocky-10-latest-x86_64-minimal.iso
```

> If `curl` returns a 404, open the directory in a browser and copy the current filename; the "latest" names change occasionally.

### 6.2 Create the VM (Text-Mode Installer in Your Terminal)

Start a `tmux` session first, so a dropped SSH connection doesn't interrupt the installer view (reattach with `tmux attach -t install`):

```bash
tmux new -s install
```

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

### 6.4 Find and Pin the VM's IP

```bash
virsh domifaddr vm1                    # e.g. 192.168.122.10
```

libvirt's DHCP leases aren't guaranteed to persist, so pin the address to keep `ProxyJump` working:

```bash
virsh domiflist vm1                    # note the MAC, e.g. 52:54:00:ab:cd:ef
virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:ab:cd:ef' name='vm1' ip='192.168.122.10'/>" --live --config
```

The reservation applies the next time the VM requests a lease (for example, after a reboot).

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

All commands run from the Mac (thanks to `LIBVIRT_DEFAULT_URI`), in fish or zsh:

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
| Delete VM + disk | `virsh undefine vm1 --remove-all-storage --nvram --snapshots-metadata` |
| SSH into VM | `ssh vm1` |
| Update the host | `ssh thinkcentre 'sudo dnf update -y'` |
| VPN status (NetBird) | `netbird status` (add `-d` for details) |
| VPN status (Tailscale) | `tailscale status` |

---

## Optional: Access from Anywhere (Tailscale or NetBird)

Both options give the ThinkCentre a stable private name and IP that work from any network, with no router port forwarding. Once connected, update `HostName` for `thinkcentre` in `~/.ssh/config` (Phase 3.2). All SSH, `virsh`, Cockpit, and `ProxyJump` VM access then works unchanged.

> In an office, check your IT policy before installing either one.

### Option 1: Tailscale

**On the host:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up                      # prints a login URL; open it on the Mac
```

**On the Mac:** Install Tailscale from the App Store and sign in with the same account.

Use `HostName thinkcentre` (MagicDNS) or the host's `100.x.x.x` IP.

### Option 2: NetBird (Open Source)

NetBird is a fully open-source alternative with a free hosted tier. You can self-host its control server later without changing clients. The headless host joins with a **setup key** instead of a browser login.

**1. Create an account** at <https://app.netbird.io>.

**2. Connect the Mac.** Download the macOS installer from the dashboard (**Add Peer → macOS**), open NetBird from the menu bar, click **Connect**, and sign in. Or use Homebrew:

```bash
brew install netbirdio/tap/netbird
brew install --cask netbirdio/tap/netbird-ui
sudo netbird service install
sudo netbird service start
netbird up                             # opens the browser to sign in
```

**3. Create a setup key.** In the dashboard: **Setup Keys → Create Setup Key**, then name it and click **Create**. A one-off key is fine.

> Don't mark the key as **ephemeral**, or the host is removed from your network whenever it goes offline.

**4. Connect the ThinkCentre:**

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sh
sudo netbird up --setup-key <YOUR_SETUP_KEY>
netbird status
```

The script adds NetBird's repository and installs it as a system service, so it reconnects after reboots and `dnf update` keeps it current.

**5. Find the host's name.** Each peer gets a name under `netbird.cloud`, based on its hostname: usually `thinkcentre.netbird.cloud`. Confirm it on the dashboard's **Peers** page or with `netbird status -d`, then test from the Mac:

```bash
ping thinkcentre.netbird.cloud
```

Use `HostName thinkcentre.netbird.cloud` in `~/.ssh/config`, and Cockpit at `https://thinkcentre.netbird.cloud:9090`.

**NetBird commands:**

| Task | Command |
|---|---|
| Check connection and peers | `netbird status` (add `-d` for details) |
| Connect | `netbird up` |
| Disconnect | `netbird down` |
| Remove device from your network | `sudo netbird logout` |
| Restart the service (host) | `sudo systemctl restart netbird` |

**Tips:**

- The default access policy lets all your peers reach each other. If you add other people or devices, tighten it under **Access Control → Policies**, for example by allowing only your Mac to reach the ThinkCentre on ports 22 and 9090.
- NetBird connects devices directly when possible and falls back to relay servers on strict networks. Relayed connections work but are slower; `netbird status -d` shows whether each peer is direct (P2P) or relayed.

---

## Performance Notes

- **VMs run entirely on the ThinkCentre's hardware.** With KVM, guests typically get ~95%+ of bare-metal CPU performance. The Mac only sends keystrokes and receives text.
- **SSH on a local network** uses a few KB/s at ~1 ms latency, so typing feels instant, even over Wi-Fi. The extra `ProxyJump` hop adds no noticeable delay. Over a mesh VPN, latency depends on whether the connection is direct or relayed.
- **For best VM performance:**
  - CPU model: `host-passthrough`
  - Disk and network: **virtio**
  - Don't overcommit RAM; leave 1–2 CPU cores and some RAM for the host
  - An **SSD** in the ThinkCentre is the single biggest speed upgrade

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `brew: command not found` (fish) | Add `/opt/homebrew/bin/brew shellenv \| source` to `~/.config/fish/config.fish` (Phase 1.1) |
| "No matches for wildcard" (fish) | Type the full filename instead of using `*` (Phase 1.4) |
| `virsh` connects to the Mac instead of the host | Check `echo $LIBVIRT_DEFAULT_URI`; set it as shown in Phase 5.1 for your shell |
| `virsh` on the host says the VM doesn't exist | Plain `virsh` is using `qemu:///session`; set `LIBVIRT_DEFAULT_URI=qemu:///system` (Phase 4.2) |
| `virsh`: permission denied | Make sure `you` is in the `libvirt` group (`groups`), then log out and back in |
| virt-manager on Mac hangs when connecting | Use SSH keys (no passwords); include `?socket=...` and `--no-fork` |
| virt-manager: `nc: command not found` | On the host: `sudo dnf install -y nmap-ncat` |
| SSH still accepts passwords after hardening | Run `sudo sshd -T \| grep -i passwordauth`; make sure the hardening file sorts first (`00-...`, Phase 4.6) |
| Cockpit page won't load | `sudo systemctl status cockpit.socket` and check the firewall (Phase 4.5) |
| `thinkcentre.local` doesn't resolve | Check `avahi-daemon` is running and `mdns` is allowed in the firewall; if Mac and host are on different segments, use a mesh VPN (2.4 Option B) |
| Can't reach the host at all on office network | The network may isolate devices; use Tailscale or NetBird (2.4 Option B) or ask IT |
| Host IP changed and SSH stopped working | Use `thinkcentre.local` or a mesh VPN name instead of a raw IP (2.4) |
| `thinkcentre.netbird.cloud` doesn't resolve | Run `netbird status` on both machines; confirm the exact peer name on the dashboard's **Peers** page |
| NetBird ping works but SSH/Cockpit times out | `sudo firewall-cmd --get-active-zones`; NetBird's interface must be in a zone allowing `ssh` and `cockpit` (the default `public` zone does) |
| Host disappeared from NetBird after going offline | The setup key was ephemeral; create a non-ephemeral key and run `sudo netbird up --setup-key ...` again |
| NetBird connection feels slow | `netbird status -d`; a relayed connection is slower than direct P2P, which strict networks may block |
| `virsh domifaddr` shows nothing | Wait for the VM to fully boot; try `virsh domifaddr vm1 --source arp` |
| VM IP changed and `ssh vm1` fails | Pin the IP with a DHCP reservation (Phase 6.4) |
| `virsh console` is blank | Press Enter; if still blank, enable `serial-getty@ttyS0` inside the VM |
| `virsh undefine` fails (nvram or snapshots) | Add `--nvram --snapshots-metadata` (Phase 7) |
| Rocky 10 VM won't boot | Use `--cpu host-passthrough` (Rocky 10 guests need x86-64-v3) |
| `virt-host-validate` fails on hardware virtualization | Enable VT-x in the BIOS (Phase 2.1) |
| Running out of disk space | Check `df -h /`; see the partitioning note in Phase 2.2 |
