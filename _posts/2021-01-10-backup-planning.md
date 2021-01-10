---
layout: single
title:  "Backups: Planning an Effective Strategy"
date:   2021-01-10 18:10:00
categories: [Linux Administration]
tags: linux backups BorgBackup borg rsync nextcloud
comments: true
---

If you've ever asked a sysadmin or computer geek for advice, you've probably heard the universal refrain:  "Always have a backup" or "RAID is not a backup" or my personal favorite "One is none" (referring to copies of your data).

While virtually everyone working in this space agrees on the above, you'll get wildly different answers to your next question: "How do I do it?"

The intention of this post is to cut through the noise and give you an understanding of the  concepts you should be thinking about before committing to a particular technology or piece of software. 

***NOTE*** - Due to the nature of this blog, most of the example applications I provide are FOSS and Linux-compatible, but you can apply the same concepts to other apps and operating systems.

## Break Your Data Into "Tiers"

Whether you're talking about a server or a desktop, it's a safe bet that not every file on your hard drive(s) holds equal importance. That is, there are some files you care about vastly more than others. When setting up a backup system, sorting these files by priority will save you time, money, and disk space down the road.

### Tier 1: Files you can't afford to lose

Family pictures, tax documents, home videos; things that aren't replaceable or would incur significant cost to replace, financial or otherwise. **These should exist in at least 3 places**.

### Tier 2: Files that you *can* replace, but don't want to

Music and movies, either ripped from physical media or "obtained" from elsewhere, are good contenders for this category. You may have spent hours meticulously cataloging your media collection, but those files can always be re-downloaded or ripped. **These should exist in at least 2 places**.

### Tier 3: Files that don't matter

Installed applications, games, operating system files, miscellaneous downloads, etc. Files in this category can be easily replaced with a simple re-install or are publicly available on the internet. It's not necessary to back up these files unless you've got extra terabytes burning a hole in your pocket.

## Choose a Strategy for Each Tier

There are a huge amount of software options for handling your backups, and the strengths and weaknesses of each primarily depend on the type of data your backing up. It's important to understand these characteristics and so you can apply them to your different data tiers. 

### Cloud Sync Services

