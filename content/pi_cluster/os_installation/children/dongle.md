---
layout: post
title:  "Create a Rapsberry PI Rescue Dongle"
menuTitle: "Rescue Dongle"
weight: 20 
date:   2018-06-20
categories: [wiki]
description: ""
thumbnail: "img/placeholder.jpg"
disable_comments: true
authorbox: true
toc: true
mathjax: true
tags: [rpi]
published: true
---

I encountered multiple issues trying to repartition my SD on my PI.
Because the / directory is mounted, it never really worked safely for me 
to use fdisk. Morevoer some of the powerfull tools such as gparted need
X11 installed, which I don't have by default.

Hopefully the new PI3 B and B+ are able to boot from USB, hence the idea of creating a Rescue Dongle

<!--more-->

## Consideration regarding USB boot.

It seems that all new PI 3B+ have OTP for USB boot mode setup by default.
For the PC 3B, you have to activate using the /boot/config.txt

master-pi is a 3B+, nas-pi and home-pi are 3B:

Let's check the /boot/config.txt
~~~
ansible picluster -i inventory/ -m shell -a "grep program_usb_boot_mode /boot/config.txt"

master-pi.kubedge.cloud | FAILED | rc=1 >>


nas-pi.kubedge.cloud | SUCCESS | rc=0 >>
program_usb_boot_mode=1

home-pi.kubedge.cloud | FAILED | rc=1 >>
~~~

The flag for OTP USB flag is set on the 3B+ (by default on master-pi) and on the 3B where I did add the entry to the config.txt (nas-pi) 
~~~
ansible picluster -i inventory/ -m shell -a "vcgencmd otp_dump | grep 17:"

master-pi.kubedge.cloud | SUCCESS | rc=0 >>
17:3020000a

nas-pi.kubedge.cloud | SUCCESS | rc=0 >>
17:3020000a

home-pi.kubedge.cloud | SUCCESS | rc=0 >>
17:1020000a
~~~

## Creation of the Rescue Dongle

- Flash Raspbian on the Dongle using the normal procedure (Wind32DiskImager,..)
- Remove the SD card from the PI3.
- Plug keyboard, mouse, screen ... onto the PI
- Boot the PI3 on the Dongle by removing the SD card
- Take the time to setup the Raspbian:
  + Enable VNC
  + Enable SSH
  + Setup default hostname
  + Setup password for Pi account
  + Setup resolution (for VNC later)
  + Install tools such as gparted
- Shutdown the PI and put back the normal SD card.

## Usage

The idea is to boot a PI from the Dongle and then apply the fixes to the SD card.

- Shutdown the PI.
- Remove the SD card.
- Insert the Dongle into USB port of PI
- Reboot the PI.
- Connect to the PI using the VNC or SSH. The PI has started from the OS installed on the Dongle.
- Insert the SD card into the PI. You will most likely have a popup and the filesystem from the SD card is automatically mounted.
- Start to use the tools to fix your SD card. For instance: ![](/images/rescuepi/rescuing_sd.png)
- Shutdown and reboot. The PI will restart from the SD card.

## Application: Change SD card partition

First step is to reboot a PI without SD card from the Dongle and connect VNC Viewer to it.

First insert the SD and close the popups
![](/images/rescuepi/insert_sd.png)

In a terminal run, sudo gparted
~~~
sudo gparted

unmount the mmc root partition
~~~
![](/images/rescuepi/unmount_partition.png)

~~~
Resize the current partition (down to 16G), create an extended one and 7*2G logical partition in that 14G partition)
~~~
![](/images/rescuepi/create_partitions.png)

~~~
Apply the changes
~~~
![](/images/rescuepi/applying_changes.png)

~~~
Shutdown or Reboot. The PI should restart from the SD anyway.
~~~
![](/images/rescuepi/shutdown.png)

### Reference Links

- TBD

