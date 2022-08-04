---
title: RaspberryPi
date: 2022-08-04 12:10:01 -500
categories: [homelab, hardware, raspberrypi]
tags: [servers, hardware, network, nutanix, rack, raspberrypi]
---

# RaspberryPi
Unless you've been living under a rock I'm sure you've heard about these fantastic SBCs. Not only are they energy efficient, fun, extremely hard to find at this time due to global chip shortages they have millions of uses. In the homelab space you can use your imagination and save $ on your electric bill running services like VPN, DNS, remote KMM.

# My Homelab Pis
* internetpi - monitoring my Internet connection
* ntppi - NTP with USB GPS receiver
* backup - syncing my NAS to backup NAS, Github repos. You need a solid backup strategy
* vpnpi - wireguard VPN access to my homelab while on the road
* pikvm - allow remote access to KMM switch for IP console, out-of-band bios, recovery, booting ISOs over network
* homeassitant - not really homelab but keeps my smarthome and wife happy. Looking to upgrade to Homeassistant Yellow as soon as they start shipping.

# Pi Software
I'm running Raspbian OS slowly moving from 32bit to 64bit. Almost every one has Docker and I will go into more detail about which containers are being used on which.


![img-description](https://images.prismic.io/rpf-products/d3bbc517-f9e1-42eb-816d-9b2bc10ad814_Pi%204.jpg?ixlib=gatsbyFP&auto=compress%2Cformat&fit=max&w=1247&h=831) RaspberryPi-4
