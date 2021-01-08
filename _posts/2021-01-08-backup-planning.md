---
layout: single
title:  "Backups: Planning an Effective Strategy"
date:   2021-01-08 12:45:00
categories: [Linux Administration]
tags: linux backups borg
comments: true
---

If you've ever asked a sysadmin or computer geek for advice, you've probably heard the universal refrain: "Don't forget to back up your data!" or "Always have a backup" or "RAID is not a backup" or my personal favorite "One is none" (referring to copies of your data).

While virtually everyone working in this space agrees on the above, there will be wildly different answers to your next question: "How do I do it?"

The intention of this post is to cut through all the noise and give you an understanding of the broad concepts you should be thinking about before committing to a particular technology or piece of software. 

## Break Your Data Into "Tiers"

Whether you're talking about a server or a desktop, it's a safe bet that not every file on your hard drive(s) holds equal importance. That is, there are some files you care about vastly more than others. When setting up a backup system, sorting these files by priority will save you time, money, and disk space down the road.

### Tier 1: Files you can't afford to lose

Family pictures, tax documents, home videos; things that aren't replaceable or would incur significant cost to replace, financial or otherwise. 

### Tier 2: Files that you *can* replace, but don't want to

Music and movies, either ripped from physical media or "obtained" from elsewhere, are good contenders for this category. You may have spent hours meticulously cataloging your media collection, but those files exist elsewhere and could be re-downloaded or ripped. 

### Tier 3: Files that don't matter

Installed applications and games, operating system files, miscellaneous downloads. Files in this category can be easily replaced with a simple re-install or are publicly available on the internet. 

## Choose a Strategy for Each Tier

There are a huge amount of software options for handling your backups, and the strengths and weaknesses of each primarily depend on the type of data your backing up. It's important to understand these characteristics and so you can apply them to your different data tiers. 

### Cloud Syncing

Services like Google Drive, Dropbox, or self-hosted Nextcloud (my personal choice) use desktop and mobile applications to automatically sync your files between your devices and a centralized cloud server. Files that you save to a defined folder on your computer are instantly uploaded to the cloud and downloaded to the other devices. The files are accessible from anywhere via a web interface. 

**Advantages**

+ **Easy to set up** - Pick a service, create an account, download the application and sync your files. 

+ **Instant and automatic** - Backups occur whenever a file is added to change. No need to schedule a backup time or manually press a button.

+ **Resilient** - The web applications almost always have a "trash bin" for deleted file recovery, version control for rollbacks, and server infrastructure that gets its own backups. Not to mention the files exist offline on your synced devices.

+ **Highly Available** - Your files are accessible from any computer anywhere in the world via a web browser, including your smartphone.

**Disadvantages**

- **Expensive for high volumes** - The free tiers of most sync services only give you around 5-15GB of storage, so for anything larger than that you'll have to pay for additional storage. 

- **Cumbersome for large files** - It's easy for the desktop apps to get bogged down with large files and slow down your computer, and once your get up into the 100s of gigabytes, you' could fill up the hard drives of all your devices pretty quickly. You'll also be severely limited by the slow upload speeds of most home broadband connections.

- **Reliance on third-party** - You have to trust that Google and Dropbox will respect your privacy and keep your files safe. **EXCEPTION:** Self-hosted Nextcloud server.

**Best for**

- **Tier 1** - Documents take up very little space and upload quickly. Even a years worth of photos from your smartphone will only take a few GB in most cases. 

- **Tier 2 (smaller files)** - Application config files or even mp3s if you have a small collection. 

### Offline Syncing

This is as simple as it gets: plug in a USB hard drive and copy your files to it. While it's possible to accomplish this with a drag-and-drop, there are many great software tools that can help us with this: FreeFileSync, DirSyncPro, and rsync are all great options for the desktop. 

**Advantages**

- **SImple** - Does not require extensive knowledge to set up and understand what's happening. 

- **Fast** - A local USB3 connection is going be faster than your internet upload speed 9 times out of 10. 

- **Cheap** - 4TB USB hard drives are less than $100 with no subscription fee required.

- **Offline and in your control** - No need to rely on any third-party service or server that may be untrustworthy.

**Disadvantages**

- **Not good for laptops** - You probably don't want to carry your laptop around with a USB hard drive hanging out of it. With a mobile device, you'll have to remember to periodically plug it in and run the backup manually. 

- **Current or resilient: pick one** - If you have a computer that can be tethered to a USB HDD 24/7, you can automate your backups and run them as often as you like. However, you then open yourself up to a single-point-of-failure situation where your O.S. could accidentally corrupt the data or a disaster like an electrical surge or fire could take out both at once. To avoid this, you can detach the drive and store it in a secure location after the backup completes, but then you have to remember to plug it back in and your backups likely won't remain current. 

**Best for**

- **Tier 2** - Large files and folders are no problem for an external drive. USB storage is cheap and fast. 

- **Tier 1** - Just don't use it as your only backup.
















