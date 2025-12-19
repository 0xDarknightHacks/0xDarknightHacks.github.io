---
title: My Homelabbing Journey Part 2
date: 2025-12-19
categories: [homelab,proxmox,selfhosting,networking]
tags: [journey,homelabbing,writeups,learning,notes]
---

# Reviving Ancient Routers: My Thrifting Wins and Building a DD-WRT Wireless Client Bridge
Hey folks! If you've been following my homelab journey, you know I'm obsessed with starting small, squeezing value out of old gear, and thrifting like it's an extreme sport. This time? I turned two dirt-cheap market finds into a **wireless client bridge** so I could finally move my homelab into my actual room – no cable runs across the house, no mom vetoes.

### How the Thrifting Addiction Started

I absolutely love Tunisian souks – those massive, chaotic markets where everything is used, super cheap, and usually a total gamble. The electronics corners? Pure dopamine.

One day, thrifting with friends, I spot a **Cisco logo** buried in a pile. I literally vaulted over stuff to grab it: a **Linksys E900** router. Quick inspection – powers on, looks good. Price? **7 TND** (about $2.20 USD). Instant buy.

That purchase unlocked something dangerous. I started hunting routers and switches obsessively – haggling hard (sorry vendors), cursing when prices crept over my tiny budget. I scored tons of D-Link junk, random Cisco bits, and then... jackpot: a **Linksys E1200** for roughly the same price, plus two Ethernet cables for pocket change.

No power adapters included, obviously. Scoured my room's electronics graveyard – nothing matched the 12V 0.5A spec. Risked a 9V adapter just to test: both routers lit up! Later bought proper chargers for ~30 TND total (under $10).

Hard resets, poked around the stock web interfaces... and then they sat collecting dust for months. Why? Classic procrastination + a dumb misconception: I thought flashing custom firmware required a USB port on the router.

### The Lightbulb Moment: DD-WRT Support Exists!

Fast-forward to literally a couple days ago. I finally researched these models properly. Heart racing: both the **E900** and **E1200 v2** are fully supported by **DD-WRT** (and potentially OpenWRT/Tomato too).

Why risk bricking perfectly good routers? One big reason:

My fiber ISP modem/router is locked down in the guest room. Running the homelab there meant cables everywhere, monitor + keyboard hassle, and zero privacy. Running a long Ethernet cable to my room? Not happening – impractical, expensive, and firmly in the "mom will say no" zone.

Solution: **Wireless Client Bridge** – connect wirelessly to the ISP Wi-Fi, output stable wired Internet to my OptiPlex homelab server.

Test subject: the E1200.

**Useful Resources**
- [DD-WRT Installation](https://wiki.dd-wrt.com/wiki/index.php/Installation)
- [DD-WRT Supported Devices](https://wiki.dd-wrt.com/wiki/index.php/Supported_Devices)
- [DD-WRT Supported Devices 802.11n - Linksys](https://wiki.dd-wrt.com/wiki/index.php/Supported_Devices_802.11n#Linksys)
- [Linksys E1200v2 DD-WRT Wiki](https://wiki.dd-wrt.com/wiki/index.php/Linksys_E1200v2)


**Always** check your exact hardware version and use the correct trailed build for the initial flash.
You can use the following DD-WRT build locations:
- [](https://ftp.dd-wrt.com/dd-wrtv2/downloads/betas/)
- [](https://download1.dd-wrt.com/dd-wrtv2/downloads/betas/)
- [](https://dd-wrt.com/support/other-downloads/?path=betas/)
Or you can simply follow the [Firmware FAQ](https://wiki.dd-wrt.com/wiki/index.php/Firmware_FAQ#Where_do_I_download_firmware.3F) on the DD-WRT website.

### Flashing DD-WRT: Sweaty Palms Edition

Connected via Ethernet to my laptop:

1. Performed a 30-30-30 hard reset.
2. Accessed stock Linksys interface → Firmware Upgrade.
3. Uploaded the **trailed initial build** ([19519 K2.6 mini e1200v2](https://download1.dd-wrt.com/dd-wrtv2/downloads/betas/2012/07-20-12-r19519/broadcom_K26/dd-wrt.v24-19519_NEWD-2_K2.6_mini-e1200v2.bin)).
4. Waited... rebooted... power cycled.
5. No setup page? Panic. Then another 30-30-30 reset → DD-WRT setup wizard appeared!
6. Set admin password.
7. Upgraded to a full featured build (-nv64k.bin).
8. More resets. Success!

### Configuring the Client Bridge (What Actually Worked For Me)

I followed dozens of guides and YouTube videos – most didn't work perfectly on my setup. This is the exact configuration that finally gave me stable wired Internet:

**Goal**: Connect wirelessly to ISP router → provide Internet only via LAN ports (no additional Wi-Fi access point).

1. **Setup → Basic Setup**
   - WAN Connection Type: **Disabled**
   - Router IP: Static IP in your main network but unused (e.g., 192.168.1.50)
   - Gateway & Local DNS: Your ISP router's IP (usually 192.168.1.1)
   - Assign WAN Port to Switch (gives you 5 LAN ports total)
   - DHCP Server: **Disabled**

2. **Setup → Advanced Routing**
   - Operating Mode: **Router** (not Gateway)

3. **Security → Firewall**
   - SPI Firewall: **Disable**

4. **Services → Services**
   - DNSMasq: **Disable**
   - ttraff Daemon: **Disable**

5. **Wireless → Basic Settings**
   - Wireless Mode: **Client Bridge**
   - Wireless Network Name (SSID): Exactly match your ISP Wi-Fi SSID

6. **Wireless → Wireless Security**
   - Match your ISP Wi-Fi security type, algorithm, and key

7. **Wireless → Advanced Settings**
   - Authentication Type: **Shared Key**

8. **Status → Wireless**
   - Scroll to bottom → Site Survey → Find your ISP network → **Join**

9. **Save** changes throughout (not Apply yet) → finally **Apply Settings**

10. Power cycle the router, remove the temporary Ethernet cable.

Test: Connect a device to one of the LAN ports → Internet works! You can also ping/access the DD-WRT web interface at its new static IP.

### Wrap-Up & Next Steps

My homelab is now happily running in my room, powered by a router that cost less than a coffee. Planning to flash the E900 as a backup or extender. I might even record a proper video tutorial soon.

Thrifting + open-source firmware = unbeatable combo.

Thanks for reading – more homelab adventures coming soon. 

**Inshallah** this becomes a long series. The more I break, the more I’ll share.

Stick around if you want to learn together.

```bash{.dua-tawfik}
رَبِّ زِدْنِي عِلْمًا ۞ وَفَهْمًا وَحِكْمَةً
رَبِّ يَسِّرْ وَلَا تُعَسِّرْ ۞ وَتَمِّمْ بِالْخَيْرِ
```

#### ~ Darknight