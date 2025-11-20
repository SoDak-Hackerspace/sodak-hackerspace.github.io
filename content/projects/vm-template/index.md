---
title: "Building a Virtual Machine Template"
date: 2025-11-20T13:04:14-06:00
draft: false
comments: false
images: ["./debian.png"]
---
# Overview

Deploying a VM template for our Proxmox server is a good idea for many little reasons including: installing tools, disabling unnecessary services, configuration of default services, speed of deployment, and the list goes on and on. All of these reasons can really be summed up in one overarching reason and that's minimizing configuration drift. Having a baseline template reduces the time spent on troubleshooting installations and makes it easier to configure automation tooling like Ansible in the future. In this post, I'll give a little bit of a detailed walkthrough for how I like to setup a Debian template. I'll make a few more, most notably Rocky Linux to have a RHEL flavor in our environment which will be important for the [FreeIPA](../ipa/index.md) project. But I'll choose Debian because it will be the defacto distribution of choice.

## Quick Specs

| Field              | Value            |
| -------------------|------------------|
| OS                 | Debian 13        |
| RAM                | 2G               |
| CPU                | 2 Cores          |
| Storage            | 32GB             |


## LVM and Disk Setup

Since we're using BIOS and not UEFI, we'll first create a 1GB parition for `/boot`. This should be more than enough space for storing kernel images and grub.

