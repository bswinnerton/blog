---
layout: post
title: Configuring KVM on Clear Linux
author: Brooks Swinnerton
---

Every since stumbling on the subreddit [/r/homelab](https://www.reddit.com/r/homelab), I've been hooked on building a homelab of my own. I recently purchased two [Intel NUC](https://www.intel.com/content/www/us/en/products/boards-kits/nuc/kits/nuc7i3bnk.html) devices with the intent of setting up a [Kubernetes](https://kubernetes.io/) cluster to host my websites from my home.

In order for this project to work out I need more than two hosts, but I don't want to spend any more money on physical hardware. After reading a wonderful colleague's [blog post](https://blog.sophaskins.net/blog/setting-up-a-home-hypervisor/) on the benefits of setting up a hypervisor, I thought this would be a great way to 1) double the number of hosts available, and 2) offer some sort of DRAC-like functionality for when I'm not physically present to troubleshoot a problem. So I set out to make my two NUCs KVM hosts so that they can host virtual machines.

I decided on Clear Linux as the host OS after seeing [this](https://twitter.com/jessfraz/status/923685906481078272) tweet. This distribution has a kernel that's optimized to run on Intel hardware and it's _incredibly_ lightweight. It uses a custom package manager called `swupd` that handles no-downtime updates and categorizes themes of packages into "bundles" that can be installed. The folks who work on it can often be found in the #clearlinux room in IRC and are super nice and always willing to lend a hand.

## Installing KVM

The first step on our adventure to getting KVM set up on a Clear Linux machine is to install KVM.

We start by adding two "bundles": `kernel-kvm` and `kvm-host` using the `swupd` tool.

```
sudo swupd bundle-add kernel-kvm kvm-host
```

The `kernel-kvm` bundle installs a KVM-specific kernel, and the `kvm-host` bundle installs [these](https://github.com/clearlinux/clr-bundles/blob/456b8f473f8d97dc9b001026c4a8f63c5066e953/bundles/kvm-host) packages to help you get started with KVM.

As part of this installation process, a new Linux group has been added: `kvm`. If you're using a user account other than `root`, we add that user to the group:

```
sudo usermod -G kvm -a brooks
```

And finally, we need to enable the `libvirtd` service, which is the toolkit that we use to manage the virtualized hosts:

```
sudo systemctl enable libvirtd
```

## Configure the Network

In order for our virtual machines to receive IP addresses and be accessible from the local network, we need to create a new interface; a bridge interface.

Clear Linux uses systemd-networkd to manage persistent network configurations. We start off by creating a directory in `/etc` that networkd will automatically check to see if there are any custom specifications on boot:

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

Next, we create the bridge's `.network` file which defines how the interface should be configured. We're interested in defining how it should get an IP address:

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

And finally, we have to wire our existing physical interface to the newly created bridge (`br0`). Make sure that you use the correct `Name` value that matches the output of `ip addr`. In my case, it's `eno1`.

The naming of this file is very important. If you have any files in `/lib/systemd/network/` that have a `[Match]` stanza that targets your physical interface, you'll need to use the same filename in `/etc/systemd/network/` as I learned [here](https://unix.stackexchange.com/questions/411936/configuring-a-bridge-interface-with-systemd-networkd). This is the correct filename for a vanilla Clear Linux installation:

`/etc/systemd/network/80-dhcp.network`

```
[Match]
Name=eno1

[Network]
Bridge=br0
```

Lastly, we need to restart the network services in order for our changes to take affect. You can either use `systemctl restart systemd-networkd` or restart the computer.

Once your machine is back online, you can verify everything worked by executing `networkctl` and looking for an output similar to:

```
IDX LINK             TYPE               OPERATIONAL SETUP
  1 lo               loopback           carrier     unmanaged
  2 br0              ether              routable    configuring
  3 wlp58s0          wlan               no-carrier  configuring
  4 eno1             ether              carrier     configuring

4 links listed.
```

We hope to see `br0` as "routable", and don't have to worry about a setup of `configuring`.

## Build VMs

Now for the exciting part! Building and launching the virtual machines.

First, let's start with creating a directory to store the ISOs that we'll use for the installation media:

```
sudo mkdir /var/lib/libvirt/isos/
sudo chown root:kvm /var/lib/libvirt/isos/
sudo chmod g+rwx /var/lib/libvirt/isos/
```

Next, let's create another directory to store the virtual machine images:

```
sudo mkdir /var/lib/libvirt/images/
sudo chown root:kvm /var/lib/libvirt/images/
sudo chmod g+rwx /var/lib/libvirt/images/
```

The following step depends on which distribution of Linux that you would like to install. I opted for Debian, but just about anything would work here. We start by fetching the ISO from the internet:

```
wget -P /var/lib/libvirt/isos/ https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.3.0-amd64-netinst.iso
```

Now we need to create a new disk image for the virtual machine to use. Here I'm creating a 10GB image using the `qcow2` format which supports overlays.

```
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/debian.qcow2 10G
```

Next, we need to write a configuration file outlining the VM that we would like to create. In other Linux distributions that have a GUI, you'll commonly see people use `virt-install`, but unfortunately that's not available to us in Clear Linux, so we need to craft the file ourselves.

`/var/lib/libvirt/debian.xml`

```xml
<domain type='kvm'>
  <name>debian</name>
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
      <source file='/var/lib/libvirt/images/jupiter.qcow2'/>
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

The TL;DR of this file is that we're creating a virtual machine named "debian" with 3GB of RAM, 2 CPUs, and three devices mounted: a NIC which will use the host's bridge interface, a CD drive to boot the ISO, and the disk that we just created. Finally, I've configured the graphics to use VNC since there's no GUI on our installation of Clear Linux, so we'll need to access the VM remotely from another machine.

If you're using a Mac, it's important to set a `passwd` value for the `<graphics />` tag as the built-in VNC client requires a password to work properly.

If you plan on creating other VMs, you'll want to generate a different [UUID](https://www.uuidgenerator.net/) and [MAC address](http://www.centos.org/docs/5/html/5.2/Virtualization/sect-Virtualization-Tips_and_tricks-Generating_a_new_unique_MAC_address.html) for each subsequent file.

## Start VM

Now we're ready to start the virtual machine. First, we define the virtual machine based on the XML file (which will keep the system persisted after reboots) and then we turn it on.

```
sudo virsh define /var/lib/libvirt/debian.xml
sudo virsh start debian
```

Finally, we can configure the machine to start automatically when the host turns on:

```
sudo virsh autostart debian
```

## Connect to VM

In order to connect to the machine, we'll be using VNC from our own computer. In my case, I'm using OSX, so I'll be using the built in VNC client.

In order to see which port the guest is utilizing for KVM, we can run:

```
sudo virsh vncdisplay debian
```

And we should see similar output to:

```
127.0.0.1:0
```

The `:0` there is significant. It implies that the machine is listening over VNC on port `5900`. If the value was `127.0.0.1:1`, it would imply that the machine is listening over port `5901`.

We can create a reverse SSH tunnel to the KVM host so that any requests to localhost's port `5901`, it'll actually go to the virtual machine's port `5900`:

```
ssh -L 5901:localhost:5900 -N user@kvmhost
```

Next, in Finder, we open up the "Connect to Server" prompt with <kbd>Command</kbd> + <kbd>k</kbd>, and type in the address to our tunneled connection : `vnc://localhost:5901`.

And voila! You should now see the installation media on your virtual machine.

## Post Installation

Once your installation is complete, you'll need to shut down the virtual machine:

```
sudo virsh shutdown debian
```

And then adjust the XML file so that the boot device is pointed to the hard drive since we no longer need the installer.

We'll need to modify this line:

```xml
<boot dev='cdrom'/>
```

To:

```xml
<boot dev='hd'/>
```

And then start your machine back up with:

```
sudo virsh start debian
```

## Creating a Template for Future Images

If you plan on making more virtual machines, instead of installing the OS from the installation media each time, you can instead create an "template image".

What's really neat about the `qcow2` format is that you can specify the "backing" image. Then, in the image new _non_-template image, it only saves the _difference_ from the template, and more than one machine can share the same base image.

To create the base image, start by powering off your virtual machines:

```
sudo virsh stop debian
```

And then rename the debian image to something that implies it's template:

```
sudo mv /var/lib/libvirtd/images/debian.qcow2 /var/lib/libvirtd/images/template.qcow2
```

And then create a new image which uses the template as a backing image:

```
qemu-img create -f qcow2 -b template.qcow2 debian.qcow2
```

And then start your machine:

```
sudo virsh start debian
```

You'll notice that the `debian.qcow2` image is quite small, that's because it's overlayed on top of the template.

If you wanted to make a new virtual machine, all that you would need to do is create the new XML file (with a different UUID and MAC address), and then create a new image with the template as its base:

```
qemu-img create -f qcow2 -b template.qcow2 debian2.qcow2
```

And you won't have to worry about reinstalling the OS. Oh! And `qcow2` images can be layered on top of one another!

That's all for this blog post. Stay tuned for the next step in creating a Kubernetes cluster on these VMs ✌️ .
