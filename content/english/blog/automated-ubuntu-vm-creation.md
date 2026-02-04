---
title: Automated Ubuntu VM Creation on Proxmox (Cloud-Init, No UI Clicks)
meta_title: "Automated Ubuntu VM Creation on Proxmox with Cloud-Init"
description: "Create Ubuntu virtual machines on Proxmox automatically using a
  Bash script and cloud-init. No UI clicks, validated inputs, SSH keys, and
  clean defaults."
---
## Automated Ubuntu VM Creation on Proxmox (Cloud-Init, No UI Clicks)

### Meta title

**Automated Ubuntu VM Creation on Proxmox with Cloud-Init**

### Meta description

Create Ubuntu virtual machines on Proxmox automatically using a Bash script and cloud-init. No UI clicks, validated inputs, SSH keys, and clean defaults.

***

Spinning up Ubuntu VMs on Proxmox is straightforward, but doing it _repeatedly_ through the web UI gets old fast. Too many clicks, easy to miss a setting, and hard to keep things consistent.

This guide walks through a Bash script that automates the entire process. It creates an Ubuntu cloud-init VM with sensible defaults, handles networking, validates inputs, and boots the VM ready for SSH — all from the command line.

It’s designed for people who want **simple**, **repeatable, predictable VM creation**.

***

### Quick start (run as root on the Proxmox host)

```bash
wget https://raw.githubusercontent.com/cfunkz/Proxmox-Cloud-Init/main/start-vm
chmod +x start-vm
./start-vm
```

