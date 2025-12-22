---
title: My Homelabbing Journey Part 1 
date: 2025-12-16
categories: [homelab,proxmox,selfhosting]
tags: [journey,homelabbing,writeups,learning,notes]
toc: true
---

# From Laptop Virtualization to Bare-Metal Dreams
Hey everyone! Welcome to the first post in what I hope will become a series documenting my homelabbing adventures. This isn't my first rodeo with homelabbing, I've been tinkering for a few years, but it's my first serious step into dedicated hardware. Let's dive into how it all started!

### The Early Days: Type-2 Hypervisor Magin on a Budget Laptop
Like many of us, my homelab journey began on whatever hardware I had at hand: my daily driver laptop. Back then, I picked up an MSI GF63 Thin 10SCSR (10th gen Intel i7, upgraded to 32GB RAM, and with GTX 1650 Ti). Young me didn't know much about hardware reputations or thermals, but I did know virtualization demands RAM and a decent CPU. So, 32GB and an i7-10750H it was!

Using a Type-2 hypervisor (think VirtualBox (what I used in my case) or VMware Workstation), I built a full-blown cybersecurity lab:

- Active Directory domain with multiple domain controllers
- Vulnerable machines for practice (Windows and Linux boxes)
- An "attacker" Kali machine
- Monitoring tools: SIEM (like ELK or Splunk), IDS/IPS (Suricata or Snort), firewall (pfSense), and more

It was incredibly fun—breaking things, building defenses, experimenting with tools. Virtualization opened up a whole world, combining my love for servers, containers, storage redundancy, and cybersecurity. From the moment I spun up my first VM, I was hooked. "This stuff is cool," I thought, even if I wasn't the "geekiest" kid around.

But bottlenecks hit hard. Running everything on my host machine meant performance dips, overheating, and resource contention. Still, it got me through years of studying and experimenting. (I will dedicate it to another project realted to another series inshallah but mostly either ArchLinux or LFS on Gentoo)

### Mentoring and the Spark for a Real Lab
Fast-forward: I had the amazing opportunity to mentor at my school's Google Developer Student Club (GDSC—now part of Google Developer Groups). Working with my awesome co-mentor Ranim and some talented ML/AI folks, we guided younger students through tech workshops. It was one of the best periods of my life—teaching, learning, and building community.

The challenge? Many students' laptops couldn't even enable virtualization in BIOS (looking at you, budget machines with locked features). They needed a safe playground to experiment with cybersecurity concepts, tools, and vulnerable environments—step by step, without overwhelming them.

Sure, platforms like Hack The Box (HTB), TryHackMe (THM), or VulnHub exist (and they're great!), but for total beginners, we wanted something custom and guided. That's when Ranim and I brainstormed: What if we built a shared cybersecurity homelab?

My brain (thanks, ADHD-fueled idea traffic) went wild. Tracking everything was tough, but the creativity flowed. Unlike many homelab YouTubers with spare enterprise gear lying around, I had... an ancient LG laptop (dual-core, ~4GB RAM—more heater than computer). I tried turning it into a Debian server, but a locked BIOS password killed that dream. CMOS reset was an option but too risky as I didn't want to lose that laptop (call it attachment issues idc).

Broke college student mode activated. Thankfully, my parents helped out, and I sourced a used build from an IT contact: Intel i3 3rd gen, 16GB RAM, 512GB SSD. Pricey for the specs? Yeah. But exciting!

Before buying, I researched OS options—straight Debian? Or a bare-metal hypervisor? That's when I discovered Proxmox VE (built on Debian, bonus!). It blew my mind: clustering, ZFS, containers, VMs—all in one free package.

### The Joy of Installation and Early Experiments
Once the hardware arrived, I was ecstatic. Bootable USB with Proxmox ISO in hand, I set up a mini workstation next to my router: monitor, keyboard, mouse from my laptop, and my trusty little 8-port unmanaged switch wiring everything together. It felt so real—like a proper lab!

Installation was smooth, and I dove headfirst into Proxmox's features. VMs, LXC containers—it was all there. I deployed:

- Pi-hole for ad-blocking
- WireGuard VPN
- Docker experiments
- Even a lightweight Kubernetes cluster with K3s
- Reverse proxies (Nginx Proxy Manager, anyone?)

The Proxmox web interface became my new best friend (and speaking of best friends, apologoies for talking about my proxmox node on a daily basis for weeks).

### The Setbacks: IOMMU Woes and "Creative" RAID
Not everything went perfectly. I tried installing TrueNAS for proper storage management... and hit the infamous "no IOMMU support" wall. Enabled all the flags in the bootloader, but nothing. Turns out, the i3 didn't support VT-d—PCIe passthrough was a no-go.

My RAID dreams? Undeterred! With just the single 512GB SSD, I partitioned it and set up software RAID (mdadm mirroring). Totally suboptimal, but hey—it let me learn RAID concepts hands-on.

Life got in the way soon after. School resumed, internship hunt consumed me—exhausting times. The lab went dormant for over a year. Alhamdulillah, things worked out job-wise.

### The Revival: Hunting for Better Hardware
Fast-forward to now (late 2025), with more knowledge and a steady job, the homelab itch returned—stronger than ever. Time for an upgrade!
I scoured eBay, Amazon, Facebook Marketplace, and Kleinanzeigen (shoutout to my brother in Germany for the affordable options). First target: a Dell OptiPlex 3050 Mini (16GB DDR4, 256GB NVMe) for €100, haggled to €95. Logistics failed—heartbreak.

But persistence pays off. Found a reliable seller on Facebook Marketplace, and contacted him on WhatsApp with great feedback. Alhamdulillah, scored a Dell OptiPlex 5070:

- i5-9500T (6 cores, coffee lake goodness—VT-d supported!)
- 16GB DDR4 RAM
- 256GB NVMe SSD

Picked it up this past Sunday. Can't wait!

### What's Next?
Planning a proper setup:

- Main node: New OptiPlex with Proxmox
- Backup/secondary: Old i3 machine (maybe join as a cluster node?)
- Storage: Got a couple of HDDs to experiment with—ZFS? RAID? We'll see.

Dreaming of a small Proxmox cluster for high availability and migration fun.


Stay tuned for upcoming parts where I will cover: Installation, clustering attempts, self-hosting services, and inevitable troubleshooting.

**Inshallah** this becomes a long series. The more I break, the more I’ll share.

Stick around if you want to learn together.

```bash{.dua-tawfik}
رَبِّ زِدْنِي عِلْمًا ۞ وَفَهْمًا وَحِكْمَةً
رَبِّ يَسِّرْ وَلَا تُعَسِّرْ ۞ وَتَمِّمْ بِالْخَيْرِ
```

#### ~ Darknight