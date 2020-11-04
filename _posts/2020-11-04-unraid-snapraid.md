---
layout: post
title: "Comparison of Unraid and Snapraid"
date: 2020-11-04 19:00:00 +0000
tags: tech storage comparison
---

Following on from my [previous post]({% post_url 2020-11-03-binpacking %}), I've been experimenting with different setups for storing media on an old computer. The characteristics of this use case is that the files are relatively large, write-once-read-many, do not require high-speed or highly-parallel access, and the disks are a highly heterogenous jumble of spare drives, subject to frequent change. The requirements are just that I have a directory to dump files in, spread them across drives in some manner, and provide protection against up to two disk failures. This makes something like ZFS or btrfs overkill for my setup, so the two solutions that I looked at were Unraid and a custom SnapRAID + mergerfs setup.

<!--more-->

# TL;DR

| Unraid | SnapRAID |
| -      | -        |
| **Closed source, but decently priced and OS is very transparent. Your data is *not* locked in** | **Open source** |
| **Complete package** | **Just parity** |
| Union mounting built-in | Basic read-only symlink-snapshot union mounting. Try mergerfs |
| Sharing built-in | Manually set-up SMB or NFS or whatever |
| SMART monitoring built-in | Use smartd |
| Email notifications built-in | Manual setup |
| Web GUI built in | Try OpenMediaVault |
| Automatic task scheduling | No task scheduling, and remember that syncs and checks have to be run manually. Community systemd units exist |
| Easy UI for VMs and Docker | Too many alternatives to list. Maybe Proxmox? |
| Big community, good for beginners or people who want something opinionated to "set it and forget it" | Go your own way, good for people who want more control or to integrate it with an existing server install |
| **Works on blocks** | **Works on files** |
| Have to clear disks before adding | Can start with populated disks |
| Supports XFS or btrfs | Supports virtually any mountpoint, even Windows hosts |
| Parity sync proportional to raw block size | Parity sync proportional to actual usage. Partial checks supported |
| Consumes entire block device for parity | Parity stored as plain files |
| **Live parity** | **Snapshot parity** |
| Automatically simulates failed disks from parity | Must mount a replacement disk and wait for lost files to be recovered one-by-one |
| No protection against accidental deletion | Snapshot parity gives you a grace period to recover from stupid mistakes, but it's no substitute for proper backups! |
| Data is protected immediately | Unsynced data is unprotected. Deleted or changed data can remove protection from files on other drives until the next sync |
| **No built-in integrity checks** (community plugins available) | **Checksums all files** |
| **"Mover" built-in** | **Write your own?** |

# Licensing

Let's start with the elephant in the room. SnapRAID is an open source project while Unraid costs between $60 and $130 depending on how many drives you use. Unless you have ethical objections to proprietary software I don't think this is a big difference. The Unraid licenses are fairly cheap when you consider how much hardware costs, and they're lifetime licenses too.

Some people are concerned that Unraid locks in your data - this isn't true, I moved from Unraid to SnapRAID with no issue. Your data drives are plain XFS or btrfs, perhaps with LUKS. The parity drive format is undocumented afaik, but I'm sure someone's reverse engineered it. Presumably the first parity drive is plain xor.

The Unraid system is just Slackware and you can easily SSH in and do whatever you want to it. The Unraid developers interact a lot with the community and often promote community plugins and tutorials, so they're very open to hacking and experimenting on the OS.

# Complete package vs just parity

The first difference to understand is that "Unraid" refers to a whole operating system that provides redundancy, union mounting, networked file access, monitoring, scheduling, notifications, VM and Docker management, and a web GUI to configure it all. On the other hand SnapRAID is *just* a utility for adding redundancy to a set of existing mountpoints, and everything else must be provided elsewhere. I have heard that OpenMediaVault provides an Unraid-like experience for SnapRAID and mergerfs. I haven't tried it myself yet, but will try to do a write-up if I do.

Almost everyone using SnapRAID will want union mounting - the ability to view all the disks as one pool. SnapRAID comes with built-in support for a read-only union view, in the form of a command which will populate a pool directory with symlinks to the actual files. Most users will want something more powerful, which offers live updates and write support. The most popular solution is mergerfs, though I think Windows users often go with Stablebit Drivepool. I wrote [a post comparing the write strategies of mergerfs and Unraid]({% post_url 2020-11-03-binpacking %}), but the executive summary is that union mounting is a fairly solved problem and both Unraid and mergerfs do it well. Mergerfs is more powerful, but assumes more expertise from users.

Another thing that you'll really want to setup is SMART monitoring of your drives with email alerts. Unraid has this out-of-the-box, for custom SnapRAID systems you'll probably want to use smartd and your preferred MTA.

# Blocks vs files

The other major difference is that Unraid works on block devices while SnapRAID works on files. IMO SnapRAID is the clear winner here.

Unraid is more like mdadm, where you take some raw block devices, create virtual RAID devices, and then work on top of those. The Unraid OS only supports XFS or btrfs with optional LUKS, but in theory I expect the storage virtualisation layer could support any FS. This also means that adding a new disk requires a parity sync; the way this is usually done is by "pre-clearing" the new disk to all zeroes, since `a xor 0 = a`, meaning that the existing disks do not need updating. Unfortunately this means you can't officially add a full disk to Unraid, though I expect it should be theoretically possible if you do a full parity sync afterwards. The other disadvantage of this is that a parity sync or check time is proportional to the size of your largest data drive, regardless of how much actual space is used.

SnapRAID OTOH does not really care about filesystems or block devices; it just works on directories, and is so relaxed about semantics that it even runs fine on Windows. You can start off with full drives, which is how I was able to move from Unraid to SnapRAID with only a parity sync. Even the parity itself is just plain files rather than requiring a raw block device, as Unraid does, and will be proportional to actual usage, meaning you can take a risk and temporarily use a parity drive smaller than a data drive, as long as you don't use more storage on a single drive than the parity drive can hold.

# Live vs snapshot parity

TODO

Ironically this means that *Un*raid is more "real" RAID than Snap*RAID* IMO, since I consider high-availability to be the main purpose of RAID.

# Scrubbing and integrity

TODO

# The mover

TODO