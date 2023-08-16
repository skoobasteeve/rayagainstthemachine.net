---
layout: single
title:  "Building a Reproducible Nextcloud Server, Part one: Choosing the stack"
date:   2023-08-15 11:59:00
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

I chose DigitalOcean back in 2016 mainly due to its excellent guides and popularity around the Jupiter Broadcasting community (got that sweet $100 promo code!). It was much easier to use than most other VPS providers and could have you up-and-running with an Ubuntu server and a public IP in minutes. In 2023, the VPS market is a bit more commoditized and there are some other great options out there. Linode initially came to mind, but their future became a bit murkier after they got acquired by Akamai in 2022, hyperscalers like AWS and Azure are too expensive for this use-case. I eventually landed on [Hetzner Cloud](https://www.hetzner.com/cloud) for the following reasons:

- Incredible value - for roughly $5 USD per month you get 2 vCPUs and 2GB of ram with 20TB of monthly traffic. That's basically double the specs of competing offerings.
- Great reputation - Hetzner has been around for 20+ years and has lots of good will in the tech community for their frugal dedicated server offerings. I wouldn't have chose them initially since their Cloud product didn't have offerings in the U.S., but recently they've expanded to include VPS's in Virginia and Oregon.
- Full-featured Terraform provider - This isn't unique to Hetzner, but it was a requirement for my new setup and their provider works great.

### Why not self host?

While I have a reliable server at home and 300mbps uploads, it's never going to match the bandwidth and reach of a regional data center. This wouldn't matter to me for most things, but I treat my Nextcloud server as a full Dropbox replacement, and it needs to perform as such. On that same note, I feel comfort knowing that it's separated from the more experimental environment of my homelab.

# Linux Distribution

One of the great benefits of containerized applications is that the host operating system matters much less than it used to, and the choice will likely come down to personal preferences. As long as it can run your chosen container runtime and you're familiar with the tooling, your choice will probably work as well as any other.

I've been running Ubuntu on my servers for years due to ease-of-use and my familiarity with it on the desktop. However, I've recently been using Fedora on my home computers and have gotten accustomed to Red Hat / RPM quirks and tooling in recent years. For this reason, and the ease of getting the latest Podman release (more below), I ended up choosing CentOS Stream 9. 

# Docker vs. Podman







