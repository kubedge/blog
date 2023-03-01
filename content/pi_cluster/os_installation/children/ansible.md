---
layout: post
title:  "Using Ansible to manage Raspberry PI cluster"
menuTitle: "Ansible"
weight: 30
date:   2018-07-07
categories: [wiki]
description: ""
thumbnail: "img/placeholder.jpg"
disable_comments: true
authorbox: true
toc: true
mathjax: true
tags: [ansible, rpi]
published: true
---

Even if the ultimate goal is to manage completly the cluster using Kubernetes,
the ability to use Ansible during debug process is very usefull.
The goal here is to setup ansible inventory, basic playbooks.

<!--more-->

![](/images/hack4easy/ansible.png)

## Ansible Installation on the master node

Let's install ansible-core using apt-get. To reduce the footprint, on the PI we only deploy ansible-core.
We will leverage ansible-galaxy to install additional collections if needed

```bash
sudo apt install ansible-core
```

~~~
$ ansible --version

ansible 2.2.1.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = Default w/o overrides
~~~

~~~
sudo apt-get install sshpass

Reading package lists... Done
Building dependency tree
Reading state information... Done
The following NEW packages will be installed:
  sshpass
0 upgraded, 1 newly installed, 0 to remove and 6 not upgraded.
Need to get 11.2 kB of archives.
After this operation, 30.7 kB of additional disk space will be used.
Get:1 http://raspbian.mirrors.lucidnetworks.net/raspbian stretch/main armhf sshpass armhf 1.06-1 [11.2 kB]
Fetched 11.2 kB in 2s (4,785 B/s)
Selecting previously unselected package sshpass.
(Reading database ... 26786 files and directories currently installed.)
Preparing to unpack .../sshpass_1.06-1_armhf.deb ...
Unpacking sshpass (1.06-1) ...
Setting up sshpass (1.06-1) ...
Processing triggers for man-db (2.7.6.1-2) ...
~~~

## Configure ansible

Create directorties for ansible
~~~
mkdir -p mgt/inventory
mkdir -p mgt/playbooks
mkdir -p mgt/roles
mkdir -p mgt/group_vars
mkdir -p mgt/files
mkdir -p mgt/inventory/host_vars
~~~

Let's check the internal cluster network
~~~
cat /etc/hosts

192.168.2.1 kubemaster-pi.clusterX.kubedge.cloud kubemaster-pi
192.168.2.101 kube-node01.clusterX.kubedge.cloud kube-node01
192.168.2.102 kube-node02.clusterX.kubedge.cloud kube-node02
192.168.2.103 kube-node03.clusterX.kubedge.cloud kube-node03
192.168.2.104 kube-node04.clusterX.kubedge.cloud kube-node04
~~~

Let's create an rsa key for Ansible SSH. Note
the cluster is still using the default kubedge account
created by Ubuntu. Will change is later
once ansible is up.
~~~
cd ~/mgt

ssh-keygen -t rsa -f mgtkey

ssh-copy-id -i mgtkey.pub kubedge@kubemaster-pi
ssh-copy-id -i mgtkey.pub kubedge@kube-node01
ssh-copy-id -i mgtkey.pub kubedge@kube-node02
ssh-copy-id -i mgtkey.pub kubedge@kube-node03
ssh-copy-id -i mgtkey.pub kubedge@kube-node04
~~~

Create a first ansible host_var. We will use the mgtkey for ssh/ansible.
~~~
cd ~/mgt/inventory/host_vars
cat kubemaster-pi.clusterX.kubedge.cloud

ansible_host: 192.168.2.FOOBAR
ansible_port: 22
ansible_user: kubedge
ansible_ssh_private_key_file: mgtkey
~~~

~~~
for i in kube-node01 kube-node02 kube-node03 kube-node04
do
cp kubemaster-pi.clusterX.kubedge.cloud $i.clusterX.kubedge.cloud
done
~~~

Replace FOOBAR by the proper value
~~~
vi *.clusterX.kubedge.cloud
~~~

Create the main inventory file
~~~
cd ~/mgt/inventory
cat hosts

---
[picluster:children]
masters
workers

[masters]
kubemaster-pi.clusterX.kubedge.cloud

[workers]
kube-node01.clusterX.kubedge.cloud
kube-node02.clusterX.kubedge.cloud
kube-node03.clusterX.kubedge.cloud
kube-node04.clusterX.kubedge.cloud
~~~

### Use ansible

#### Simple ping

~~~
ansible picluster -i inventory/ -m ping

kube-node01.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
kube-node04.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
kube-node03.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
kubemaster-pi.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
kube-node02.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
~~~

~~~
ansible masters -i inventory/ -m ping

kubemaster-pi.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
~~~

~~~
ansible workers -i inventory/ -m ping

