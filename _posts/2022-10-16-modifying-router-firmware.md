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
<kbd>![Main Page](https://user-images.githubusercontent.com/53023744/196054477-ca2a2870-7fa0-43ce-a4c4-577538c0d536.png)</kbd>

Shortly, this is what I've tried:
* Searching for classic command injections in the ping/traceroute/etc interfaces.
  * The httpd server uses fork(), which behind the scenes calls execve().
    * How do i know that? We'll see later.
  * This means that we can't inject commands, unless we find a way to manipulate the binaries in our favor (basically a LOLBIN).
* Looking into Elon Gliksberg's [article](https://elongl.github.io/exploitation/2021/05/30/pwning-home-router.html) about a similar product - Linksys WRT54G.
  * I've found that in my router's firmware the bug via the language interface was patched.

### Firmware & Sources
I have to say, finding the right firmware was a cancerous journey.
Linksys had taken down many of the previously available firmwares from their site.
Even though I tried looking into the Wayback Machine - I was stuck with the sources only (which I did find!).

At first, you might think to yourself:
"Wow! I've got all of the sources I need and I can explore everything I need!".
Well, you'll be partially right, because having the sources **does** help.
But it doesn't always tell you to what architecture the firmware was compiled to,
and it also may have some patches that you won't.

Luckily enough, I eventually found the right firmware in some [forum](https://forum.dd-wrt.com/phpBB2/viewtopic.php?p=654578).
You can dowload it here if the site is down: [WRT160Nv3_0_03_003.zip](https://github.com/MaximAshin/MaximAshin.github.io/files/9795539/WRT160Nv3_0_03_003.zip)

### msfvenom
msfvenom is a handy utility that allows us to create binary files via arguments.
In our case, I searched for a mipsel bind shell that we could upload via the modified firmware:

```msfvenom -l payloads | grep mipsle```

<kbd>![image](https://user-images.githubusercontent.com/53023744/196055645-ef0680da-0e41-45f0-8bd2-2f577fc7628f.png)</kbd>

Right after, I created the binary itself like so:

```msfvenom -p linux/mipsle/shell_bind_tcp LPORT=50505 -f elf > bs_backdoor```


### Victory
As we can see, index.asp indeed had my addition:

<kbd>![image](https://user-images.githubusercontent.com/53023744/196055310-483b7831-5482-4cb8-820e-e62346944ac8.png)</kbd>

I quickly fired up Metasploit's listener and got a connection:

<kbd>![image](https://user-images.githubusercontent.com/53023744/196055470-b6343f0c-cf0c-4b13-a3a1-dd64ca53b435.png)</kbd>


I have a shell! :)
