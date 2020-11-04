---
layout: post
title: "Unraid vs Snapraid"
date: 2020-11-04 18:00:00 +0000
tags: tech storage comparison
---

Following on from my [previous post]({% post_url 2020-11-03-binpacking %}), I've been experimenting with different setups for storing media on an old computer. The characteristics of this use case is that the files are relatively large, write-once-read-many, do not require high-speed or highly-parallel access, and the disks are a highly heterogenous jumble of spare drives, subject to frequent change. The requirements are just that I have a directory to dump files in, spread them across drives in some manner, and provide protection against up to two disk failures. This makes something like ZFS or Btrfs overkill for my setup, so the two solutions that I looked at were Unraid and a custom SnapRAID + mergerfs setup.

<!--more-->

# TL;DR

See the sections beneath the table for discussion of these points. Summary at the bottom of the article.

| Unraid | SnapRAID |
| -      | -        |
| **Closed source, but decently priced and OS is very transparent. Your data is *not* locked in** | **Open source** |
| **Complete package** | **Just redundancy** |
| Union mounting built-in | Basic read-only symlink-snapshot union view. Try mergerfs |
| Sharing built-in | Manually set-up SMB or NFS or whatever |
| SMART monitoring built-in | Use smartd |
| Email and webhook notifications built-in | Manual setup |
| Web GUI built-in | Try OpenMediaVault |
| Task scheduling built-in | No task scheduling, and remember that syncs and checks have to be run manually. Community systemd units exist |
| Easy UI for VMs and Docker | Too many alternatives to list. Maybe Proxmox? |
| Opinionated with a big community, good for beginners or people who want to "set it and forget it" | Go your own way, good for people who want more control or to integrate it with an existing server setup |
| **Works on blocks** | **Works on files** |
| Have to clear disks before adding | Can start with populated disks |
| Supports XFS or Btrfs (+ LUKS) | Supports virtually any mountpoint, even Windows hosts |
| Parity sync duration proportional to raw block size | Parity sync duration proportional to actual usage. Partial checks supported OOTB |
| Consumes entire block device for parity | Parity stored as plain files |
| **Live parity** | **Snapshot parity** |
| Data is protected immediately | Unsynced data is unprotected. Deleted or changed data can remove protection from files on other drives until the next sync. Cannot sync whilst writing |
| Automatically simulates failed disks from parity | Must mount a replacement disk and wait for lost files to be recovered one-by-one |
| Write speed bottlenecked by parity disks (without mover) | Parity sync can be run during quiet hours |
| No protection against accidental deletion | Snapshot parity gives you a grace period to recover from stupid mistakes |
| Not a substitute for a proper offsite backup | Also not a substitute for a proper offsite backup |
| **"Mover" built-in** | **rsync + cron?** |
| **No built-in integrity checks** (community plugins available) | **Checksums all files** |
| Has (small) potential to silently restore corrupted data | Verifies all restored data with checksums |
| Power loss during write can leave array in ambiguous state | Syncs are transactional |

# Licensing

Let's start with the elephant in the room. SnapRAID is an open source project while Unraid costs between $60 and $130 depending on how many drives you use. Unless you have ethical objections to proprietary software I don't think this is a big difference. The Unraid licenses are fairly cheap when you consider how much hardware costs, and they're lifetime licenses too.

Some people are concerned that Unraid locks in your data - this isn't true; I moved from Unraid to SnapRAID with just a parity sync. Your data drives are plain XFS or Btrfs, perhaps with LUKS. The parity drive format is undocumented afaik, but I'm sure someone's reverse engineered it. Presumably the first parity drive is plain xor.

The Unraid system is just Slackware and you can easily SSH in and do whatever you want to it. The Unraid developers interact a lot with the community and often promote community plugins and tutorials, so they're very open to hacking and experimenting on the OS.

The main disadvantage of Unraid is that it's "all-or-nothing", as discussed in the next section. I'm not sure how comfortable Unraid is running in a VM, or if you always need to dedicate a physical host.

# Complete package vs just redundancy

The first difference to understand is that "Unraid" refers to a whole operating system that provides redundancy, union mounting, networked file access, monitoring, scheduling, notifications, VM and Docker management, and a web GUI to configure it all. On the other hand SnapRAID is *just* a utility for adding redundancy to a set of existing mountpoints, and everything else must be provided elsewhere. I have heard that OpenMediaVault provides an Unraid-like experience for SnapRAID and mergerfs. I haven't tried it myself yet, but will try to do a write-up if I do.

Almost everyone using SnapRAID will want union mounting (the ability to view all the disks as one pool). SnapRAID comes with built-in support for a read-only union view, in the form of a command which will populate a pool directory with symlinks to the actual files. Most users will want to replace this with more powerful which can offer live updates and write support. The most popular solution is mergerfs, though I think Windows users often go with Stablebit Drivepool. I wrote [a post comparing the write strategies of mergerfs and Unraid]({% post_url 2020-11-03-binpacking %}), but the executive summary is that union mounting is a fairly solved problem and both Unraid and mergerfs do it well. Mergerfs is more powerful, but assumes more expertise from users.

