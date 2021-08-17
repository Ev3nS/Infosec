---
layout: post
title: Hackthebox - Archetype Walkthrough
published: true
---
This one was a classic, the room revolves around a misconfiguration in mssql that allows the user to enable_xp_cmdshell which opens a path to achieve remote code execution on the machine via xp_cmdshell.

## Information Gathering

### Starting with a port scan:
First we do a fast scan in order to identify what ports are up and running:
1. **nmap 10.10.10.27 -p- -Pn --min-rate 1500 -vvv -oN AllPorts.txt**

And after that we run a full scan on the ports that we have identified:
2. **nmap 10.10.10.27 -p 135,139,445 -sVC -O -v -oA FoundPorts.txt**

which gives us the following output:

> 
Nmap scan report for 10.10.10.27
Host is up (0.36s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
No exact OS matches for host.

We can look at the running services see that the running OS is Windows. Even more, it seems to be running a Microsoft Windows Server between version 2008 R2 and 2012 which are known to have quite a handful of exploits. Only in 2021 there have been [186 exploits](https://stack.watch/product/microsoft/windows-server-2008/#:~:text=In%202021%20there%20have%20been,had%20382%20security%20vulnerabilities%20published.&text=However%2C%20the%20average%20CVE%20base,2021%20is%20greater%20by%200.42.) with an average CVE of 7.9 out of 10.

## The SMB share

Having port 445 up hints me to connect to HTB Archetype using smb.

**smbclient -L 10.10.10.27**
> Sharename       Type      Comment
    ---------       ----      -------
    ADMIN$          Disk      Remote Admin
    backups         Disk
    C$              Disk      Default share
    IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available

Looks like we have found quite a few shares, even without using a password.
The most interesting share is "**backups**" which is also the only non-default share.

Using **smclient \\\\10.10.10.27\\backups** we can connect to this share.

Looks like we found an interesting config file **prod.dtsConfig**, we can download the file with: 
> get prod.dtsConfig
