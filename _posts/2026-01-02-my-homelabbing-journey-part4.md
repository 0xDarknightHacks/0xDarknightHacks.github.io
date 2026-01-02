---
title: My Homelabbing Journey Part 4
date: 2026-01-02
categories: [homelab,proxmox,self-hosting,immich,backup]
tags: [journey,homelabbing,writeups,learning,notes]
toc: true
---

# Self-Hosting Immich to Save My iPhone (And a Trip to Ain Draham)
Hey everyone, I'm back! Sorry for the radio silence—procrastination played a role, but I also got swept up in real life. Spent an amazing time with family in **Ain Draham**, a stunning spot in northern Tunisia. Think dense forests, fresh air, beautiful scenery, and total nature vibes. It was the perfect recharge.

### The Urgent Need: iPhone Storage Crisis

This all ties back to the homelab. My iPhone (64GB model—why did I do that to myself?) was completely full right before the trip. I needed space for photos and videos of Ain Draham, forests, family moments... and cats, obviously.

Normally I'd start self-hosting with Pi-hole, but this time? Priority shift. I needed a **photo/video backup solution**, stat.

No research needed—I already knew **Immich** was the move. It's an open-source, self-hosted Google Photos alternative with a killer mobile app.

### Installing Immich on Proxmox

Used the trusty **Proxmox VE Helper Scripts** (tteck's scripts are life-savers).

Created an LXC container and ran the Immich script—super straightforward.

Bumped storage to 150GB (way over the 20GB minimum) since I had my 1TB drive available. I'm not a heavy photographer—mostly friend hangouts, silly moments, and yes... cats. All cats deserve documentation.

Skipped advanced features like facial recognition (needs GPU passthrough) and maps for now. Basic backup was the goal—I was in a rush.

Setup the mobile app: point it to my server's local IP, log in, enable auto-backup. Easy.

### The Backup Marathon

What I *didn't* expect: the initial upload taking a **full day**. Thousands of photos/videos trickling up slowly. Sometimes it paused if the app wasn't foreground—iOS battery optimization things.

Researched: some folks reported slowness on older versions, but I was on the latest. Still, it got done eventually.

Pro tip: Made a second copy on another drive. You know the meme—"I'm an IT guy, I have backups of my backups."

Freed up my phone just in time for the trip. Captured all the Ain Draham magic without storage anxiety.

### Verdict on Immich

Absolutely recommend self-hosting it. Privacy, no subscriptions, full control.

That said, check alternatives (Photoprism, Nextcloud Memories, etc.)—pick what vibes with you.

For me? Immich wins for the app and features. I'll experiment with AI stuff later when I snag a cheap compatible GPU.

### Quick Hardware Update

Grabbed a **USB-to-Gigabit Ethernet adapter**—the OptiPlex onboard is only 100Mbps in some modes, so this unlocks full speeds.

### Teaser for Next Part

The big one is coming: a deep-dive tutorial on Pi-hole + Cloudflare DDNS + Nginx Proxy Manager. It drove me absolutely nuts—expect rants, fixes, and victory.

Stay tuned!

Thanks for reading – more homelab adventures coming soon. 

**Inshallah** this becomes a long series. The more I break, the more I’ll share.

Stick around if you want to learn together.

```bash{.dua-tawfik}
رَبِّ زِدْنِي عِلْمًا ۞ وَفَهْمًا وَحِكْمَةً
رَبِّ يَسِّرْ وَلَا تُعَسِّرْ ۞ وَتَمِّمْ بِالْخَيْرِ
```

#### ~ Darknight