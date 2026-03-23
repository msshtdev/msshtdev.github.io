---
# Common-Defined params
title: "Connecting to Wi-Fi LAN from Android without disabling WireGuard. But isn't there any better way?"
date: "2026-03-20T10:14:35+09:00"
description: "Connecting Wi-Fi LAN from Android device without disabling WireGuard"
categories:
  - "tech"
  - "network"
  - "vpn"
tags:
  - "tech"
  - "network"
  - "vpn"

# Theme-Defined params
thumbnail: "img/wireguard.svg" # Thumbnail image
lead: "Connecting Wi-Fi LAN from Android device without disabling WireGuard" # Lead text
authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "search"
  - "recent"
  - "taglist"
---


## Problem

Last week, I bought a new Pixel 9a. While setting up, I realized I couldn't connect to the Wi-Fi LAN with Wireguard Interface on.

After cursorily searching, I found it's almost impossible to modify the Android routing policy from a normal user unlike GNU/Linux.

But It's a bit cumbersome to turn on/off every time I want to connect to the LAN, so I took a different approach.

I aimed only IPv4 LAN network and used [the normal WireGuard app](https://download.wireguard.com/android-client/), and some other apps (e.g., Proton and Mullvad) may behave differently.

## TL;DR

1. Write following IPs to the AllowedIPs value in Wireguard conf
    ```
    64.0.0.0/2, 32.0.0.0/3, 128.0.0.0/3, 224.0.0.0/3, 176.0.0.0/4, 208.0.0.0/4, 16.0.0.0/4, 0.0.0.0/5, 200.0.0.0/5, 160.0.0.0/5, 168.0.0.0/6, 196.0.0.0/6, 12.0.0.0/6, 194.0.0.0/7, 174.0.0.0/7, 8.0.0.0/7, 193.0.0.0/8, 11.0.0.0/8, 173.0.0.0/8, 192.0.0.0/9, 172.128.0.0/9, 192.192.0.0/10, 172.64.0.0/10, 172.32.0.0/11, 192.128.0.0/11, 172.0.0.0/12, 192.176.0.0/12, 192.160.0.0/13, 192.172.0.0/14, 192.170.0.0/15, 192.169.0.0/16, ::/0 ,
    ```
2. Interface on, and don't forget to set `Block connections without VPN` to off
3. Bugger off!!

## Solution

I believe the reason I couldn’t connect to the LAN is:

1. The AllowedIPs setting in the WireGuard configuration was set to `0.0.0.0/0, ::/0`, which forces all traffic--including packets destined for the LAN--to be routed through the WireGuard peer.

2. On Android, it appears that routing control is limited. For example, a standard user cannot change route table priorities as you might on GNU/Linux.

Since we can’t modify the routing behavior (point 2), we need to adjust AllowedIPs to exclude LAN addresses.

WireGuard doesn’t support “invert matching,” so we have to explicitly calculate the CIDR ranges that bypass the LAN networks.

Doing this manually can be tedious, especially when multiple LAN networks are involved. To simplify the process, I wrote a [script](https://github.com/msshtdev/scripts/blob/main/network/vpn/allowedIPs.py).

The script takes multiple LAN CIDRs as input and computes the CIDR ranges that should be used for AllowedIPs.

If you want to bypass all [Private IPv4 addresses](https://en.wikipedia.org/wiki/Private_network#Private_IPv4_addresses) ranges (which is usually sufficient in most cases, I reckon), you can run it like this:

```
python3 allowedIPs.py "10.0.0.0/8" "172.16.0.0/12" "192.168.0.0/16"
```

It will output AllowedIPs similar to those shown in the Short Answer.

## Lastly

But, Isn't there any beeter way???

https://github.com/msshtdev/scripts/blob/main/network/vpn/allowedIPs.py
