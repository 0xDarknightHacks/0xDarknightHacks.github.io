---
title: My Homelabbing Journey Part 3
date: 2025-12-22
categories: [homelab,proxmox,optiplex,self-hosting,debian,linux,repurposing,old-hardware]
tags: [journey,homelabbing,writeups,learning,notes]
toc: true
---

# Finally Installing Proxmox on the OptiPlex
Welcome back! let’s get to **Part 3** – the one where things actually start coming together. We’ve got thrifted routers bridged with DD-WRT, wired Internet in my room at a stable ~100 MB/s download... time to fire up the main star: **Proxmox on the Dell OptiPlex 5070**.

### The ISO Adventure

Grabbed the latest Proxmox VE ISO from the official site. Yeah, I went with the bleeding-edge version (9.1-1 at the time) even though it was super fresh—stability? We'll see.

First attempt: **Ventoy**. I love the idea—one USB, multiple ISOs. Plugged in my drive, set it up... and it just refused to boot properly on the OptiPlex. No idea why. I swear it wasn’t user error (okay, maybe a little).

Ended up with a corrupted partition mess. Spent way too long fixing the USB, then gave up and went with the reliable classic: **balenaEtcher**. Flashed the ISO, done in minutes. Lesson re-learned: sometimes simple wins.

### The Display Port Surprise

USB ready, plugged into the OptiPlex, keyboard and mouse connected... nothing on screen.

Turns out OptiPlex 5070s (and many modern Dells) use **DisplayPort** primarily, and my ancient monitor only had VGA and HDMI. Tried VGA → black screen. Dug out a USB-C-to-HDMI adapter I had lying around → still nothing.

Sigh. Quick trip to the shop: bought a proper **DisplayPort cable** for 25 TND (~$8 USD). Worth every coin.

Back home, plugged everything in:

- DisplayPort to my old monitor
- Random ancient keyboard (couldn’t find my good one—still hunting)
- Mouse I had from before

(No creamy mechanical keyboard yet, but one day... one day. Same for a decent monitor.)

### Installation: Smooth Sailing

Booted into the Proxmox installer. Everything went perfectly: disk selection, networking, root password, done.

Rebooted into the web interface at `https://ip:8006`. That first login feeling? Pure satisfaction.

### Post-Install Magic with Helper Scripts

I could have gone manual: edit sources.list, disable enterprise repo, add no-subscription repo, remove subscription nag, update, etc.

But why suffer when community heroes exist?

Ran the famous **Proxmox VE Post-Install Script** from [community-scripts.github.io](https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install&category=Proxmox+%26+Virtualization):

```bash
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/install/post-pve-install.sh)"
```

## The Cursed LG Laptop Revival Attempt (And Why It Failed)
Once proxmox was up and running, I figured I dive into a side quest that turned into a full-blown horror story: trying to breathe new life into our family's ancient **LG CD500 laptop**.

This beast has been in the family since around 2012 (maybe earlier—who knows). It's still *technically* functional, running Windows 7 infested with every virus known to mankind, locked legacy BIOS, cracked hinges, the works. I thought: perfect low-power server candidate! Pi-hole, Jellyfin, lightweight Docker stuff—why not?

Spoiler: it didn't go as planned.

### Step 1: Teardown and Parts Harvest

First move: disassemble the laptop. Inside:

- 250GB 2.5" SATA HDD (original)
- 2 × 2GB DDR3 RAM sticks (total 4GB—painful by 2025 standards)

Not exactly a powerhouse, but enough for basic self-hosting.

I had a bigger idea: swap in my 1TB HDD (the one I was using as extra storage in the OptiPlex Proxmox node, I forgot to mention that).

So I:

- Pulled the 1TB drive from the OptiPlex
- Ran full SMART tests in Proxmox (all good)
- Mounted it read-only to check data
- Wiped it completely

Now... what to install?

### The Plan: Bare-Metal Debian

Goal: lightweight Debian (stable or testing) as a low-power server.

