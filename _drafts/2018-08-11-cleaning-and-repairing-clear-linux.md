---
layout: post
title: Cleaning and repairing Clear Linux
author: Brooks Swinnerton
---

```
11:15 < arjan_work> if you ever want to clean up stuff you added to /usr that way
11:15 < arjan_work> swupd verify --fix --picky
11:15 < arjan_work> will remove all files from /usr that the OS itself did not put there
11:16 < arjan_work> and get you back into a clean state
11:16 < bswinnerton> Oh wow. I hope I didn't put anything in there xD
11:16 < bswinnerton> Is there a command to check if I have, before cleaning?
11:18 < arjan_work> yeah without --fix iirc but let me double check
11:19 < arjan_work>    -Y, --picky             List (without --fix) or remove (with --fix) files which should not exist
11:19 < bswinnerton> Awesome, that's handy.
11:19 < bswinnerton> Thank you
11:20 < bswinnerton> Ohhh, interesitng. Yes I was fiddling around with udev rules last night.
11:20 < bswinnerton> I have a file that I created: /usr/lib/udev/rules.d/90-libvirt-usb-awus036ach.rules, that would be removed. Where's a more suitable place to put
                     that?
11:20 < bswinnerton> (it's meant to auto-reattach a USB device to libvirt)
11:22 < arjan_work> I think you can stick your own rules in /etc/udev/*
11:22 < arjan_work> overall we try to keep OS configs from user configs by letting you own /etc and the OS own /usr
11:24 < bswinnerton> That makes sense. My Linux foo isn't super great, so I wasn't aware of that distincition. I had seen on the internet references to udev rules in
                     /etc/, but I didn't see the directory so thought it might just be different in Clear Linux. I'll try creating it and see if it still loads
                     properly.
11:24 < arjan_work> swupd verify --fix (without --picky) will repair all files you modified, but will not remove extra ones
11:24 < bswinnerton> Oh cool.
11:24 < arjan_work> so if you ever break something by accident you can repair it
```
