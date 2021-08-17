---
layout: post
title: Hackthebox - Archetype Walkthrough
published: true
---
This one was a classic, the room revolves around a misconfiguration in mssql that allows the user to enable_xp_cmdshell which enables a way to achieve remote code execution on the machinethrough xp_cmdshell.

## Information Gathering

### Starting with a port scan:
First we do a fast scan in order to identify what ports are up and running:
1. **nmap 10.10.10.27 -p- -Pn --min-rate 1500 -vvv -oN AllPorts.txt**

And after that we run a full scan on the ports that we have identified:
2. **nmap 10.10.10.27 -p 135,139,445 -sVC -O -v -oA FoundPorts.txt**

which gives us the following output:

Code:
Nmap scan report for 10.10.10.27
Host is up (0.36s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
No exact OS matches for host.

We can look at the running services see that the running OS is Windows. Even more, it seems to be running a Microsoft Windows Server between version 2008 R2 and 2012 which are known to have over 186 exploits by 2021 with an average CVE of 7.9 out of 10.

## The SMB share

Having port 445 up hints me to connect to HTB Archetype using smb.

> smbclient -L 10.10.10.27

Sharename       Type      Comment
    ---------       ----      -------
    ADMIN$          Disk      Remote Admin
    backups         Disk
    C$              Disk      Default share
    IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available

Looks like we have found quite a few shares, even without using a password.
The most interesting share is "backups" which is also the only non-default share.

Using **smclient \\\\10.10.10.27\\backups** we can connect to this share.

Looks like we found an interesting config file **prod.dtsConfig**, we can download the file with: 
> get prod.dtsConfig