Services like Google Drive, Dropbox, or self-hosted [Nextcloud](https://nextcloud.com/) (my personal choice) use desktop and mobile applications to automatically sync your files between your devices and a centralized cloud server. Files that you save to a defined folder on your computer are instantly uploaded to the cloud and downloaded to the other devices. These files are accessible from anywhere via a web interface. 

**Advantages**

+ **Easy to set up** - Pick a service, create an account, download the application and sync your files. 

+ **Instant and automatic** - Backups occur whenever a file is added or changed. No need to schedule a backup ahead of time or manually press a button.

+ **Resilient** - The web applications almost always have a "trash bin" for deleted file recovery, version control for rollbacks, and server infrastructure that gets its own backups. The files also exist offline on your synced devices.

+ **Readily Available** - Your files are accessible from any computer anywhere in the world via a web browser, including your smartphone.

**Disadvantages**

- **Expensive for high volumes** - The free tiers of most sync services only give you around 5-15GB of storage, so for anything larger than that you'll have to pay for the additional storage. 

- **Cumbersome for large files** - It's easy for the desktop apps to get bogged down with large files and slow down your computer, and once your get up into the hundreds of gigabytes, you could fill up the hard drives of all your devices pretty quickly. You'll also be severely limited by the slow upload speeds of most home broadband connections.

- **Reliance on third-party** - You have to trust that Google, Dropbox, or other services will respect your privacy and keep your files safe. **EXCEPTION:** [Self-hosted Nextcloud server.](https://docs.nextcloud.com/server/20/admin_manual/installation/)

**Best for**

- **Tier 1** - Documents take up very little space and upload quickly. Even a years worth of photos from your smartphone will only use a few GB in most cases. 

- **Tier 2 (smaller files)** - Application config files or even mp3s if you have a small collection. 

### Local File Sync

This is as simple as it gets: plug in a USB hard drive or connect to a network share and copy your files to it. While it's possible to accomplish this with a drag-and-drop, there are many great software tools that can help: [FreeFileSync,](https://freefilesync.org/) [DirSyncPro](https://www.dirsyncpro.org/), and [rsync](https://wiki.archlinux.org/index.php/rsync) are all great options for the desktop. 

**Advantages**

- **SImple** - Does not require extensive knowledge to set up and understand what's happening, and files aren't hidden behind compression or custom folder structures. Great for people who want visual verification of a successful backup.

- **Fast** - A local USB3 or gigabit LAN connection is going be faster than your internet upload speed 9 times out of 10. 

- **Cheap** - In the long run, local storage is always cheaper than cloud.

- **Offline and in your control** - No need to rely on any third-party service or server that may be untrustworthy.

**Disadvantages**

- **Not as resilient as other options** - File sync applications, while simple, don't always have more advanced features like versioning and verification, making it easier to run into problems without realizing it. 

**Best for**

- **Tier 2** - Without the storage limitations of the cloud, it's great to use on all your files, no matter how large.

### Local Backups

What's the difference between local backup and local sync? It's largely semantics, but for the purposes of this post I'll make some distinctions. The file sync applications I mentioned above are great for just that: ensuring that a folder in one location is an exact copy of a folder in another location. This is great, but if you want to get *serious* about your backups, a more specialized application is what you want. 

Backup applications like [BorgBackup](https://www.borgbackup.org/), [Duplicati](https://www.duplicati.com/), and [BackupPC](https://backuppc.github.io/backuppc/) can also back up to a USB HDD or network share, but they also provide features that make your backups more efficient and resilient: versioning, encryption, de-duplication, and compression to name a few. 

**Advantages**

- **Secure** - Encryption allows you to protect your backups with a password.

- **Resilient** - Retention of multiple versions and built-in error checking lowers your chances of losing data significantly. 

- **Cheap** - Same hardware requirements as local sync, and all the software examples I gave are free and open-source.

- **Offline and in your control** - No need to rely on any third-party service or server that may be untrustworthy.

**Disadvantages**

- **Complexity** - The additional features these apps offer can make them more difficult to set up, and the extra features mean the backups will be hidden by specialized file names and folder structures. 

**Best for**

- **Tier 2** - Without the storage limitations of the cloud, it's great to use on all your files, no matter how large.

- **Tier 1** - Combine with a cloud option and you'll have hard time ever losing your data. 

### Cloud Backups

We already established the cloud as a great option for your Tier 1 files, so what makes cloud backups different from cloud sync services? Cloud backup applications use the same methodology as the local backup applications I mentioned above, but instead of going to a USB HDD or NAS, we're sending them to a cloud storage provider. Encryption, compression, de-duplication and versioning are features you can expect to find, and many of the apps I referenced previously are cloud-compatible. [BorgBackup](https://www.borgbackup.org/), [Restic](https://restic.net/), and [Duplicati](https://www.duplicati.com/) are all great options in this space.

**Advantages**

- **Secure** - Encryption allows you to use third-party hosting and storage services with fewer privacy and security concerns. They can't see your files, only the encrypted data.

- **Resilient** - Retention of multiple versions and built-in error checking lowers your chances of losing data significantly. 

- **No hardware required** - No need to connect to local hard drive or network share, all you need is an internet connection and the backup will run in the background at a scheduled time. 

- **Cheaper storage options than cloud sync** - Since device sync and web availability isn't a requirement for this type of backup, you're options are opened up to virtually any cloud storage vendor. 

**Disadvantages**

- **Can still be expensive for high volumes** - While usually cheaper than cloud sync services, you'll still have to pay a subscription fee based on your usage. 

- **Complexity** - Just like local backups, the applications take more work to set up than a simple sync service and the backups are stored in esoteric formats and folder structures.

**Best for**

- **Tier 1** - When your files are too large to practically store on a sync service. Also good for use in addition to a sync service.

## Local Backups - USB or NAS?

If you're starting from nothing, getting a USB hard drive will likely be your first step. If you don't have one lying around already, they can be purchased cheaply and are easy to plug in and use. However, if you can afford it, the ideal solution is either a pre-built NAS appliance (Synology, iXSystems, or QNAP) or a DIY NAS using either a traditional x86 PC or something like a Raspberry Pi or RockPro64. Either way, it's important to understand the advantages and limitations of both options.

### USB Hard Drives

**Advantages**

- **Cheap** - 4TB drives are less than $100 at most retailers.

- **Simple to set up** - Have you ever plugged in a USB cable before? That's it!

- **Portable** - Keep them on your desk or hide them in a drawer when you're done.

**Disadvantages**

- **Not good for laptops** - You probably don't want to carry your laptop around with a USB hard drive hanging out of it. With a mobile device, you'll have to remember to periodically plug it in and run the backup manually.

- **Current or resilient: pick one** - If you have a computer that can be tethered to a USB HDD 24/7, you can automate your backups and run them as often as you like. However, you then open yourself up to a single-point-of-failure situation where your O.S. could accidentally corrupt the data, or a disaster like an electrical surge or fire could take out both at once. To avoid this, you can detach the drive and store it in a secure location after the backup completes, but then you have to remember to plug it back in and your backups likely won't remain current as a result.

### NAS

**Advantages**

- **Shareable** - Since a NAS resides on your network, you can back up multiple laptops, desktops, and servers at once without having to plug in a cable. 

- **Great for laptops** - Since you don't need to plug in a hard drive, your laptop can back itself up automatically at a scheduled time. 

- **Separate from your devices** - Since a NAS is not directly connected to any one computer, the single-point-of-failure scenario is less likely. 

**Disadvantages**

- **Pre-built devices can be expensive** - Even Synology's cheapest option with 4TB will be close to 4x the price of a USB hard drive with the same storage. 
- **Setup and maintenance** - Whether pre-built or custom, a NAS is a full computer running on your network that needs to be configured, updated, and maintained. To minimize this, choose a purpose-built device from Synology or QNAP. 

## So what should you do?

Everybody's situation is different, but here are some general conclusions you can draw from this information:

1. **Tier 1** data should be backed up both locally and with a cloud sync or backup service.

2. **Tier 2** data should be backed up locally in at least one additional location.

3. Local backups are preferred over simple file syncs for their reliability and resiliency.

4. If you can't maintain a local backup, combine a cloud sync service with a cloud backup application.

5. **The most important thing is that you do *something***. Your data is too valuable and your computer's hard drive is too volatile to leave things up to chance.

## Bonus: What I Use

- Some **Tier 1** data is synced to a Nextcloud server hosted on DigitalOcean and that server is backed up to my local NAS using [BorgBackup](https://www.borgbackup.org/) and [Borgmatic](https://torsion.org/borgmatic/).

- All **Tier 1** data on my NAS is backed up nightly to [BorgBase](https://www.borgbase.com/) using BorgBackup and Borgmatic.

- **Tier 2** data on my NAS is backed up nightly to external hard drives using ZFS and [Sanoid/Syncpoid](https://github.com/jimsalterjrs/sanoid).

Have a great backup system of your own? As always, feel free to leave a comment or reach out to me directly with questions or feedback. Since BorgBackup is clearly a favorite of mine, I'll be sure to cover it in detail in a future post.

Thanks for reading and happy hacking!
