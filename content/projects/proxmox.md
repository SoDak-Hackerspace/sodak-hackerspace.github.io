---
title: "Proxmox"
author: "trashpanda"
date: 2024-06-23T17:00:50-05:00
description: "Level 1 Hypervisor For Virtual Server Deployment"
tags: ["proxmox","virtualization"]
categories: ["infrastructure"]
---

# Overview

Proxmox is a level 1 hypervisor which means it performs virtualization at the bare metal level rather than sitting on top of an existing OS. Proxmox is a Debian based KVM stack, which aligns well with one of the missions at the hackerspace to promote free and open source software, and while you can pay for enterprise support, you can also just roll with a free version and add 3rd party community repositories. <br><br>

Some of the additional features of Proxmox include: 

- Web UI for management of the server
- Support for multiple storage solutions
- Easy to backup and restore
- Scalability
- LXC container support
- Virtual Networking
- API for future automation projects

So now that I've laid out the reasons for using Proxmox, let's talk about the specifics of what the installation at the hackerspace will look like. 

## Hardware

The hardware for our first Proxmox server will be an old HP EliteDesk 705 G4 DM. It's got a 4 Core AMD processor and currently 24GB of RAM. An odd configuration I know, but it's what I was able to pull out of the dumpster... literally. It'll be interesting to see how far this little machine will go. In my head I have a lot of plans for all the services that we could run and none are very CPU intensive. I honestly think we may hit a RAM limitation first, which would be an easy thing to upgrade before looking at either getting a beefier box or adding nodes to the Proxmox server and forming a cluster. 

## Installation Specifics

The server has an M.2 slot and a 2.5in harddrive slot so we'll populate the M.2 slot with a small boot drive and the 2.5in with a 1TB SSD for VM storage, backups, ISOs, containers, etc. The boot disk will be configured with ext4, so nothing fancy on that front, but I think BTRFS is a great option for the VM storage disk. It allows for snapshots, does integrity checks, and has served me well over the years. I know it's not best practice to have backups on the same disk as the data, so maybe that will get stored on the OS disk if room allows. There will most likely be a project in the future to configure a more robust backup strategy but for now we need to get off the ground. 

My policy on hypervisors is to keep them lean and clean so aside from the OS, I don't put much of anything else on them. That being said I do have a future project in mind to allow for remote access to the server using [Headscale](https://headscale.net/) which is an open source, self-hosted [Tailscale](https://tailscale.com/) solution. I think a good name for the first server in the SoDak Hackerspace is `blackelk.pve.sodakhacker.space`, after one of the most interesting South Dakotans, Nicholas Black Elk. 

## Post Installation Setup

WIP
