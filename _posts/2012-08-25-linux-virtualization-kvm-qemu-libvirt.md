---
layout: post
title: "KVM, QEMU, and libvirt: Running Virtual Machines on Linux"
date: Aug-25-2012
categories: [Linux, SysAdmin]
tags: [linux, virtualization, kvm, qemu, libvirt, vm]
image: /assets/img/virtualization-kvm-server-room.jpg
---

Every cloud instance you have ever rented is, underneath, a Linux box running someone else's virtual machine on top of KVM. You do not need a hypervisor vendor or a GUI to do the same thing on your own hardware. The kernel already has a type-1 hypervisor built in; QEMU and libvirt are what turn it into something you can actually drive.

## Checking That Your Hardware Can Virtualize

Before installing anything, confirm the CPU exposes virtualization extensions and the kernel module is loaded. Nearly every x86_64 server and desktop from the last decade supports this, but it is sometimes disabled in firmware.

```bash
# Intel VT-x or AMD-V present?
egrep -c '(vmx|svm)' /proc/cpuinfo

# Is the kvm module loaded?
lsmod | grep kvm

# A friendlier check on Debian/Ubuntu
sudo apt install cpu-checker
kvm-ok
```

*Tip:* A count of `0` from the `egrep` command almost always means virtualization is disabled in the BIOS/UEFI, not that the CPU lacks support. Reboot into firmware setup before assuming the hardware can't do it.

## Installing the KVM, QEMU, and libvirt Stack

`KVM` is the kernel module, `QEMU` emulates the devices a VM sees, and `libvirt` is the management layer that gives you `virsh` and a stable API instead of hand-built QEMU command lines.

```bash
# Debian/Ubuntu
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst

# RHEL/CentOS/Fedora
sudo dnf install qemu-kvm libvirt virt-install bridge-utils

# Add yourself to the libvirt group so you don't need sudo for every virsh command
sudo usermod -aG libvirt,kvm $USER

# Confirm the daemon is running
sudo systemctl status libvirtd
```

*Tip:* Log out and back in after the `usermod`, or the group membership won't take effect in your current shell session.

## Managing VM Lifecycle with virsh

`virsh` is the single command you will use for almost everything: listing, starting, stopping, and permanently removing virtual machines.

```bash
# List running VMs
virsh list

# List every VM, running or not
virsh list --all

# Start, gracefully shut down, or force-stop a VM
virsh start webserver01
virsh shutdown webserver01
virsh destroy webserver01

# Permanently remove the VM definition (does not touch its disk)
virsh undefine webserver01

# Make a VM start automatically when the host boots
virsh autostart webserver01
```

*Tip:* `virsh destroy` does not delete anything, despite the alarming name. It is the equivalent of pulling the power cord. `undefine` is the one that actually removes the VM's configuration.

## Creating a VM from the Command Line with virt-install

You rarely need a GUI to provision a new VM. `virt-install` scripts the whole process: disk creation, network attachment, and the install media.

```bash
sudo virt-install \
  --name webserver01 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20,path=/var/lib/libvirt/images/webserver01.qcow2 \
  --cdrom /var/lib/libvirt/images/debian-12.iso \
  --os-variant debian12 \
  --network bridge=virbr0 \
  --graphics none \
  --console pty,target_type=serial
```

*Tip:* `--graphics none` with a serial console lets you install entirely over SSH, which is the only option that makes sense on a headless server.

## Inspecting and Editing a VM's Configuration

Every VM's definition is an XML document. You rarely need to hand-edit it, but knowing how is essential for troubleshooting.

```bash
# Dump the full XML definition
virsh dumpxml webserver01

# Edit it safely, with validation before it's applied
virsh edit webserver01

# Re-apply a definition from a file
virsh define webserver01.xml
```

*Tip:* `virsh edit` opens the XML in `$EDITOR`, validates it on save, and refuses to apply a broken definition. Editing the XML file on disk directly and expecting libvirt to notice does not work.

## Storage Pools, Volumes, and qemu-img

libvirt organizes VM disks into storage pools, but the underlying files are standard QEMU disk images you can inspect directly.

```bash
# List storage pools and the volumes inside one
virsh pool-list --all
virsh vol-list default

# Create a new qcow2 disk image
qemu-img create -f qcow2 /var/lib/libvirt/images/extra-disk.qcow2 10G

# Inspect an existing image: format, size, backing file
qemu-img info /var/lib/libvirt/images/webserver01.qcow2

# Convert between formats, e.g. raw to qcow2
qemu-img convert -f raw -O qcow2 disk.img disk.qcow2
```

*Tip:* `qcow2`'s copy-on-write and snapshot support make it the right default. Use `raw` only when you need the last few percent of I/O performance and don't need snapshots.

## Networking: NAT, Bridges, and Checking What's Attached

By default, libvirt gives VMs NAT networking through a virtual bridge, which is fine for outbound access but not for exposing services directly on your LAN.

```bash
# List virtual networks and their state
virsh net-list --all

# Inspect the default NAT network
virsh net-dumpxml default

# See which bridge a VM's interface is attached to
virsh domiflist webserver01

# Attach a VM's interface to an existing LAN bridge instead of NAT
virsh attach-interface webserver01 bridge br0 --model virtio --config
```

*Tip:* If a VM needs to be reachable from other machines on your LAN, attach it to a real bridge like `br0`, not the default NAT network. NAT only forwards what you explicitly configure.

## Snapshots: Fast Rollback Before Risky Changes

Internal snapshots on qcow2 disks let you roll back a VM to a known-good state in seconds, which is invaluable before an upgrade you're not fully confident about.

```bash
# Take a snapshot
virsh snapshot-create-as webserver01 pre-upgrade "Before kernel upgrade"

# List snapshots for a VM
virsh snapshot-list webserver01

# Roll back to a snapshot
virsh snapshot-revert webserver01 pre-upgrade

# Delete a snapshot once you no longer need it
virsh snapshot-delete webserver01 pre-upgrade
```

*Tip:* Internal qcow2 snapshots are fast but live inside the same disk file, so they are not a backup. Export the disk or use `virsh dumpxml` plus a copy of the image for anything you actually need to survive a disk failure.

## Bonus: Nested Virtualization

Running a VM inside a VM is occasionally necessary for CI pipelines or testing hypervisors themselves, and it depends on the host exposing the right flag.

```bash
# Check if nested virtualization is enabled for Intel
cat /sys/module/kvm_intel/parameters/nested

# Enable it (Intel), persists until reboot
echo 1 | sudo tee /sys/module/kvm_intel/parameters/nested

# Make it persistent via a modprobe config
echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm-nested.conf
```

*Tip:* Nested virtualization carries a real performance cost. Use it for functional testing, not for anything where the inner VM's performance needs to resemble production.
