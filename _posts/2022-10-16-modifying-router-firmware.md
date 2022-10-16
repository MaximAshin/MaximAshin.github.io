---
layout: post
title: 'Modifying Router Firmware - Linksys WRT160N V3'
date: 2022-10-16
categories: exploitation
---
# Preface

There is a point in a man's (or woman's) life where he wakes up and decides he needs to learn some embedded, or hardware related cyber.
My friend had an old router collection that he found laying around, and I knew it was the right time.

# Goal
My goal is simple:
Find an RCE vulnerabillity (post auth), and pop a shell on the device.

# Attack Surface
### NMAP
I started with enumerating all of the open ports on the device with `nmap`.
This is the scan result:
```
Nmap scan report for 192.168.1.1
Host is up (0.034s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
80/tcp    open  http    Tomato WAP firmware httpd (Linksys WRT160Nv3 WAP)
|_http-title: 401 Unauthorized
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=WRT160Nv3
|_http-server-header: httpd
1780/tcp  open  http    Cisco DPC3828S WiFi cable modem
10080/tcp open  amanda?
Service Info: OS: Linux; Device: WAP; CPE: cpe:/o:linux:linux_kernel, cpe:/h:cisco:dpc3828s
```

I'll be short - after looking into each of the running services, I couldn't find anything interesting or helpful.
The best option was to explore the httpd server and the web pages it was serving to me.

### Web Server
This is the main page you get when you try to access the router:


### Firmware & Sources
I have to say, finding the right firmware was a cancerous journey.
Linksys had taken down many of the previously availible firmwares,
and even though I tried looking into the Wayback Machine - I was stuck with the sources only.

At first, you might think to yourself "Wow! I've got all of the sources I need and I can explore everything I need!".
Well, you'll be partially right, because having the sources does help.
But it doesn't always tell you to what architecture the firmware was compiled to,
and it also may have some patches that the developers applied - Which you'll have non of.

Luckily enough, I eventually found the right firmware in some [forum](https://forum.dd-wrt.com/phpBB2/viewtopic.php?p=654578).
