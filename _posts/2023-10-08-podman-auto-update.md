---
layout: single
title:  "Easily Update Your Containers with Podman Auto-Update"
date:   2023-10-08 16:00:00
excerpt: "Use this handy built-in feature of Podman to update all your container images with a single command."
categories: [Linux Administration]
tags: linux nextcloud podman docker container update
comments: true
---

I've written previously about the joys of using Podman to manage your containers, including the benefits of using it over Docker, but one of my favorite quality-of-life features is the [podman auto-update](https://docs.podman.io/en/stable/markdown/podman-auto-update.1.html) command.

In short, it replaces the series of commands you would normally run to update containers, for example:

1. `podman pull nextcloud-fpm:27`
2. `podman stop nextcloud-fpm`
3. `podman rm nextcloud-fpm`
4. `podman run [OPTIONS] nextcloud-fpm:27`
5. Repeat for each container.

Not only does podman auto-update save you all these steps, it will also automatically roll back to the previous image version if there are errors starting the new version, giving you some peace of mind when updating important applications.

## Requirements
* Podman installed
* Containers managed with systemd
* Containers you want to update must use the `--label "io.containers.autoupdate=registry"` run option

## Instructions

Recreate your existing systemd-managed containers with the `--label "io.containers.autoupdate=registry"` option. To do this, just edit your container's service file to include the option. See the below partial example for my Nextcloud container:

``` systemd
ExecStartPre=/bin/rm -f %t/%n.ctr-id
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--pod-id-file %t/pod-nextcloud-pod.pod-id \
	--sdnotify=conmon \
	--replace \
	--detach \
	--env MYSQL_HOST=mariadb \
	--env MYSQL_DATABASE=nextcloud \
	--env MYSQL_USER=${MYSQL_USER} \
	--env MYSQL_PASSWORD=${MYSQL_PASSWORD} \
	--volume %h/.podman/nextcloud/nextcloud-config:/var/www/html:z \
	--volume /mnt/nextcloud-data/data:/var/www/html/data:z \
	--label "io.containers.autoupdate=registry" \
	--log-driver=journald \
	--name nextcloud-app docker.io/library/nextcloud:${NEXTCLOUD_VERSION}-fpm
ExecStop=/usr/bin/podman stop --ignore --cidfile=%t/%n.ctr-id
```
\
Once you're done, reload the systemd daemon and restart the service.
``` shell
systemctl --user daemon-reload
systemctl --user restart container-nextcloud.service
```
\
Next, run the auto-update command with the `--dry-run` option. With this option, you'll get a preview of which containers will be updated without the update taking place.

``` shell
podman auto-update --dry-run

UNIT                       CONTAINER                     IMAGE                               POLICY      UPDATED
pod-nextcloud-pod.service  643fd5d3e2cb (nextcloud-app)  docker.io/library/nextcloud:27-fpm  registry    pending
pod-nextcloud-pod.service  71e48b691447 (mariadb)        docker.io/library/mariadb:10        registry    pending
pod-nextcloud-pod.service  9ed555fecdfa (caddy)          docker.io/library/caddy             registry    pending
```

### Output explained
* `podman auto-update` will show updates for every container that has the "io.containers.autoupdate=registry" label and do them all at once
* The `UNIT` column shows the same "pod" service for each container. This is because my containers are all managed by a single Podman pod.
* The `UPDATED` column shows "pending", which means there is an update available from the container registry.

\
Once you're ready to update, run the command again without the `--dry-run` option.

``` shell
podman auto-update
```
\
Podman will begin pulling the images from the registry, which may take a few minutes depending on your connection speed. If it completes successfully, you'll get fresh output with the `UPDATED` column changed to `true`.

``` shell
UNIT                       CONTAINER                     IMAGE                               POLICY      UPDATED
pod-nextcloud-pod.service  643fd5d3e2cb (nextcloud-app)  docker.io/library/nextcloud:27-fpm  registry    true
pod-nextcloud-pod.service  71e48b691447 (mariadb)        docker.io/library/mariadb:10        registry    true
pod-nextcloud-pod.service  9ed555fecdfa (caddy)          docker.io/library/caddy             registry    true
```
\
During this process, the were restarted automatically with the latest image. You can verify this with `podman ps`.

``` shell
podman ps

CONTAINER ID  IMAGE                                    COMMAND               CREATED        STATUS             PORTS                                     NAMES
0c5523997648  localhost/podman-pause:4.6.1-1692961071                        2 minutes ago  Up 2 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  0e075fb7b67b-infra
4ba992e83eeb  docker.io/library/caddy:latest           caddy run --confi...  2 minutes ago  Up 2 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  caddy
2a7d448b1b6b  docker.io/library/nextcloud:27-fpm       php-fpm               2 minutes ago  Up About a minute  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  nextcloud-app
9ec017721f16  docker.io/library/mariadb:10             --transaction-iso...  2 minutes ago  Up 2 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  mariadb
```
\
That's it! Your Podman containers were updated to the latest image version with a single command. This is a small feature, but one I've come to love in my time using Podman. If you get stuck, check out the project's [documentation for the auto-update command](https://docs.podman.io/en/stable/markdown/podman-auto-update.1.html). If you have broader questions about running Podman, I recommend reading my [series on building a reproducible Nextcloud server with Podman]({% link _posts/2023-08-27-nextcloud-podman.md %}).

Happy hacking!


