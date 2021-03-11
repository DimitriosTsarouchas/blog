---
title: Local File Inclusion CTF
author: Dimitrios Tsarouchas
date: 2021-02-02 20:55:00 +0800
categories: [CTFs, TryHackMe]
tags: [ctf, tryhackme, linux]
pin: true
featured-image: /assets/img/lfi-2.jpg
featured-image-alt: LFI
---

![Local File Inclusion](/assets/img/lfi-2.jpg)

## Introduction

In this article we are going to capture root flag on a Linux machine using Local File Inclusion vulnerability. We have shown in [previous article](/posts/Local-File-Inclusion/) that an attacker gets advantage of improper input sanitisation by changing the value of a parameter in the website's URL. 

## Exploring the website to find ways for LFI attack

Opening the given ip address using Firefox we are directed to Falcon's website home page where we can see three articles been posted. We are looking for a parameter that we can modify to gain access to `/etc/passwd` file. 

![Falconfeast website](/assets/img/LFI/falconfeastwebsitemainpage.png)
*falconfeast's website home page*

By clicking on view details on the "Hacking this world" post and checking the URL address bar we can find the `name` parameter along with its value.

![Parameter name](/assets/img/LFI/Parametername.png)
*Parameter name seem to be vulnerable to LFI*

## Searching for users in the /etc/passwd file

Now that we found the variable `name` we will try to check if it is vulneable to LFI attack by passing the value of `../../../../etc/passwd`. 

And....BOOM!!!

The `/etc/passwd` file is now visible to us showing all the users that live inside the system. One user that stands out and might be of interest of further investigation is the *flaconfeast* account. Another valuable information displayed here is the next column that might probably indicate the password of that account (*falconfeast:rootpassword*). 

We can also get the password hash from the `/etc/shadow` file by passing the value of `../../../../etc/shadow` and crack it using Hashcat. 

![Found user falconfeast](/assets/img/LFI/Founduserfalconfeast.png)
*Found user falconfeast in /etc/passwd file and a password candidate*

![Falconfeast hash](/assets/img/LFI/passwordhash.png)
*Found falcnfeast's password hash in /etc/shadow file*

## Login via SSH as falconfeast

We need to find the user flag so we login via SSH as falconfeast user using the password of rootpassword. If it fail, that would mean we have to crack the obtained password hash because the password (*rootpassword*) was incorrect.

```terminal
#ssh falconfeast@10.10.173.178                  
falconfeast@inclusion:~$ pwd
/home/falconfeast
falconfeast@inclusion:~$ ls -l
total 8
drwxr-xr-x 2 root        root        4096 Jan 21  2020 articles
-rw-r--r-- 1 falconfeast falconfeast   21 Jan 22  2020 user.txt
falconfeast@inclusion:~$ cat user.txt 
```

## Privilege Escalation

The password (rootpassword) found at `/etc/passwd` gave us access to flaconfeast's account and we are now logged into the system with user privileges. Our next goal is to escalate our privileges for gaining root access and find the root flag. 

We need to find a vector such as a binary that falconfeast can execute as superuser and exploit it to gain root access. By using `sudo -l` we can check if falconfeast is able to execute any command as a superuser in that machine.

![Sudo -l](/assets/img/LFI/sudo-l.png)
*What commands falconfeast can execute as superuser?*

As we can see falconfeast can execute as the root user the `/usr/bin/socat` command. 

Lets visit [GTFOBins](https://gtfobins.github.io/gtfobins/socat/#sudo){:target="_blank"} website and find a way to exploit socat.

![GTFOBins](/assets/img/LFI/gtfo.png)
*GTFOBins sudo socat*

Now we need to run the command that will give us root access on the system and will find the root flag.

```terminal
falconfeast@inclusion:~$ sudo socat stdin exec:/bin/sh
whoami
root
pwd
/home/falconfeast
cd /root
ls -l
total 4
-rw-r--r-- 1 root root 21 Jan 22  2020 root.txt
cat root.txt
```