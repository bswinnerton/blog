---
layout: post
title: Configuring KVM on Clear Linux
author: Brooks Swinnerton
---

Every since stumbling on the subreddit [/r/homelab](https://www.reddit.com/r/homelab), I've been hooked on building a homelab of my own. I recently purchased two [Intel NUC](https://www.intel.com/content/www/us/en/products/boards-kits/nuc/kits/nuc7i3bnk.html) devices with the intent of setting up a [Kubernetes](https://kubernetes.io/) cluster to host my websites from my home.

In order for this project to work out I need more than two hosts, but I don't want to spend any more money on physical hardware. After reading a wonderful colleague's [blog post](https://blog.sophaskins.net/blog/setting-up-a-home-hypervisor/) on the benefits of setting up a hypervisor, I thought this would be a great way to 1) double the number of hosts available, and 2) offer some sort of DRAC-like functionality for when I'm not physically present to troubleshoot a problem.

This is a blog post on how to use the [Clear Linux](https://clearlinux.org/) project to create a KVM host.

If this is your first time hearing about Clear Linux, it's an extremely lightweight distribution of Linux that has its own package manager (dubbed `swupd`) and strikes a great balance of minimalism but usefulness. Not to mention the folks who develop it are almost always around in the #clearlinux IRC channel and always willing to lend a hand.

## Installing KVM

The first step on our adventure to getting KVM set up on a Clear Linux machine is to install KVM.

We start by adding two "bundles": `kernel-kvm` and `kvm-host` using the `swupd` tool.

```
sudo swupd bundle-add kernel-kvm kvm-host
```

The `kernel-kvm` bundle installs a KVM-specific kernel, and the `kvm-host` bundle installs [these](https://github.com/clearlinux/clr-bundles/blob/456b8f473f8d97dc9b001026c4a8f63c5066e953/bundles/kvm-host) packages to help you get started with KVM.

As part of this installation process, a new Linux group has been added: `kvm`. If you're using a user account other than `root`, we need to add that user to the group:

```
sudo usermod -G kvm -a brooks
```

And finally, we start need to enable the `libvirtd` service, which is the toolkit that we use to manage our virtualized hosts:

```
sudo systemctl enable libvirtd
```

## Configure the Network

In order for our virtual machines to receive IP addresses and be accessible from the local network, we need to create a new interface; a bridge interface.

Clear Linux uses systemd-networkd to manage persistent network configurations. We need to start off by creating a folder that networkd will automatically look to to see if there are any custom specifications:

```
sudo mkdir /etc/systemd/network/
```

First, we create a `.netdev` file which defines the configuration of the new network device. Use your favorite editor to edit the file with the contents below:

`/etc/systemd/network/br0.netdev`

```
[NetDev]
Name=br0
Kind=bridge
```

Next, we create the bridge's `.network` file which defines how the interface should get an IP address:

`/etc/systemd/network/br0.network`

```
[Match]
Name=br0

[Network]
DHCP=yes
```

I've opted to use DHCP since my router has the ability to reserve addresses based on MAC addresses, but you could also add something similar to this:

```
[Network]
Address=10.0.0.2/16
Gateway=10.0.0.1
DNS=10.0.0.1
DNS=8.8.8.8
```

And finally, we have to wire our existing physical interface to the newly created bridge (`br0`). Make sure that you use the correct `Name` value that matches the output of `ip addr`.

The naming of this file is very important. If you have any files in `/lib/systemd/network/` that have a `[Match]` stanza that targets your physical interface, you'll need to use the same filename in `/etc/systemd/network/` as I learned [here](https://unix.stackexchange.com/questions/411936/configuring-a-bridge-interface-with-systemd-networkd). This is the correct filename for a vanilla Clear Linux installation:

`/etc/systemd/network/80-dhcp.network`

```
[Match]
Name=eno1

[Network]
Bridge=br0
```

Lastly, now is a great time reboot your computer and ensure that the bridge interface is properly configured.

Once your machine is back online, you can verify everything worked by executing `networkctl` and looking for an output similar to:

```
IDX LINK             TYPE               OPERATIONAL SETUP
  1 lo               loopback           carrier     unmanaged
  2 br0              ether              routable    configuring
  3 wlp58s0          wlan               no-carrier  configuring
  4 eno1             ether              carrier     configuring

4 links listed.
```

We hope to see `br0` as routable, and don't have to worry about a setup of `configuring`.

## Build VMs

Now for the exciting part! Building and launching the virtual machines.

First, let's start with creating a folder to store the ISOs that we'll use as the installation media:

```
sudo mkdir /var/lib/libvirt/isos/
sudo chown root:kvm /var/lib/libvirt/isos/
sudo chmod g+rwx /var/lib/libvirt/isos/
```

Next, let's do the same to store the actual virtual machine images:

```
sudo mkdir /var/lib/libvirt/images/
sudo chown root:kvm /var/lib/libvirt/images/
sudo chmod g+rwx /var/lib/libvirt/images/
```



```
wget -P /var/lib/libvirt/isos/ https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.3.0-amd64-netinst.iso

sudo qemu-img create -f qcow2 /var/lib/libvirt/images/debian.img 10G
```

`/var/lib/libvirt/ubuntu.xml`

```xml
<domain type='kvm'>
  <name>jupiter</name>
  <uuid>bee8e041-ddf2-4772-ba18-8a4998cd8d83</uuid>
  <memory>3145728</memory>
  <currentMemory>3145728</currentMemory>
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
  <on_poweroff>preserve</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <disk type='file' device='disk'>
      <source file='/var/lib/libvirt/images/jupiter.img'/>
      <target dev='hda' bus='ide'/>
    </disk>
    <disk type='file' device='cdrom'>
      <source file='/var/lib/libvirt/isos/debian-9.3.0-amd64-netinst.iso'/>
      <target dev='hdb' bus='ide'/>
      <readonly/>
    </disk>
    <interface type='bridge'>
      <source bridge='br0'/>
      <mac address="00:16:3e:03:2e:5a"/>
    </interface>
    <graphics type='vnc' port='-1' autoport='yes' passwd='correcthorsebatterystaple'/>
  </devices>
</domain>
```

It's important to set a `passwd` value for the `<graphics />` tag as the OSX VNC client requires a password to work properly.

http://www.centos.org/docs/5/html/5.2/Virtualization/sect-Virtualization-Tips_and_tricks-Generating_a_new_unique_MAC_address.html
https://dustymabe.com/2015/01/11/qemu-img-backing-files-a-poor-mans-snapshotrollback/

## Start VM

```
sudo virsh define /var/lib/libvirt/jupiter.xml
sudo virsh start jupiter
```

Heads up: Using `sudo virsh start` causes the VM to be destroyed on shutdown

## Connect to VM

```
ssh -L 5901:localhost:5900 -N brooks@nuc7i3.brooks.network
```

<kbd>Command</kbd> + <kbd>k</kbd>: `vnc://localhost:5901`

sudo virsh autostart saturn
