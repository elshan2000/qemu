# QEMU Virtual Machine Guide

**Run a complete VM with just 4 commands**

QEMU is a **machine emulator and virtualizer**.
It can emulate entire systems, including CPU, memory, and devices — running **any OS on any CPU architecture**.

---

## Modes of Operation

### QEMU Without KVM

* QEMU **emulates every CPU instruction** in software.
* Works on **any host CPU**, regardless of architecture.
* **Very slow** performance — good for testing, debugging, or simulation.

### QEMU With KVM

* QEMU **offloads CPU execution** to hardware via KVM (Kernel-based Virtual Machine).
* Performance close to native (up to **10× faster**).
* Requires CPU virtualization support:

| Vendor    | Feature       | Description                            |
| --------- | ------------- | -------------------------------------- |
| **Intel** | VT-x (`vmx`)  | Hardware virtualization for Intel CPUs |
| **AMD**   | AMD-V (`svm`) | Hardware virtualization for AMD CPUs   |

---

## Create a Virtual Disk

```bash
qemu-img create -f qcow2 win11.img 30G
```

* `qcow2` format supports **snapshots**, compression, and dynamic sizing.

---

## Install Operating System

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -m 6G \
  -smp 4 \
  -hda win11.img \
  -boot d \
  -cdrom windows_11.iso
```

**Notes:**

* `-enable-kvm` → Use hardware acceleration.
* `-boot d` → Boot from CD-ROM for OS installation.
* `-smp 4` → Allocate 4 CPU cores.
* `-m 6G` → Allocate 6 GB RAM.

---

## Run the Installed OS with SPICE (QXL Display)

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -m 6G \
  -smp 4 \
  -device qxl-vga \
  -hda win11.img \
  -boot c \
  -spice port=5905,addr=127.0.0.1,disable-ticketing=on \
  -device virtio-serial-pci \
  -chardev spicevmc,id=vdagent,debug=0,name=vdagent \
  -device virtserialport,chardev=vdagent,name=com.redhat.spice.0 \
  -display none
```

**What this does:**

* Uses **QXL video driver** for better graphics performance.
* Enables **SPICE remote desktop** for high-speed display and clipboard sharing.
* Adds **SPICE agent** for mouse sync, clipboard, and resolution change.

---

## Connect to the VM via SPICE

### On the Host:

```bash
sudo apt install virt-viewer
remote-viewer spice://127.0.0.1:5905
```

### Inside the Guest:

Install **SPICE Guest Tools**:
[https://www.spice-space.org/download.html](https://www.spice-space.org/download.html)

---
# Exploring QEMU Features

## Disk Options

| Option                               | Description                                    |
| ------------------------------------ | ---------------------------------------------- |
| `-hda win10.img`                     | Simple legacy method to attach a disk          |
| `-drive file=win10.img,format=qcow2` | Recommended modern syntax (supports snapshots) |

**Use `-drive`** syntax — it’s more flexible and supports multiple disks easily.

---

## CPU Emulation

You can emulate or expose specific CPU models:

```bash
qemu-system-x86_64 -enable-kvm -cpu Haswell -m 6G -smp 4 -hda win11.img -boot c
```

* `-cpu EPYC` → emulate AMD CPU
* `-cpu Haswell` → emulate Intel CPU
* `-cpu host` → pass host CPU features directly
* You can even emulate 32-bit CPUs on a 64-bit host.

Check all options:

```bash
qemu-system-x86_64 -cpu help
```

---

## Networking Modes

| Mode       | Description                            | Visibility     | Example                                                          |
| ---------- | -------------------------------------- | -------------- | ---------------------------------------------------------------- |
| **user**   | Built-in NAT mode (no root needed)     | VM behind NAT  | `-nic user,model=virtio-net-pci`                                 |
| **bridge** | Connects VM to host LAN (same network) | Visible on LAN | `-nic bridge,br=br0,model=virtio-net-pci`                        |
| **tap**    | Direct virtual Ethernet interface      | Fastest        | `-netdev tap,id=n0,ifname=tap0 -device virtio-net-pci,netdev=n0` |

---

## TUN vs TAP

| Device  | Layer              | Use Case                              |
| ------- | ------------------ | ------------------------------------- |
| **TUN** | Layer 3 (IP)       | Used by VPNs for IP tunneling         |
| **TAP** | Layer 2 (Ethernet) | Used for VMs, bridges, and containers |

Think of:

* **TAP** → virtual Ethernet cable
* **TUN** → virtual IP pipe

Create a TAP device manually:

```bash
sudo ip tuntap add dev tap0 mode tap user $USER
```

---

## Port Forwarding (for SSH or HTTP)

Example:

```bash
-netdev user,id=n0,hostfwd=tcp::2222-:22
```

Maps:

* **Host port 2222 → Guest port 22**

Then you can SSH into the VM:

```bash
ssh -p 2222 user@127.0.0.1
```

---

## Summary

* **QEMU without KVM** → universal but slow (pure software).
* **QEMU with KVM** → near-native performance (hardware-accelerated).
* **QXL + SPICE** → better graphics, mouse sync, clipboard sharing.
* **qcow2** → supports snapshots and compression.
* **VirtIO** → faster disk and network drivers.
* **Create and run a VM** → with just 2 commands
