---
layout: single
title:  "Painless On-Demand NAS Connections in Linux with Systemd Automount"
date:   2020-10-07 19:00:00
categories: [Linux Administration]
tags: linux samba nas systemd ubuntu
comments: true
---

If you have a NAS, you probably want to be able to connect to it automatically each time you log into your Linux machine. Adding an entry to your fstab is great for desktops with persistent ethernet connections, but you quickly run into trouble using this method on a laptop that's frequently jumping on and off the network. Your distribution's file manager may have a way to do this, but wouldn't it be nice to have a consistent method accross all modern distributions? Systemd to the rescue!

Instead of trying to mount the network shares at login or boot, we're going to mount them on-demand by combining Systemd mount and automount units. With this method, the shares won't appear until you navigate to the designated mount points in the terminal or file manager.

*NOTE:* These instructions were written for **Ubuntu 20.04**, but the principles should apply to nearly all modern distributions. Adjust for your distro's package manager and install locations. 

### Systemd Unit Files

If you're not familiar with Systemd unit files and how they work, I would highly recommend reading the [Freedesktop.org article](https://www.freedesktop.org/software/systemd/man/systemd.unit.html) on the subject. Unit files are effectively services that can be run on-demand or triggered by events or specific times. For our purposes, we're going to use both [mount](https://www.freedesktop.org/software/systemd/man/systemd.mount.html#) and [automount](https://www.freedesktop.org/software/systemd/man/systemd.automount.html#) unit files. 

### Create a mount point

You'll need to create dedicated folders on your machine where the shares will be mounted.

```bash
$ sudo mkdir -p /mnt/smb/sambashare
$ sudo mkdir -p /mnt/nfs/nfsshare
```

### Create a credentials file

If your Samba server uses authentication, you'll need to create a file with your login details that Systemd can use to connect. These should be saved in a safe location with restricted permissions. 

```bash
$ sudo nano /etc/samba/smbcreds
```

```
username=[USERNAME]
password=[PASSWORD]
```

```bash
$ sudo chmod 600 /etc/samba/smbcreds
```

### Install the client packages

#### Samba

```bash
$ sudo apt install samba cifs-utils
```

#### NFS

```bash
$ sudo apt install nfs-common
```

### Create the systemd unit files

To make this work, we need (2) unit files for each connection: the **mount** unit and the **automount** unit. These use a specific naming convention that follows the path of the mount point. For example, if our share is at `/mnt/smb/sambashare` the mount file should be named `mnt-smb-sambashare.mount`. The same goes for the automount file. **If you don't name the units this way they will not function.**  

The below instructions assume your samba share is located at `//example.server/sambafiles`.

```bash
$ sudo nano /etc/systemd/system/mnt-smb-sambashare.mount
```

```
[Unit]
Description=samba mount for sambafiles
Requires=systemd-networkd.service
After=network-online.target
Wants=network-online.target

[Mount]
What=//example.server/sambafiles
Where=/mnt/smb/sambashare
Options=vers=2.1,credentials=/etc/samba/smbcreds,iocharset=utf8,rw,x-systemd.automount,uid=1000
Type=cifs
TimeoutSec=30

[Install]
WantedBy=multi-user.target
```

A few notes on the above file:  

* `vers=2.1` - adjust this based on the version of samba running on your server
* `uid=1000` - adjust this based on your local user ID to avoid permissions problems. This is usually 1000 on a desktop system. 

\
Next we need to create the automount file in the same location.

```bash
$ sudo nano /etc/systemd/system/mnt-smb-sambashare.automount
```

```
[Unit]
Description=samba automount for yourfiles
Requires=network-online.target

[Automount]
Where=/mnt/smb/sambashare
TimeoutIdleSec=0

[Install]
WantedBy=multi-user.target
```

#### NFS

The below instructions assume your NFS share is located at `example.server:/srv/nfsfiles`.

```bash
$ sudo nano /etc/systemd/system/mnt-nfs-nfssahre.mount
```

```
[Unit]
Description = nfs mount for nfsfiles

[Mount]
What=example.server:/srv/nfsfiles
Where=/mnt/nfs/nfsshare
Type=nfs
Options=defaults

[Install]
WantedBy=multi-user.target
```

\
Same as before, we need to create the automount file in the same location.

```bash
$ sudo nano /etc/systemd/system/mnt-smb-nfsshare.automount
```

```
[Unit]
Description=nfs automount for nfsfiles
Requires=network-online.target

[Automount]
Where=/mnt/nfs/nfsshare
TimeoutIdleSec=0

[Install]
WantedBy=multi-user.target
```

\
NFS mounts by nature are a bit more straightforward than samba. In an all-Linux environment they will lead to fewer headaches and I recommend them highly. 

### Put them to work!

At this point your unit files are ready to go but systemd doesn't know about them. Run the following commands to test your mount:

```
$ sudo systemctl daemon-reload
$ sudo systemctl start mnt-nfs-nfsshare.mount
$ sudo systemctl start mnt-smb-smbshare.mount
```

\
If all went well, you should see your file shares at their designated mount points. To verify, check the status of your service and look for any errors.

```
$ sudo systemctl status mnt-nfs-nfsshare.mount
● mnt-smb-sambashare.mount - samba mount for sambafiles
   Loaded: loaded (/etc/systemd/system/mnt-smb-sambashare.mount; static; vendor preset: enabled)
   Active: active (mounted) since Wed 2020-10-07 18:05:34 EDT; 1min 1s ago
    Where: /mnt/smb/sambashare
     What: //example.server/sambafiles
  Process: 13005 ExecMount=/bin/mount //example.server/sambashare /mnt/smb/sambashare -t cifs -o vers=2.1,credentials=/etc/samba/smbcreds,iocharset=utf8,rw,x-systemd.automount,uid=1000 (code=exited, status=0/SUCCESS)
    Tasks: 0 (limit: 4915)
   CGroup: /system.slice/mnt-smb-sambashare.mount

Oct 07 18:05:34 raylyon-ThinkPad-T450s systemd[1]: Mounting samba mount for sambafiles...
Oct 07 18:05:34 raylyon-ThinkPad-T450s systemd[1]: Mounted samba mount for sambafiles.
```

\
Next, enable your automount files to start at boot. This will allow your shares to mount on-demand. 

```
$ sudo systemctl enable mnt-nfs-nfsshare.automount
$ sudo systemctl enable mnt-smb-smbshare.automount
```

\
That's it! To test, reboot your system, open the mountpoint in terminal or the file manager, and your share will mount before your eyes. If you have any questions or critiques on the above instructions, please shoot me an [email](mailto:ray@raylyon.net) or open a [Github issue](https://github.com/skoobasteeve/skoobasteeve.github.io.2/issues). 

Thanks for reading and happy hacking!