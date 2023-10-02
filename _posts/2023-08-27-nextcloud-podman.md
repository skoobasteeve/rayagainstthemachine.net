---
layout: single
title:  "Building a Reproducible Nextcloud Server, Part one: Choosing the stack"
date:   2023-08-27 10:00:00
excerpt: "After successfully hosting a Nextcloud instance on the same VPS for 7 years, I decided to rebuild it from scratch with modern tooling."
categories: [Self-Hosting, Linux Administration]
tags: linux nextcloud podman docker container vps
comments: true
---

Nextcloud was the first application I *really* self-hosted. I don't mean self-hosting like running the Plex app in the system tray on your gaming PC; I mean a dedicated VPS, exposed to the world, hosting my personal data. The stakes were high, and over the last seven years, it pushed me to grow my Linux knowledge and ultimately made me a far better sysadmin. 

A lot happened during that seven years. Containers and infrastructure-as-code blew up and changed the IT industry. Nextcloud as a company and an application grew tremendously. I got married. Throughout all these changes, my little $5 DigitalOcean droplet running Nextcloud on the LAMP stack kept right on ticking. Despite three OS upgrades, two volume expansions, and fifteen(!) Nextcloud major-version upgrades, that thing refused to die. It continued to host my (and my wife's) critical data until the day I decommissioned it just under 60 days ago. 

# Why change?

As a sysadmin and a huge Linux nerd, I'd been following the technology and industry changes closely, and every time I heard about something new or read a blog post I couldn't help but wonder "if I rebuilt my Nextcloud server today, how would I do it?". Everything is a container now, and infrastructure and system configuration is all defined as text files, making it reproducible and popularizing the phrase "cattle, not pets".  I wanted a chance to embrace these concepts and use the skills I spent the last seven years improving. Plus, what sysadmin doesn't like playing with the new shiny?

# Goals

So what did I want to accomplish with this change? 

1. **Cutting-edge technologies** - Not only did I want to play with the latest tools, I wanted to become proficient with them by putting them into production.
2. **Reproducibility** - Use infrastructure-as-code tooling so I could spin up the whole stack and tear it back down with only a few commands.
3. **Reliability** - Whatever combination of hardware and technologies I ended up with, it needed to be absolutely rock-solid. The only reason this thing should break is if I tell it to (intentionally or not)

# Hosting provider

I chose DigitalOcean back in 2016 mainly due to its excellent guides and popularity around the Jupiter Broadcasting community (got that sweet $100 promo code!). It was much easier to use than most other VPS providers and could have you up-and-running with an Ubuntu server and a public IP in minutes. In 2023, the VPS market is a bit more commoditized and there are some other great options out there. Linode initially came to mind, but their future became a bit murkier after they got acquired by Akamai in 2022, while hyperscalers like AWS and Azure are too expensive for this use-case. I eventually landed on [Hetzner Cloud](https://www.hetzner.com/cloud) for the following reasons:

- Incredible value - for roughly $5 USD per month you get 2 vCPUs and 2GB of ram with 20TB of monthly traffic. That's basically double the specs of competing offerings.
- Great reputation - Hetzner has been around for 20+ years and has lots of goodwill in the tech community for their frugal dedicated server offerings. I wouldn't have chose them initially since their Cloud product didn't have offerings in the U.S., but recently they've expanded to include VPSs in Virginia and Oregon.
- Full-featured Terraform provider - This isn't unique to Hetzner, but it was a requirement for my new setup and their provider works great.

### Why not self host?

While I have a reliable server at home and 300mbps uploads, it's never going to match the bandwidth and reach of a regional data center. This wouldn't matter to me for most things, but I treat my Nextcloud server as a full Dropbox replacement, and it needs to perform as such. On that same note, I feel comfort knowing that it's separated from the more experimental environment of my homelab.

# Linux Distribution

One of the great benefits of containerized applications is that the host operating system matters much less than it used to, and the choice mostly comes down to personal preference. As long as it can run your chosen container runtime and you're familiar with the tooling, your choice will probably work as well as any other.

I've been running Ubuntu on my servers for years due to ease-of-use and my familiarity with it on the desktop. However, I've recently been using Fedora on my home computers and have gotten accustomed to Red Hat / RPM quirks and tooling in recent years. For this reason, and the ease of getting the latest Podman release (more below), I ended up choosing [CentOS Stream 9](https://www.centos.org/centos-stream/). 

# Docker vs. Podman

I've been using [Docker](https://www.docker.com/) to host a number of applications on my home server for the last few years with great success, and Docker is still far-and-away the most popular way to run individual containers. However, as the [OCI standard](https://opencontainers.org/) has become more widely adopted, other tools like [Podman](https://podman.io/) have started to appear. Podman, backed by Red Hat, offers near 1:1 command compatibility with Docker and has some lovely added benefits such as:
- Designed to run without root - Podman runs containers as a standard user, greatly reducing the risk to the server if one of the containers is compromised.
- No daemon required - On the same note, there isn't a continuously running daemon in the background with root access to your system. The risks of the Docker socket are [well-documented](https://docs.docker.com/engine/security/protect-access/), and this negates that risk entirely.
- Modern and lightweight - One of the benefits of not being first is that you can learn from everyone else's mistakes. Podman is built using lessons learned from Docker while creating an easy pathway to move from individual containers to full Kubernetes deployments.

Podman has been under rapid development recently, and there's a lot of excitement about it in Linux circles. While Docker would have worked just fine for my purposes, I decided to use this project as an opportunity to get familiar with Podman and see if it could potentially replace my other Docker-based applications.

# Deployment

Unlike my previous Nextcloud server which was like a zen garden that I tended carefully, I wanted my new server to be completely reproducible on a moment's notice. Using containers accomplishes part of this approach, but still leaves many parts of the server configuration to automate! Thankfully, there are a ton of tools available in 2023 to help with this.

## Terraform

To deploy the server itself, with associated volumes, firewall, etc, [Terraform](https://www.terraform.io/) was the obvious choice. While there are some competitors coming up like [Pulumi](https://www.pulumi.com/), Terraform is still the dominant player in the field and popularized the infrastructure-as-code concept. I had some experience using it at work, but I had never had the opportunity to build something from scratch with it. After reading the documentation for the [Hetzner Cloud provider](https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs), I was confident Terraform would be able to give me everything I needed. 

## Ansible

Once the VPS is deployed and I have SSH access, Terraform's job stops. This is where I would typically connect to the server and start installing packages, configuring the webserver, and doing all the other server setup tasks I've done a thousand times over the years. If only there was a tool that could do all these steps for me while simultaneously documenting the entire setup!

Enter [Ansible](https://www.ansible.com/). Anything you could possibly think to do on a Linux box, Ansible can do for you. Think of it like a human-readable Bash script that handles all the rough edges for you. While writing the playbooks takes some work, once you have them written, you can run them again and again and expect (mostly) the same results each time. I chose Ansible due to it's stateless, agent-less architecture and the ability to run it from any computer with SSH access to the target hosts. Like Terraform, I love that the entire configuration is text-based and easily managed with Git. 

# What's next?

This post talked about the ideas and goals I had going into this project, and in Part 2 I'll talk about the details of the implementation, and how sometimes things seem a lot easier in a blog post than they turn out to be in reality! If you're interested in the nitty-gritty of how these tools work for a project like this, stay tuned for the next post in the series.

[*Link to Part two*]({% link _posts/2023-08-28-nextcloud-podman-part2.md %})




