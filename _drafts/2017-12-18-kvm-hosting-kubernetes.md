---
layout: post
title: Using Clear Linux as a KVM Host
author: Brooks Swinnerton
---

Introduction to KVM

Introduction to Clear Linux

## Install KVM

```
sudo swupd bundle-add kernel-kvm
sudo swupd bundle-add kvm-host

sudo adduser brooks kvm
sudo usermod -G kvm -a brooks

sudo systemctl enable libvirtd
```

## Configure Network

https://major.io/2015/03/26/creating-a-bridge-for-virtual-machines-using-systemd-networkd/

`/etc/systemd/network/br0.netdev`

```
[NetDev]
Name=br0
Kind=bridge
```

`/etc/systemd/network/br0.network`

```
[Match]
Name=br0

[Network]
DHCP=yes
```

`/etc/systemd/network/80-dhcp.network`

```
[Match]
Name=eno1

[Network]
Bridge=br0
```

## Build VMs

```
sudo mkdir /var/lib/libvirt/isos
sudo chown root:kvm /var/lib/libvirt/isos/
sudo chmod g+rwx /var/lib/libvirt/isos/

sudo mkdir /var/lib/libvirt/images
sudo chown root:kvm /var/lib/libvirt/images/
sudo chmod g+rwx /var/lib/libvirt/images/

wget -P /var/lib/libvirt/isos/ http://releases.ubuntu.com/16.04.3/ubuntu-16.04.3-desktop-amd64.iso

sudo qemu-img create -f qcow2 /var/lib/libvirt/images/ubuntu.img 10G
```

`/var/lib/libvirt/ubuntu.xml`

```xml
<domain type='kvm'>
  <name>ubuntu</name>
  <uuid>85badf15-244d-4719-a2da-8c3de0641373</uuid>
  <memory>1677721</memory>
  <currentMemory>1677721</currentMemory>
  <vcpu>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc'>hvm</type>
    <boot dev='cdrom'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <pae/>
  </features>
  <clock offset='localtime'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>destroy</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <disk type='file' device='disk'>
      <source file='/var/lib/libvirt/images/ubuntu.img'/>
      <target dev='hda' bus='ide'/>
    </disk>
    <disk type='file' device='cdrom'>
      <source file='/var/lib/libvirt/isos/ubuntu-16.04.3-desktop-amd64.iso'/>
      <target dev='hdb' bus='ide'/>
      <readonly/>
    </disk>
    <!--<interface type='bridge'>
      <source bridge='br0'/>
      <mac address="0E:1F:35:AB:45:0C"/>
    </interface>-->
    <graphics type='vnc' port='5900' autoport='yes' passwd='virtualmachinesarecool'/>
  </devices>
</domain>
```

It's important to set a `passwd` value for the `<graphics />` tag as the OSX VNC client requires a password to work properly.

## Start VM

```
sudo virsh create ubuntu.xml
sudo virsh start ubuntu
```

## Connect to VM

```
ssh -L 5901:localhost:5900 -N brooks@nuc7i3.brooks.network
```

<kbd>Command</kbd> + <kbd>k</kbd>: `vnc://localhost:5901`
