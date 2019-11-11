---
layout: post
title: USB hotplug support in libvirt
author: Brooks Swinnerton
---

### Create the new disks

```
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/kube-node-a1-gluster.qcow2 75G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/kube-node-a2-gluster.qcow2 75G

sudo qemu-img create -f qcow2 /var/lib/libvirt/images/kube-node-b1-gluster.qcow2 75G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/kube-node-b2-gluster.qcow2 75G
```

### Redefine the `virsh` configs to use the new disks

```
sudo virsh shutdown kube-node-a1
sudo virsh shutdown kube-node-a2
sudo virsh undefine kube-node-a1
sudo virsh undefine kube-node-a2

sudo virsh shutdown kube-node-b1
sudo virsh shutdown kube-node-b2
sudo virsh undefine kube-node-b1
sudo virsh undefine kube-node-b2

cd /data/home-lab/kvms/hosts/
sudo virsh define kube-node-a1.xml
sudo virsh define kube-node-a2.xml
sudo virsh autostart kube-node-a1
sudo virsh autostart kube-node-a2

cd /data/home-lab/kvms/hosts/
sudo virsh define kube-node-a1.xml
sudo virsh define kube-node-a2.xml
sudo virsh autostart kube-node-a1
sudo virsh autostart kube-node-a2
```

### Enable the right kernel modules

```

```