Another thing that you'll really want to setup is SMART monitoring of your drives with email alerts. Unraid has this out-of-the-box. For custom SnapRAID systems you'll probably want to use smartd and your preferred MTA or other notification dispatcher.

# Blocks vs files

Another key difference is that Unraid works on block devices while SnapRAID works on files. IMO SnapRAID is the clear winner here.

Unraid is more like mdadm, where you take some raw block devices, create virtual RAID devices, and then work on top of those. The Unraid OS only supports XFS or Btrfs with optional LUKS, but in theory I expect the storage virtualisation layer could support any FS. This also means that adding a new disk requires a parity sync; the way this is usually done is by "pre-clearing" the new disk to all zeroes, since `a xor 0 = a`, meaning that the existing disks do not need updating. Unfortunately this means you can't officially add a full disk to Unraid, though I expect it should be theoretically possible if you do a full parity sync afterwards. The other disadvantage of this is that a parity sync or check time is proportional to the size of your largest data drive, regardless of how much actual space is used.

SnapRAID OTOH does not really care about filesystems or block devices; it just works on directories, and is so relaxed about semantics that it even runs fine on Windows. You can start off with full drives, as I did when I moved from Unraid to SnapRAID. Even the parity itself is just plain files rather than requiring a raw block device, as Unraid does, and will be proportional to actual usage, meaning you can take a risk and temporarily overcommit a parity drive smaller than a data drive, as long as you don't use more storage on a single drive than the parity drive can hold.

# Live vs snapshot parity

The main difference between Unraid's storage system and SnapRAID is that Unraid runs as a daemon, constantly maintaining the virtual devices discussed previously, while SnapRAID is actually just a set of terminating commands that you run manually or on a schedule. You add/delete/update files and run `snapraid sync`. If a disk fails, you install, format, and mount a new one, then tell SnapRAID to start restoring the files onto it. You won't have access to all your files until this finishes. This also means your new files are unprotected until you run a sync, and if you delete files from one drive, you remove protection from *some* files on other drives until you sync again, though the integrity features described later mean it should never result in silent corruption of restored files.

Unraid is more like a traditional RAID setup. Parity is written before returning "success" to the writing program. Normal reads on a healthy array only use one data disk, but when a disk fails the system will immediately start using the other disks to emulate it from parity, allowing you to continue using the whole array without interruption. Ironically this means that *Un*raid is more "real" RAID than Snap*RAID* IMO, since I consider high-availability to be the main purpose of RAID.

This does mean that a file deleted from Unraid is deleted immediately, while SnapRAID allows you until the next sync to notice your mistake and restore it, however this should be considered a little bonus rather than something to rely on. If you really need that, use a filesystem with proper snapshots like ZFS, and remember that snapshots and disk redundancy are still not a substitute for offsite backup.

# The mover

Writing to hard drives can be slow, especially since Unraid array writes are limited by the speed of the parity disk. Unraid provides a built in utility to use an SSD as a write cache. Files are initially written to the SSD, and on a schedule (by default, in the early hours of the morning), they are moved to the array proper. Unlike the rest of Unraid, this works at a file, rather than block, level. The SSD is included in the union mount so it's essentially transparent to the user. Since files on the cache are not covered by array parity, there is built in support for using two SSDs in RAID1 using Btrfs.

SnapRAID doesn't have anything like this, but the concept is fairly simple, so it shouldn't be too hard to write your own script using rsync and cron or systemd timers, or just do it by hand. Your writes aren't bottlenecked by parity drives in the first place, due to the snapshot model.

# Scrubbing and integrity

Both systems can perform scrubs/parity checks, however Unraid is more limited.

On a base install, Unraid parity checks are all-or-nothing, while SnapRAID defaults to scanning 8% of the files with the oldest scrub date, meaning that every file is checked every three months if you run the command weekly. As discussed before, Unraid has to scan the whole block device while SnapRAID just scans actual files.

Unraid suffers from the same problem as traditional RAID setups: if the calculated parity does not match the stored parity, is it the data or the parity which is wrong? SnapRAID is more similar to ZFS in that it stores actual file checksums, so can always answer correctly (beyond reasonable doubt). Presumably using dual parity on Unraid should help here.

Silent data corruption on hard disks is often considered to be virtually a myth, since it would be very unlikely to happen in such a way that the hard disk's own error correction doesn't detect it and return a media error, however it's not the only way that the parity can become desynced. I believe Unraid is vulnerable to a "write hole" if power is lost during a write, whereas SnapRAID performs its syncs in a transactional manner, so can safely resume them.

As a bonus, SnapRAID also provides a `dup` command to list duplicate files, and also supports hard links, so it may be possible to leverage these for some limited deduplication support.

# Summary

Both are great solutions. Unraid is more opinionated while SnapRAID is more flexible. There are tools to simplify SnapRAID setup, giving it an Unraid-like UX, but the reverse - using Unraid as a facet of an existing Linux install - is not possible. SnapRAID's unique advantage is the file-based model, while Unraid's is the fact that it provides high availability like a traditional RAID system. The checksumming built into SnapRAID is nice, but there are Unraid plugins that do similar things.