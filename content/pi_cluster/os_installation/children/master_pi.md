---
layout: post
title:  "OS Installation on Master PI"
menuTitle: "OS on Master PI"
weight: 10
date:   2018-07-07
categories: [wiki]
description: ""
thumbnail: "img/placeholder.jpg"
disable_comments: true
authorbox: true
toc: true
mathjax: true
tags: [ansible, hypriot, rpi]
published: true
---

This page has been updated to use Ubuntu Server 22.04.02 LTS 64 bits. The instruction used be based on hypriotos

<!--more-->

## Overview

Kubedge uses Ubuntu 64 because the quickest to set up. SSH is enabled by default when using the imager. You don't need to plug a screen or a keyboard to the PI.

{{% notice note %}}
Raspberry Pi OS is now available in 32 and 64 bits. Ubuntu 22.04 LTS is also available.
Even so the Raspberry PI 3B have only 1G of RAM, picking the 64 bits architecture provide wider choice of software.
{{% /notice %}}

- The 2018 procedure called for deploying [HypriotOS ARM32V7](https://github.com/hypriot/image-builder-rpi/releases/download/v1.9.0/hypriotos-rpi-v1.9.0.img.zip) and [HypriotOS ARM64V8](https://github.com/DieterReuter/image-builder-rpi64/releases/download/v20180429-184538/hypriotos-rpi64-v20180429-184538.img.zip). Flashing all 3 or 5 SD cards used to be done **Win32DiskImager** or similar. It takes around 30 seconds per card.

- The updated procedure is based on Rapsberry Pi Imager v1.7.3 and the Ubuntu Server 22.04.02 LTS (64 BIT)

![](/images/ubuntu64/Screenshot_2023-02-26_103134.jpg)
![](/images/ubuntu64/Screenshot_2023-02-26_104230.jpg)
![](/images/ubuntu64/Screenshot_2023-02-26_104252.jpg)

1. Set Hostname to `master-pi`
2. Enable SSH
    * Use password authentication (will update the procedure to use key later)
3. Set username and password
    * Username: kubedge
    * Password: hypriot
4. Configure WLAN
    * SSSID: yourssid
    * password: yourssidpwd
    * wireless LAN country: US
5. Set local settings
    * America/Chicago
    * us


## OS on master on master PI

{{% notice warning %}}
Do not power up the slaves nodes for right now. Either plug your home router directly to the master PI, or to the switch.
The goal here is to get an IP address allocated by your home router to the master PI.
{{% /notice %}}

### Access the node

- Insert the SD card in the master PI.
- Access your home router and look for "master-pi" IP address.
- SSH to the node using **wsl**, **visual-studio-code**  or **moba-xterm** for instance. The credentials are kubedge/hypriot.

### Freeze your configuration

Cloud init is perfect for the first boot. Once the node
is up, it can be challenging not to preserve the fine tuning done
to the OS.

```bash
sudo apt-get remove --purge cloud-init
sudo apt-get autoremove
```

### Update OS

Let's update to the latest version

```bash
sudo apt-get update
sudo apt-get upgrade
```

Also the latest version of kubernetes is not rely on docker, there is a need to installed containerd.
The version of containerd provided at this time with ubuntu does not seem to work properly.
There is a need in install containerd.io

```bash
sudo -i

curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
usermod -aG docker pirate
```



### Update master PI name

#### 2018 procedure

As root, replace **black-pearl** by **kubemaster-pi**, in the two following files:

```bash
sudo -i

vi /etc/hosts
```

It seems on HypriotOS 64, you need to do

```bash
sudo hostnamectl set-hostname kubemaster-pi
```

#### 2013 procedure

Looks like the Raspberri PI imager create a cloud-init and set up the proper name 

### Configure the main admin account

If you prefer to edit using vi instead of nano

```bash
vi /etc/environment

EDITOR=vi
```
Then you need to set up the ssh keys
Either create a new ssh key (id_rsa, id_rsa.pub) and copy in id_rsa.pub into [GitHub](https://github.com/settings/keys)
You also need to add those keys in [GerritHub](https://review.gerrithub.io/#/settings/ssh-key)

#### 2018 procecdure

```bash
ssh-keygen
```
or install your private and public key into the **/home/pirate/.ssh** directory (The same key you registered in GitHub)

```bash
ssh-copy-id -i hypriotos_rsa pirate@<MASTERPI>
scp hypriotos_rsa pirate@<MASTERPI>:/home/pirate/.ssh/id_rsa
scp hypriotos_rsa.pub pirate@<MASTERPI>:/home/pirate/.ssh/id_rsa.pub
```

#### 2018 procecdure

```bash
ssh-keygen
```
or install your private and public key into the **/home/kubedge/.ssh** directory (The same key you registered in GitHub)

```bash
ssh-copy-id -i kubedge_rsa kubedge@<MASTERPI>
scp kubedge_rsa kubedge@<MASTERPI>:/home/kubedge/.ssh/id_rsa
scp kubedge_rsa.pub kubedge@<MASTERPI>:/home/kubedge/.ssh/id_rsa.pub
```

### Install GIT

Since you have ethernet access, it is a good time to install GIT in order to download usefull scripts from github.com
GIT is installed by default on Ubuntu 64 bits

```bash
sudo apt-get update
sudo apt-get install git
sudo apt-get install git-review
```

```
git config --global user.email <youremail>
git config --global user.name <yourname>
```

```bash
$ mkdir -p $HOME/proj/kubedge
$ cd $HOME/proj/kubedge
```

create a cloneit1.sh file, edit it and run it

{{% notice info %}}
If you deployed on ARM64, be sure to replace **arm32v7** by **arm64v8**
{{% /notice %}}


```bash
#!/bin/bash
GOODPATH=`pwd`
USERID=yourgithubaccount
for i in helmrepos kubeplay kubesim_5gc kubesim_base kubesim_elte kubesim_epc kubesim_lte kubesim_nr kubesim_blinkt kubesim_nats kubesim_linkio kubedge_utils
do
echo "======================================================"
echo $i
echo "======================================================"
# If you want to contribute
# git clone -b arm32v7 ssh://$USERID@review.gerrithub.io:29418/kubedge/$i && scp -p -P 29418 $USERID@review.gerrithub.io:hooks/commit-msg $i/.git/hooks/
# If you want to just pull the code
# git clone -b arm32v7 git@github.com:kubedge/$i.git
cd $GOODPATH
done
```

create a cloneit2.sh file, edit it and run it

``` bash
#!/bin/bash
GOODPATH=`pwd`
USERID=yourgithubaccount
for i in kube-rpi ansible-kube-rpi
do
echo "======================================================"
echo $i
echo "======================================================"
# If you want to contribute
# git clone ssh://$USERID@review.gerrithub.io:29418/kubedge/$i && scp -p -P 29418 $USERID@review.gerrithub.io:hooks/commit-msg $i/.git/hooks/
# If you want to just pull the code
# git clone git@github.com:kubedge/$i.git
cd $GOODPATH
done
```

{{% notice info %}}
You know have access especially through **kube-rpi** and **ansible-kube-rpi** to example files and ansible playbook usefull to install your first node:
**cd $HOME/proj/kubedge/kube-rpi/config/cluster1/hypriotos/kubemaster-pi**
{{% /notice %}}

## Reference Links

- [HypriotOS](https://github.com/hypriot/image-builder-rpi/releases)
- [Video2](https://www.youtube.com/watch?v=eZ5uX-JJbyY)

