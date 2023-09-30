---
layout: single
title:  "Building a Reproducible Nextcloud Server, Part two: Setting up Podman containers"
date:   2023-08-28 10:00:00
excerpt: "In the second installment of my Nextcloud server rebuild, we'll get our containers up-and-running with Podman."
categories: [Self-Hosting, Linux Administration]
tags: linux nextcloud podman docker container vps
comments: true
---

[*Link to Part one*]({% link _posts/2023-08-27-nextcloud-podman.md %})

Now that I've established the stack, let's dive right in to setting up the Nextcloud application with Podman. By the end of this post, we'll have Nextcloud conatiners running locally along with systemd service files to reproduce them on any system.

## Notes on rootless Podman

One of the big advantages of using Podman over Docker is that Podman was designed from the beginning to run without root privileges. This has many positive security implications, but there also a number of "gotchas" to be aware of, and I'll be pointing them out as I go through the instructions.

For more details, the Podman project maintains a helpful doc on their Github: [The Shortcomings of Rootless Podman](https://github.com/containers/podman/blob/main/rootless.md).

## Create a pod

Podman "pods" are logical groupings of containers that depend on one another. Think of a pod like a Service in Docker Compose; a group of containers that work together to run a single application or service. Once we have a pod that contains all our containers, we can stop and start all of them with a single command. For a much more thorough explanation on what pods are and how they work, check out this [excellent post](https://developers.redhat.com/blog/2019/01/15/podman-managing-containers-pods) on the Red Hat developer blog.

**Rootless Gotcha #1**  
In most Linux distributions, unprivileged applications are not allowed to bind themselves to ports below 1024. To fix:
1. Run `sudo sysctl net.ipv4.ip_unprivileged_port_start=80`
2. Create a file under `/etc/sysctl.d/` and add the line `net.ipv4.ip_unprivileged_port_start=80`. This makes the change permanent.

Now, create a new pod called "nextcloud".

``` shell
podman pod create \
--publish 80:80 \
--publish 443:443 \
--network slirp4netns:port_handler=slirp4netns \
nextcloud
```

You can see your newly created pod by running `podman pod ps`.

``` shell
POD ID        NAME        STATUS      CREATED         INFRA ID      # OF CONTAINERS
d1b78054d6f4  nextcloud   Created     2 minutes ago  f4a80daae64f  1
```

#### Options explained
* `--publish 80:80` and `--publish 443:443` opens ports 80 and 443 for the webserver. Containers within pods can communicate with eachother fully on their own isolated network, but for outside traffic to reach the containers, we need to open the necessary ports at the pod level.
* `--network slirp4netns:port_handler=slirp4netns` solves another rootless gotcha. By default, the webserver in rootless mode sees all HTTP requests as coming from the container's local IP address. This isn't very helpful for accurate logs, so the above option changes the pod's port handler to fix the issue. There may be some performance penalties for doing this, but for low-traffic servers it shouldn't be a problem.

## Create the containers

To get Nextcloud up and running, we'll use the following containers:

* nextcoud-fpm - A minimal install of Nextcloud that requires a separate webserver.
* mariadb - Database officially supported by Nextcloud.
* caddy - The Caddy webserver, which I love for the simplicity of its config and the built-in automatic SSL via Let's Encrypt.

First, create a working directory structure where you'll store all the containers' data. For this project, I broke mine out like this:

``` shell
.podman
└── nextcloud
    ├── caddy
    │   ├── config
    │   └── data
    ├── mariadb
    └── nextcloud
        ├── config
        └── data  
```

Next, I'll go over each container, showing you the full command I used to create them and explaining each option.

### MariaDB

We'll create the database container first since it doesn't technically depend on either of the other containers.

``` shell
podman run \
--detach \
--env MYSQL_DATABASE=nextcloud \
--env MYSQL_USER=nextcloud \
--env MYSQL_PASSWORD=nextcloud \
--env MYSQL_ROOT_PASSWORD=nextcloud \
--volume $HOME/.podman/nextcloud/mariadb:/var/lib/mysql:z \
--name mariadb \
--pod nextcloud \
docker.io/library/mariadb:10 
```

#### Options explained

* `--env MYSQL_DATABASE=nextcloud` - Name of the database Nextcloud will use.
* `--env MYSQL_USER=nextcloud` - Database user Nextcloud will use.
* `--env MYSQL_PASSWORD=nextcloud` - Password for the Nextcloud database user. Be sure to change this to something more secure and save it somewhere!
* `--env MYSQL_ROOT_PASSWORD=nextcloud` - Password for the extcloud database root user. Be sure to change this to something more secure and save it somewhere!
* `--volume $HOME/.podman/nextcloud/mariadb:/var/lib/mysql:z` - Creates a bind mount in the folder you created for MariaDB to store its database and configuration data. The `:z` option is needed to give directory access on selinux systems.
* `--name mariadb` - Sets the name of the container.
* `--pod nextcloud` - Attaches the container to the nextcloud pod we previously created.
* `docker.io/library/mariadb:10` - Container image we're going to download and run.

After you run the command, you can check if the container is running with the `podman ps` command.

``` shell
CONTAINER ID  IMAGE                                    COMMAND     CREATED            STATUS         PORTS                                     NAMES
f4a80daae64f  localhost/podman-pause:4.7.0-1695839078              About an hour ago  Up 29 seconds  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  d1b78054d6f4-infra
c5961a86a474  docker.io/library/mariadb:10             mariadbd    29 seconds ago     Up 29 seconds  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  mariadb
```

**Note:** The other container you see in the output, `d1b78054d6f4-infra` is a helper container for the nextcloud pod. 

### Nextcloud

``` shell
podman run \
--detach \
--env MYSQL_HOST=mariadb \
--env MYSQL_DATABASE=nextcloud \
--env MYSQL_USER=nextcloud \
--env MYSQL_PASSWORD=nextcloud \
--volume $HOME/.podman/nextcloud/nextcloud/config:/var/www/html:z \
--volume $HOME/.podman/nextcloud/nextcloud/data:/var/www/html/data:z \
--name nextcloud-app \
--pod nextcloud \
docker.io/library/nextcloud:27-fpm
```

#### Options explained

* `--env MYSQL_HOST=mariadb` - Name of the container hosting the database. Thanks to Podman's built-in DNS, container names will resolve to their IP address, so all we have to do is point Nextcloud at `mariadb` and it will find the database.
* `--env MYSQL_DATABASE=nextcloud` - Name of the database Nextcloud will use, the same that you created in the `mariadb` container.
* `--env MYSQL_USER=nextcloud` - Database user Nextcloud will use, the same that you created in the `mariadb` container.
* `--env MYSQL_PASSWORD=nextcloud` - Password for the Nextcloud database user, the same that you created in the `mariadb` container.
* `--volume $HOME/.podman/nextcloud/nextcloud/config:/var/www/html:z` - Creates a bind mount in the folder you created for Nextcloud to store its configuration files. The `:z` option is needed to give directory access on selinux systems.
* `--volume $HOME/.podman/nextcloud/nextcloud/data:/var/www/html/data:z` - Creates a bind mount in the folder you created for Nextcloud's data directory. The `:z` option is needed to give directory access on selinux systems.
* `--name nextcloud-app` - Sets the name of the container.
* `--pod nextcloud` - Attaches the container to the nextcloud pod we previously created.
* `docker.io/library/mariadb:10` - Container image we're going to download and run.

You should now have both containers running:

``` shell
CONTAINER ID  IMAGE                                    COMMAND     CREATED            STATUS         PORTS                                     NAMES
f4a80daae64f  localhost/podman-pause:4.7.0-1695839078              About an hour ago  Up 18 minutes  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  d1b78054d6f4-infra
c5961a86a474  docker.io/library/mariadb:10             mariadbd    18 minutes ago     Up 18 minutes  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  mariadb
13d5c43c0b4d  docker.io/library/nextcloud:27-fpm       php-fpm     5 seconds ago      Up 5 seconds   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  nextcloud-app
```

### Caddy

Before we start the Caddy container, we'll need to write a config in the form of a Caddyfile. Since we're just focused on getting the initial containers working, let's do a simple configuration without HTTPS:

Create file named `Caddyfile` in `$HOME/.podman/nextcloud/caddy/config/` and paste the below contents.

``` hcl
http://localhost:80 {

    root * /var/www/html
    file_server
    php_fastcgi nextcloud-app:9000 {
        root /var/www/html
        env front_controller_active true
    }
}
```

The above configuration is a bare-minimum to run Nextcloud locally on port 80. We'll make lots of tweaks before we're ready for production.

Assuming the Caddyfile is in place, run the below command to spin up the final container:

``` shell
podman run \
--detach \
--volume $HOME/.podman/nextcloud/nextcloud/config:/var/www/html:z \
--volume $HOME/.podman/nextcloud/caddy/config/Caddyfile:/etc/caddy/Caddyfile:z \
--volume $HOME/.podman/nextcloud/caddy/data:/data:z \
--name caddy \
--pod nextcloud \
docker.io/library/caddy
```

Verify that all (3) containers are running with `podman ps`.

``` shell
CONTAINER ID  IMAGE                                    COMMAND               CREATED         STATUS         PORTS                                     NAMES
f4a80daae64f  localhost/podman-pause:4.7.0-1695839078                        2 hours ago     Up 45 minutes  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  d1b78054d6f4-infra
c5961a86a474  docker.io/library/mariadb:10             mariadbd              45 minutes ago  Up 45 minutes  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  mariadb
13d5c43c0b4d  docker.io/library/nextcloud:27-fpm       php-fpm               26 minutes ago  Up 26 minutes  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  nextcloud-app
b29486a99286  docker.io/library/caddy:latest           caddy run --confi...  4 minutes ago   Up 4 minutes   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp  caddy
```
\
Go to http://localhost in your browser and...

![nextcloud-podman01](/assets/images/screenshots/nextcloud-podman01.png){:class="img-responsive"}

Ta-da! We have Nextcloud!

Since the containers are all part of the `nextcloud` pod, you can stop and start all of them with one command. You can run `podman pod stop` to take them down and `podman pod start` to bring them back up. Pretty cool eh?

## Generate Systemd files

Even better than starting/stopping your containers at the pod level is doing it with systemd! This will allow you to manage your nextcloud pod the same way as any other systemd service, including enabling it to run at system start.

Instead of writing all the systemd unit files by hand, we're going to use a handy subcommand of the podman application, `podman generate systemd`.

First, make sure the pod and all its containers are running. Then, run the below command:

``` shell
podman generate systemd --new --files --name nextcloud

/home/raylyon/container-nextcloud-app.service
/home/raylyon/container-caddy.service
/home/raylyon/container-mariadb.service
/home/raylyon/pod-nextcloud.service
```

The output gives you the path to each file, and we'll need to move these files into the systemd user directory, `$HOME/.local/share/systemd/user/` Create the directory if it doesn't already exist.

``` shell
mkdir -p $HOME/.config/systemd/user
```

Copy each of the files into the above directory.

``` shell
cp $HOME/*.service $HOME/.config/systemd/user/
```

Reload the systemd user daemon.

``` shell
systemctl --user daemon-reload
```

Start the service corresponding to the pod.

``` shell
systemctl --user start pod-nextcloud
```