Problem #1: Locked legacy BIOS with unknown password. No boot menu access, no changing boot order reliably. (I tried all possible solutions but didn't manage to fix it)

Problem #2: Can't easily boot from USB (BIOS lock + no reliable way to make it boot first).

Solution I chose: **dd the Debian netinst ISO directly to the HDD itself**.

Yes, you read that right. Debian ISOs are hybrid—they can be dd'd to a disk and boot like a USB stick. I figured: flash the ISO to the 1TB drive, slap it in the laptop, and let it boot straight into the installer.

Downloaded the latest amd64 netinst ISO from debian.org, ran:

```bash
dd if=debian-12.7.0-amd64-netinst.iso of=/dev/sdX bs=4M conv=fdatasync status=progress
```
Moved the drive back to the LG. Powered on... and it actually booted into the Debian installer! Victory... or so I thought.

### The Nightmare Begins: Partitioning Hell
Everything went fine until... **partitioning**.
No matter what I did: guided, manual, msdos table, ext4 + swap—the installer threw:
"Failed to create a file system. The ext4 file system creation in partition #X of /dev/sda failed."
Over and over.
I dropped to shell, tried manual wipes, mkfs.ext4 by hand—sometimes it worked, but the installer kept failing later steps.
Hours of troubleshooting later, I finally understood the root cause.

### Technical Deep Dive: Why It Failed
Here's the full breakdown (for anyone attempting something similar—learn from my pain):
#### The Core Issue
Debian netinst ISOs are hybrid:

- They contain an ISO9660 filesystem (for CD booting)
- Plus a partition table (for USB/HDD booting)

When you dd it to the target HDD:

- The laptop boots from the ISO9660 section (treats it like a CD)
- The installer mounts /dev/sda1 (the ISO9660 partition) as /cdrom
- The installer sees the installation media on the target disk
- For safety, it refuses to format any partition on that disk

Classic self-referential paradox. The drive is both the installer and the target.
#### Everything I Tried

- Manual partitioning with bootable flags
- Wiping the first 100-200MB with dd if=/dev/zero
- wipefs -a /dev/sda
- Force unmounting /cdrom in shell
- Manually creating filesystems before continuing
- Adding boot parameters for the ancient Intel GPU (nomodeset, vga=normal, debian-installer/framebuffer=false)
- Switching between stable and testing ISOs

Nothing fully worked. The installer kept getting confused about the codename/release because /cdrom was broken or remounted.

#### What Would Have Worked

- Using a non-hybrid full DVD ISO (no partition table conflict)
- Installing via Proxmox VM with physical disk passthrough
- Host-level debootstrap on the HDD from Proxmox (partition → format → chroot → install GRUB)

But... BIOS password blocked reliable USB booting, and I didn't want to keep swapping the drive.
CMOS battery removal? Tried but no luck. No jumper, no master password worked.

### Final Outcome: Abandon Mission
After way too many hours, I gave up.

- Wiped the 1TB drive again
- Put it back in the OptiPlex as storage
- The 250GB original HDD became a cold backup/external drive
- The LG laptop... went back to being a dusty relic

Lesson learned: Never dd a hybrid netinst ISO directly to your target internal HDD on **old locked-BIOS machines**.
If I ever get another old laptop (or somehow crack that BIOS password), I'll go the VM passthrough route—foolproof.

### Silver Lining
My main Proxmox node is running strong. 
Sometimes the win is knowing when to stop fighting.
Next up setting up services and more.

Thanks for reading – more homelab adventures coming soon. 

**Inshallah** this becomes a long series. The more I break, the more I’ll share.

Stick around if you want to learn together.

```bash{.dua-tawfik}
رَبِّ زِدْنِي عِلْمًا ۞ وَفَهْمًا وَحِكْمَةً
رَبِّ يَسِّرْ وَلَا تُعَسِّرْ ۞ وَتَمِّمْ بِالْخَيْرِ
```

#### ~ Darknight