Full script and updates live here:
👉 [https://github.com/cfunkz/Proxmox-Cloud-Init](https://github.com/cfunkz/Proxmox-Cloud-Init)​

***

### What this script handles for you

At a high level, the script automates everything you’d normally click through in the Proxmox UI:

* Downloads the Ubuntu cloud image automatically (Ubuntu Noble by default)
* Validates all inputs (VMID, storage, IP addresses)
* Creates the VM with modern defaults (UEFI, VirtIO, QEMU guest agent)
* Supports DHCP or static networking
* Configures cloud-init user and password
* Optionally injects SSH public keys
* Resizes the disk to your chosen size
* Starts the VM and removes temporary files
* Prints final SSH connection details

The goal is simple: **no surprises and no half-configured VMs**.

***

### Why automate VM creation on Proxmox?

The Proxmox UI works fine for one-off VMs. It’s less great when you need to:

* spin up multiple machines quickly
* keep configs consistent across environments
* avoid human error
* document your setup as code

This script turns VM creation into a guided, interactive workflow. You answer a few prompts, and the script runs all the required `qm` commands in the correct order.

***

### Script walkthrough

Below is a breakdown of the more important parts of the script and why they exist.

***

#### Strict mode and error handling

```bash
set -euo pipefail
```

This enables Bash “strict mode”:

* exit immediately on errors
* fail on undefined variables
* catch failures inside pipelines

It prevents the script from silently continuing when something goes wrong.

```bash
die() { echo "ERROR: $*" >&2; exit 1; }
```

A small helper used throughout the script to print errors clearly and exit cleanly.

```bash
need() {
  command -v "$1" >/dev/null || die "Missing: $1"
}
```

Before doing anything useful, the script checks for required tools like `qm` and `wget`. If something’s missing, it fails early instead of halfway through VM creation.

***

#### Prompting for input (with sane defaults)

```bash
prompt_default() {
  local q="$1" d="$2" __v="$3" a
  read -rp "$q [$d]: " a
  printf -v "$__v" "%s" "${a:-$d}"
}
```

This helper prompts the user with a default value. If Enter is pressed, the default is used. Using `printf -v` avoids subshells and keeps everything clean.

```bash
prompt_required() {
  local q="$1" __v="$2" a
  while [[ -z "${a:-}" ]]; do
    read -rp "$q: " a
  done
  printf -v "$__v" "%s" "$a"
}
```

Same idea, but for required fields. This is used for things like VMID, where an empty value isn’t acceptable.

***

#### Input validation

```bash
is_ipv4() {
  local ip="$1"
  [[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] || return 1
  IFS='.' read -r a b c d <<<"$ip"
  for o in "$a" "$b" "$c" "$d"; do
    [[ "$o" -ge 0 && "$o" -le 255 ]] || return 1
  done
}
```

IP validation happens in two stages:

1. a regex checks the format
2. each octet is verified to be between 0–255

It’s simple, readable, and catches bad input early.

***

### VM configuration

```bash
prompt_required "VMID (numeric)" VMID
[[ "$VMID" =~ ^[0-9]+$ ]] || die "VMID must be numeric"
qm status "$VMID" &>/dev/null && die "VMID $VMID already exists"
```

The script ensures the VMID is numeric and not already in use. Output from `qm status` is discarded because only success or failure matters here.

```bash
prompt_default "Storage" "local-lvm" STORAGE
pvesm status | awk 'NR>1 {print $1}' | grep -qx "$STORAGE" || die "Storage '$STORAGE' not found"
```

Storage is validated against existing Proxmox storage pools to avoid typos that would otherwise fail later.

***

### Network setup (DHCP or static)

```bash
read -rp "Use DHCP? (y/N): " USE_DHCP
USE_DHCP="${USE_DHCP,,}"
```

User input is normalized to lowercase for easier comparison.

If DHCP is selected:

```bash
IPCONFIG0="ip=dhcp"
```

Otherwise, the script prompts for static values and builds a proper cloud-init network string:

```bash
IPCONFIG0="ip=${IP_ADDR}/${CIDR},gw=${GATEWAY}"
```

This same variable is reused later when configuring cloud-init.

***

### Image download and VM creation

```bash
wget -q --show-progress -O "$IMG_PATH" "$IMG_URL" || die "Download failed"
[[ -s "$IMG_PATH" ]] || die "Image file empty"
```

The cloud image is downloaded with progress output and checked to ensure it isn’t empty or corrupted.

```bash
qm create "$VMID" \
  --name "$VMNAME" \
  --memory "$MEM" \
  --cores "$CORES" \
  --cpu host \
  --machine q35 \
  --bios ovmf \
  --scsihw virtio-scsi-pci \
  --net0 "virtio,bridge=$BRIDGE" \
  --agent enabled=1 \
  --serial0 socket \
  --vga serial0
```

This creates the VM with:

* UEFI boot
* VirtIO drivers
* host CPU passthrough
* QEMU guest agent enabled
* serial console access

These defaults work well for most modern workloads.

***

### Disk and cloud-init setup

```bash
qm importdisk "$VMID" "$IMG_PATH" "$STORAGE" --format raw
qm set "$VMID" --scsi0 "${STORAGE}:vm-${VMID}-disk-0,discard=on,iothread=1"
```

The cloud image is imported and attached as the primary disk. TRIM and I/O threading are enabled for better performance.

```bash
qm set "$VMID" --ide2 "$STORAGE:cloudinit"
```

Adds the cloud-init drive where Proxmox stores user and network data.

***

### Cloud-init configuration

```bash
qm set "$VMID" \
  --ciuser "$CIUSER" \
  --cipassword "$CIPASS" \
  --ipconfig0 "$IPCONFIG0"
```

User, password, and networking are applied in one step.

```bash
if [[ "$USE_DHCP" != "y" ]]; then
  qm set "$VMID" --nameserver "$DNS"
fi
```

DNS is only set for static configurations. DHCP handles it automatically.

```bash
if [[ -n "$SSHKEY_LINE" ]]; then
  qm set "$VMID" --sshkeys <(echo "$SSHKEY_LINE")
fi
```

SSH keys are injected using process substitution, avoiding temporary files entirely.

***

### Startup and cleanup

```bash
qm resize "$VMID" scsi0 "$DISK_SIZE"
qm set "$VMID" --boot order=scsi0
qm start "$VMID"
rm -f "$IMG_PATH"
```

The disk is resized, boot order set, the VM started, and the downloaded image cleaned up.

***

### Final thoughts

This script isn’t trying to replace Terraform or Ansible. It’s meant to solve a very specific problem: **creating clean Ubuntu VMs on Proxmox quickly and consistently**.

If you spend time clicking through the UI or retyping the same `qm` commands, this approach saves time and avoids mistakes while still being transparent and easy to modify.
