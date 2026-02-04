---
title: "Automated Ubuntu VM Creation on Proxmox: A Practical Guide"
meta_title: "Automated Ubuntu VM - Proxmox"
description: "Automated Ubuntu VM Creation on Proxmox: A Practical Guide"
date: 2022-04-04T05:00:00.000Z
image: "media/assets/images/Screenshot 2026-02-04 012905.png"
categories:
  - Tutorial
  - Technology
  - Self-Hosting
author: "David B."
tags:
  - proxmox
  - bash
  - linux
draft: false
---
# Automated Ubuntu VM Creation on Proxmox: A Practical Guide

Launching virtual machines on Proxmox doesn't have to be tedious! This bash script automates the entire process of creating an Ubuntu cloud-init VM with configurable defaults, network and zero manual configuration. The following tutorial provides a pre-made script and breaks down how it works.

## What It Does

- Downloads provided cloud image automatically (Ubuntu Noble default)
- Validates all inputs (VMID, IP addresses, storage)
- Creates VM with modern defaults (UEFI, virtio drivers, QEMU agent)
- Configures networking (static IP or DHCP)
- Sets up cloud-init user and password
- Imports SSH public keys (optional)
- Resizes disk to your specification
- Starts the VM and cleans up temporary files
- Displays final SSH connection details

## Quick Start (Run as root on Proxmox)

```bash
wget https://raw.githubusercontent.com/cfunkz/Proxmox-Cloud-Init/main/start-vm
chmod +x start-vm
./start-vm
```

​[VIEW SCRIPT](https://github.com/cfunkz/Proxmox-Cloud-Init)​

## Why Automate VM Creation?

Creating VMs through the Proxmox UI involves a lot of clicks. This script turns the workflow into interactive prompts, validates inputs, and executes all the necessary `qm` commands automatically. It's perfect for infrastructure-as-code workflows or when you need consistent, repeatable setups.

## The Script Breakdown

### Error Handling & Dependencies

```bash
set -euo pipefail
```

This is bash's strict mode. The flags mean: exit on error (`-e`), treat undefined variables as errors (`-u`), and fail a pipe if any command fails (`-o pipefail`). This prevents silent failures.

```bash
die() { echo "ERROR: $*" >&2; exit 1; }
```

A helper function that prints errors to stderr and exits. Used throughout to bail out gracefully when something goes wrong.

```bash
need() { 
  command -v "$1" >/dev/null || die "Missing: $1"
}
```

Checks if required tools exist before proceeding. Calls like `need qm` and `need wget` ensure dependencies are available.

### Input Handling Functions

```bash
prompt_default() {
  local q="$1" d="$2" __v="$3" a
  read -rp "$q [$d]: " a
  printf -v "$__v" "%s" "${a:-$d}"
}
```

This function prompts the user with a question and shows a default value in brackets. If they press Enter (blank input), it uses the default. The `-v` flag in `printf` assigns the value to a variable name passed as the third argument. This is cleaner than echo and avoids subshells.

```bash
prompt_required() {
  local q="$1" __v="$2" a
  while [[ -z "${a:-}" ]]; do
    read -rp "$q: " a
  done
  printf -v "$__v" "%s" "$a"
}
```

Similar, but loops until the user enters something. No defaults—the field is mandatory. Used for VMID since you can't create a VM without one.

### Validation Functions

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

Validates IPv4 addresses in two steps. First, a regex checks the format. Then, it splits by dots and ensures each octet is 0-255. The `<<<` is a here-string, feeding the IP to the read command. Simple and effective.

## VM Configuration Phase

```bash
prompt_required "VMID (numeric)" VMID
[[ "$VMID" =~ ^[0-9]+$ ]] || die "VMID must be numeric"
qm status "$VMID" &>/dev/null && die "VMID $VMID already exists"
```

Gets the VM ID, validates it's numeric, then checks if it's already in use. The `&>/dev/null` silently discards both stdout and stderr from `qm status`, we only care if the command succeeds or fails.

```bash
prompt_default "Storage" "local-lvm" STORAGE
pvesm status | awk 'NR>1 {print $1}' | grep -qx "$STORAGE" || die "Storage '$STORAGE' not found"
```

Gets the storage name and validates it exists. The `awk` skips the header row (`NR>1`) and prints the first column. `grep -qx` does a quiet exact match. This catches typos early.

## Network Configuration: Static or DHCP

```bash
read -rp "Use DHCP? (y/N): " USE_DHCP
USE_DHCP="${USE_DHCP,,}"

if [[ "$USE_DHCP" == "y" ]]; then
  IPCONFIG0="ip=dhcp"
  IP_ADDR="(DHCP)"
else
  prompt_default "IP address" "192.168.1.64" IP_ADDR
  # ... prompt for static config ...
  IPCONFIG0="ip=${IP_ADDR}/${CIDR},gw=${GATEWAY}"
fi
```

The `"${USE_DHCP,,}"` converts to lowercase. If DHCP is chosen, `IPCONFIG0` is set to `ip=dhcp` and skips all static prompts. Otherwise, it builds the static config string. This variable is reused later during cloud-init setup.

## Image Download & VM Creation

```bash
wget -q --show-progress -O "$IMG_PATH" "$IMG_URL" || die "Download failed"
[[ -s "$IMG_PATH" ]] || die "Image file empty"
```

Downloads the Ubuntu cloud image. The `-q --show-progress` flags give quiet operation with a progress bar. The `-s` test checks file size > 0, preventing partial downloads from silently succeeding.

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

Creates the base VM. A few highlights:

* `--cpu host` passes through CPU features for better performance
* `--machine q35` and `--bios ovmf` enable UEFI boot (modern standard)
* `--agent enabled=1` installs QEMU guest agent for better integration
* `--serial0 socket` enables serial console access

## Disk Handling

```bash
qm importdisk "$VMID" "$IMG_PATH" "$STORAGE" --format raw
qm set "$VMID" --scsi0 "${STORAGE}:vm-${VMID}-disk-0,discard=on,iothread=1"
```

Imports the cloud image and attaches it as the main disk. The `discard=on` enables TRIM to reclaim space, and `iothread=1` improves I/O performance by using a dedicated thread.

```bash
qm set "$VMID" --ide2 "$STORAGE:cloudinit"
```

Adds a cloud-init drive. This is where Proxmox stores the network config and user data.

## Cloud-Init Configuration

```bash
qm set "$VMID" \
  --ciuser "$CIUSER" \
  --cipassword "$CIPASS" \
  --ipconfig0 "$IPCONFIG0"

if [[ "$USE_DHCP" != "y" ]]; then
  qm set "$VMID" --nameserver "$DNS"
fi
```

Sets up cloud-init with the user, password, and network config. DNS is only set for static configs and DHCP provides it automatically. Notice how the `IPCONFIG0` variable we built earlier is reused here.

```bash
if [[ -n "$SSHKEY_LINE" ]]; then
  qm set "$VMID" --sshkeys <(echo "$SSHKEY_LINE")
fi
```

Optionally sets SSH public keys. The `<(...)` is process substitution as `qm` reads from a file-like interface without needing a temp file.

## Startup & Cleanup

```bash
qm resize "$VMID" scsi0 "$DISK_SIZE"
qm set "$VMID" --boot order=scsi0
qm start "$VMID"
rm -f "$IMG_PATH"
```

Resizes the disk to the requested size, sets the boot order, starts the VM, and cleans up the downloaded image. Everything is automated.




​