I like to use [LVM](https://wiki.archlinux.org/title/LVM) (Logical Volume Management) to manage storage on my servers it allows for expanding the disk in a way that's cleaner than traditional partitions. You can split LVM into three levels: Physical Volume (PV), Volume Group (VG), and Logical Volumes (LV). Basically LVM abstracts away storage and allows for creating a PV from a disk, then adding that PV to a VG, and expanding any of the LVs within the VG with that new disk storage. It's confusing at first, but this image has helped me understand what's happening under the hood:

{{< image src="./LVM.png" alt="LVM explained" >}}

| Volume Group  | Logical Volume | Initial Size   |
| ------------- | -------------- | -------------- |
| rootVG        | rootLV         | 5G             |
| rootVG        | optLV          | 3G             |
| rootVG        | tmpLV          | 2G             |
| rootVG        | usrLV          | 5G             |
| rootVG        | varLV          | 6G             |
| rootVG        | varlogLV       | 3G             |
| rootVG        | homeLV         | 3G             |
| rootVG        | swapLV         | 4G             |


## Installation

With our Debian ISO downloaded and VM provisioned through Proxmox, we can start the installation. While I'm sure that there's a way to get the disk setup in the graphical installation for Proxmox, I prefer the TUI install. I think it's actually easier to see the disk setup portion in that view. I'll skip the boilerplate stuff and get to the interesting part of the installation.

Here's the initial disk partition setup:

{{< image src="./partition_disks0.png" alt="partition start screen" >}}

We'll create a new partition of size 1GB and mount it at `/boot`. It'll be a primary although that doesn't necessarily matter. We'll place it at the beginning of the partition table and set the bootable flag on. We'll use EXT4 as the default filesystem.


{{< image src="./partition_disks1.png" alt="partition start screen" >}}

Next we'll create a new partition like before but we'll set that partition to the maximum size and *not set a mount point* in order to create a Physical Volume. With that out of the way, we'll select `Configure the Logicial Volume Manager` from the main partition disks menu.

{{< image src="./partition_disks2.png" alt="partition start screen" >}}

We'll have to write the current partitions to the disk before entering into LVM Configuration Manager. Once that's done, we'll select `Create Volume Group` and name it `rootVG` and select the partition we just made.

{{< image src="./partition_disks3.png" alt="partition start screen" >}}

We'll write those changes to disk and from here on out we'll work in the LVM Configuration Menu. With our Volume Group created, we can move on to configuring our Logical Volumes. We'll create a new logical volume, name it `rootLV` and give it a size of 5G as per our table above.

{{< image src="./partition_disks4.png" alt="partition start screen" >}}
<br>

{{< image src="./partition_disks5.png" alt="partition start screen" >}}
<br>

{{< image src="./partition_disks6.png" alt="partition start screen" >}}
<br>

We then repeat this process for each of our LVs listed in our table. Once this is done, we can finish writing the changes to the disk and are dumped out to the main menu of the partition disks menu. With our logical volumes made, we can move on to providing mount points for the newly created LVs. You can select each LV, under the `use as` select `EXT4` and then mount at the appropriate location. The only difference is the `swapLV`, which you'll need to select `swap` under `use as`:

{{< image src="./partition_disks8.png" alt="partition start screen" >}}
<br>

{{< image src="./partition_disks9.png" alt="partition start screen" >}}

Repeat this process for each of our LVs. With all of that done, we can then write the changes to the disk and move forward.

{{< image src="./partition_disks10.png" alt="partition start screen" >}}

The installation resumes with just basically clicking through the defaults. The only thing we need is to ensure that we have ssh server installed on the box when the menu presents itself. It's selected by default along with system utilities so we'll just leave that alone. Finally, we'll ensure the GRUB boot loader is installed on right disk. Should be the only option available.

{{< image src="./grub.png" alt="partition start screen" >}}

Once all that's done, we'll reboot and login to our debian template server to setup a few things.

## Post Installation

Post install there's a few initial things I want to configure on the template including installing packages, disabling ssh password auth, and making a systemd unit to handle the first boot of a newly deployed template. This gives the host new ssh host keys to avoid annoying ssh duplicated keys and security warnings when logging in via SSH. The packages that I want to install are just `vim`, `sudo`, `tmux`, and `git`. This will suffice for now and we can always go back and tweak this template if needed. It's also good practice to login and update the template every couple of months or so. Here's the package installation:

```bash
su -
apt update
apt install vim sudo git tmux
usermod -aG sudo <user>
```

These commands login as root, update the package cache, installs the packages, and adds the local user to the `sudo` group.

With that out of the way, we'll move on to modifying the SSH config to remove password access. This way key authentication is the only way to access the server remotely. This is a huge security step in my opinion and not implemented enough. If you want to place a key on a server, you can generate a key with `ssh-keygen -t ed25519` and then copy it over with `ssh-copy-id user@ip`.

{{< image src="./grub.png" alt="partition start screen" >}}

Now we can move onto the scripts that will live on the server. The first script we'll drop will be the `first-boot.sh` script which handles the machine ssh key generation when the vm is deployed from template. I throw this script in the `/root/scripts/first-boot.sh` location and then configure systemd to run it once on boot. The script checks if a file exists in the `/etc/ssh` directory and if it does then it exits without doing anything. Otherwise it creates that file and creates new keys.

```bash
#!/bin/bash
flag="/etc/ssh/new_keys"

if [[ ! -f "$flag" ]]; then
  touch /etc/ssh/new_keys
  rm -v /etc/ssh/ssh_host_*
  ssh-keygen -A
else
  :
fi

exit 0
```

The `systemd` unit file looks as such:

```systemd
[Unit]
Description=Run first-boot script to generate new SSH host keys
Before=ssh.service

[Service]
Type=oneshot
ExecStart=/root/scripts/first-boot.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Finally we can configure the cleanup script for the template located in `/root/scripts/template-cleanup.sh`. This should be run as the last thing before shutting down the server and converting to template. It will remove the ssh keys and cleanup logs and history files:

```bash
apt update
apt upgrade
apt autoremove
apt clean
truncate -s 0 /etc/machine-id
truncate -s 0 /var/lib/dbus/machine-id
rm -v /etc/ssh/ssh_host_*
truncate -s 0 /var/log/syslog
truncate -s 0 /var/log/auth.log
rm -rf /tmp/*
rm -rf /var/tmp/*
truncate -s 0 ~/.bash_history
truncate -s 0 /home/<user>/.bash_history
```


And that should be it! We now have a Debian 13 template that we can use to quickly deploy servers with LVM for easier, cleaner, and simpler disk management. There is a checklist of things you should do when you deploy from a template like this including;

- Change the hostname
- Set a static IP or reserve a DHCP lease (cleaner)
- Change password for user and root

The update process for the template, at least in Proxmox, looks like:

- Deploy from template
- Make changes
- Run `/root/scripts/template-cleanup.sh`
- Convert new VM to template
- Delete old template

Thanks for reading and I hope you learned something!