kube-node01.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
kube-node03.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
kube-node02.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
kube-node04.clusterX.kubedge.cloud | SUCCESS => {
    "changed": false,
    "ping": "pong"
~~~

#### Retrieve Facts

~~~
ansible picluster -i inventory/ -m setup
~~~

#### Update all the nodes in the cluster

First we need to add docker and kubernetes repos

~~~
cat playbooks/installkeys.yml
~~~

```yaml
---
- hosts: picluster
  tasks:
  - name: Add docker signing key (new GPG method)
    become: true
    ansible.builtin.get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'
        force: true
  - name: Add docker repository
    become: true
    ansible.builtin.apt_repository:
       repo: deb [signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu jammy stable
       state: present
       update_cache: yes
  - name: Add kubernetes signing key (new GPG method)
    become: true
    ansible.builtin.get_url:
        url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
        dest: /etc/apt/keyrings/kubernetes-archive-keyring.gpg 
        mode: '0644'
        force: true
  - name: Add kubernetes repository
    become: true
    ansible.builtin.apt_repository:
       repo: deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main
       state: present
       update_cache: yes
```

~~~
cat playbooks/aptupdate.yml
~~~

```yaml
---
- hosts: picluster
  tasks:
  - name: update apt
    become: true
    apt:
       update_cache: yes
~~~
```

Run the simple aptupdate playbook
~~~
ansible-playbook -i inventory/ playbooks/aptupdate.yml

PLAY [picluster] ***************************************************************

TASK [setup] *******************************************************************
ok: [kubemaster-pi.clusterX.kubedge.cloud]
ok: [kube-node03.clusterX.kubedge.cloud]
ok: [kube-node04.clusterX.kubedge.cloud]
ok: [kube-node02.clusterX.kubedge.cloud]
ok: [kube-node01.clusterX.kubedge.cloud]

TASK [update apt] **************************************************************
changed: [kube-node02.clusterX.kubedge.cloud]
changed: [kubemaster-pi.clusterX.kubedge.cloud]
changed: [kube-node01.clusterX.kubedge.cloud]
changed: [kube-node03.clusterX.kubedge.cloud]
changed: [kube-node04.clusterX.kubedge.cloud]

PLAY RECAP *********************************************************************
kube-node01.clusterX.kubedge.cloud         : ok=2    changed=1    unreachable=0    failed=0
kube-node02.clusterX.kubedge.cloud         : ok=2    changed=1    unreachable=0    failed=0
kube-node03.clusterX.kubedge.cloud         : ok=2    changed=1    unreachable=0    failed=0
kube-node04.clusterX.kubedge.cloud         : ok=2    changed=1    unreachable=0    failed=0
kubemaster-pi.clusterX.kubedge.cloud       : ok=2    changed=1    unreachable=0    failed=0
~~~

#### Check the version  of kubeadm

~~~
ansible picluster -i inventory -m shell -a "kubeadm version"

kube-node04.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.8", GitCommit:"c138b85178156011dc934c2c9f4837476876fb07", GitTreeState:"clean", BuildDate:"2018-05-21T18:53:18Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/arm"}

kube-node03.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.8", GitCommit:"c138b85178156011dc934c2c9f4837476876fb07", GitTreeState:"clean", BuildDate:"2018-05-21T18:53:18Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/arm"}

kube-node01.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.8", GitCommit:"c138b85178156011dc934c2c9f4837476876fb07", GitTreeState:"clean", BuildDate:"2018-05-21T18:53:18Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/arm"}

kubemaster-pi.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.8", GitCommit:"c138b85178156011dc934c2c9f4837476876fb07", GitTreeState:"clean", BuildDate:"2018-05-21T18:53:18Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/arm"}

kube-node02.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.8", GitCommit:"c138b85178156011dc934c2c9f4837476876fb07", GitTreeState:"clean", BuildDate:"2018-05-21T18:53:18Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/arm"}
~~~

#### Check the temperature

~~~
ansible picluster -i inventory -m shell -a "vcgencmd measure_temp"

kube-node03.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
temp=36.5'C

kubemaster-pi.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
temp=49.4'C

kube-node02.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
temp=33.2'C

kube-node04.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
temp=34.3'C

kube-node01.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
temp=32.2'C
~~~

#### Check the connected devices

Check the components of the PI composing the cluster (Those are PI 3B+)
~~~
ansible picluster -i inventory -m shell -a "lsusb"

kubemaster-pi.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 004: ID 0424:7800 Standard Microsystems Corp.
Bus 001 Device 003: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

kube-node02.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 004: ID 0424:7800 Standard Microsystems Corp.
Bus 001 Device 003: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

kube-node01.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 004: ID 0424:7800 Standard Microsystems Corp.
Bus 001 Device 003: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

kube-node04.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 004: ID 0424:7800 Standard Microsystems Corp.
Bus 001 Device 003: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

kube-node03.clusterX.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 004: ID 0424:7800 Standard Microsystems Corp.
Bus 001 Device 003: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
~~~

On the second cluster, some of the nodes are PI 3B instead of PI 3B+. LAN and WLAN are different.
~~~
ansible picluster -i inventory -m shell -s -a "lsusb"

master-pi.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 004: ID 0424:7800 Standard Microsystems Corp.
Bus 001 Device 003: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

home-pi.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 004: ID 10c4:8a2a Cygnal Integrated Products, Inc.
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp. SMC9514 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

nas-pi.kubedge.cloud | SUCCESS | rc=0 >>
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp. SMC9514 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
~~~

#### Other usefull commands


~~~
ansible picluster -i inventory -m shell -a "lsusb"
ansible picluster -i inventory -m shell -a "dmesg"
ansible picluster -i inventory -m shell -a "usb-devices"
ansible picluster -i inventory -m shell -a "lsblk"

ansible picluster -i inventory -m shell -a "blkid"
ansible picluster -i inventory -m shell -a "fdisk -l"
~~~


## Reference Links

- TBD

