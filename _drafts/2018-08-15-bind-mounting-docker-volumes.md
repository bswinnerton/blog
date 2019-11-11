---
layout: post
title: USB hotplug support in libvirt
author: Brooks Swinnerton
---

Due to a `ln -s` for `/var/lib/docker/volumes` pointing to `/data/docker/volumes`, I was receiving:

```
brooks@nuc7i5 ~ $ sudo docker volume create test
test
brooks@nuc7i5 ~ $ sudo docker volume rm test
Error response from daemon: unable to remove volume: remove test: Unable to remove a directory of out the Docker root /var/lib/docker: /data/docker/volumes/test/_data
```

In order to resolve this, we can use the `mount` utility to transparently mount the directory:

```
brooks@nuc7i5 ~ $ sudo mkdir /var/lib/docker/volumes/
brooks@nuc7i5 ~ $ sudo mount -o bind /data/docker/volumes/ /var/lib/docker/volumes/
```

Or in `/etc/fstab`:

```
/data/docker/volumes /var/lib/docker/volumes none bind
```
