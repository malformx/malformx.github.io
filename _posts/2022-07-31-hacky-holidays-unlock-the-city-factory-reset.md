---
title: Hacky Holidays - Unlock The City - Factory Reset
author: malformX
date: 2022-08-1 00:20:00 +0530
categories: [CTF, HackyHolidays]
tags: [CTF, B2R, FTP, SUID]
---

## GIST:

This is a writeup for challenge called Factory Reset from Deloitte hacky holidays, unlock the city. This challenge is a typical boot2root type of challenge. An IP was given. It was scanned for open ports. As the initial foothold, a FTP server with anonymous login was discovered. Also, the FTP server was vulnerable for directory traversal. Using the ability to upload the files to the target FTP folder, our own public SSH KEY was added to the user's `authorized_keys`. Thus, giving us SSH access. Further the privilege escalation can be done using insecure SUID binary, which gives us a root shell. This is a three part challenge.

## Part 1:

Part 1 covers the Initial Foothold. 

![image-20220711194355279](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711194355279.png)
_Challenge description_

We are assigned to the IP 10.6.0.100. By scanning the IP with Nmap, two open ports are detected as soon as the scan starts.  Port 21 (FTP) and Port 22 (ssh) are open. It can also be noted that, anonymous login is allowed in the FTP service. Upon interacting with the FTP service and observing the banner, FTP server information and the version can be known.

![image-20220711190531264](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711190531264.png)
_Output of: `nmap -sC -sV 10.6.0.100 -vv`_

Further, searchsploit  can be used to look if there is any known vulnerabilities resides in that particular service. But there isn't any.

![image-20220711190917519](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711190917519.png)
_Looking for exploit in searchsploit_

Normally, one can't access the directories above his current user directory in FTP. But this particular FTP service has directory traversal bug which allows access to directories that are outside the current directory.

> Note, there is another user called admin and there is a ssh dir.
{: .prompt-tip}

![image-20220711190712082](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711190712082.png)
_Interactive FTP shell_

With directory traversal, the root of the Linux file system can be accessed. This can be further enumerated by looking for the common files and directories such as `/var`,`/etc`,`crontabs`,`/proc`,`ssh dirs`,`history files`,`bash/zsh configs` and so on, which may contain sensitive data. Files can be downloaded using `get/mget` command. 



![image-20220711190738109](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711190738109.png)
_Accessing the root filesystem using Directory traversal_

The first flag was under `/var/backups/DATA/flag1.txt`. We can download the flag using `get` command and read our first flag.

![image-20220731221202395](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731221202395.png)
_Locating flag1.txt_

## Part 2:

Part 2 covers, getting a SSH shell on the box.

![image-20220731225257026](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731225257026.png)
_Part 2 description_

There was another user called admin. The directory also had a `.ssh` dir. But there was nothing in it. But there was bash_history present for admin user. Downloading and enumerating it reveals, that current user access right is set to the .ssh dir. 

![image-20220731224602477](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731224602477.png)
_Enumerating admin directory and reading history file_

`chmod 700` permits access to the current user only. Some other modes are specified below. 

![image-20220731225444129](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731225444129.png)
_Other chmod perms_


We can create a ssh key pair and copy our public key to `../admin/.ssh/authorized_keys`. By doing this, the SSH service can be accessed using our private SSH key. 

>  At this point, we have no idea that the ftp server is running by the admin user. But we can try and see if it works anyways.
{: .prompt-tip}

So i created a SSH keypair using `ssh-keygen` on my VM.  

![image-20220711191149903](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711191149903.png)
_Generating SSH key pair_

Then copied my public key to the admin user's .ssh directory and saved it as `authorized_keys` using the command `put ~/.ssh/id_rsa.pub ../admin/.ssh/authorized_keys`. 

![image-20220731221940988](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731221940988.png)
_Copying public SSH key to the authorized_keys file_

We can see it is copied successfully. So the ftp service must be running with admin user privileges. We can verify this once we get a SSH shell. After copying the public key, by using private key SSH service can be accessed. 

It gives the flag in it's banner, but i didn't note it at first. Found this flag after i got root only.

![image-20220711191440297](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711191440297.png)
_Logging in over SHH_

**ME: Getting past the flag**

<img src="https://i.imgflip.com/4xyjge.png" style="zoom: 33%;" />




Looking at the process tells that ftp service was indeed running as admin user.

![image-20220731225222812](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731225222812.png)
_Looking at the proccess and verifying our hunch_

## Part 3:

Part 3 is about doing privesc and getting root.

![image-20220731225314560](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731225314560.png)
_Part 3 description_

Root access can be gained easily if the bash_history of the admin user is read. It will have a command history which takes advantage of a SUID binary. But I'll cover the enumeration part and move on from there. We can use [Linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)  to auto enumerate the box. After executing Linpeas we will find a SUID binary in a intended way. Now that there is ssh access, the Linpeas script can be copied to the remote box using SFTP.
> When we get reverse shell instead of a fully interactive SSH shell, other methods can be used to get the enumeration script into the machine by using curl, wget, nc, python http server and so on.
{: .prompt-tip}

![image-20220711191517513](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711191517513.png)
_Copying file to the remote box using SFTP_


Once executed, it will output all sorts of information about the machine. While scrolling a bit down the SUID binary can be found. It will be highlighted, so that we won't miss it. For reading further about SUID/SGID binaries refer [here](https://www.redhat.com/sysadmin/suid-sgid-sticky-bit).

![image-20220711191610790](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711191610790.png)
_Linpeas output_

There is a cool repository called [GTFOBins](https://gtfobins.github.io/) and through that the command or procedure to take advantage of the [`find`](https://gtfobins.github.io/gtfobins/find/#suid) binary can be known.

Using the command `find . -exec /bin/bash -p \; -quit`  from GTFOBins, root access can be gained and root flag can be seen. 

![image-20220711191801243](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711191801243.png)
_Escalating to root_

After searching through the box, looking through env variables, looking for any encrypted files and cron jobs, i decided to cheat.

<img src="https://img.ifunny.co/images/29c042d3ed009d826732e4c50fd3bb4ddc79af4b59a0d1be4d1c8ad1aa0c677e_1.jpg" style="zoom:50%;" />

First, i got the recently modified date of the final flag.

![image-20220711191947050](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220711191947050.png)
_Getting the file modification timestamp_

I then used `find` to get the files that are modified roughly around that time. I tried to get all the modified files between 8:00 to 10:00. However, i only got the 1st flag which i already got from FTP and the final flag. Then i slighly increased the time range.

Command: `find / -type f -newermt "May 10 08:00" ! -newermt "May 10 10:00" 2>/dev/null`.

![image-20220731223651234](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731223651234.png)
_Using find to list files modified between two time period_

I thought, maybe it can be hidden somewhere  inside those files. So i started grepping for the flag format. But then also only two flags showd up. Again i increased the time range. This time i got the flag.

Command: `find / -type f -newermt "May 10 08:00" ! -newermt "May 10 11:00" 2>/dev/null | xargs grep "CTF{"`

![image-20220731224000139](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/image-20220731224000139.png)
_Getting all the flags_

![](https://c.tenor.com/CSfI99Vz8-cAAAAC/sleeping-clapping.gif)

After finding the flag on `/etc/motd` only i realized that the flag is on the banner and looked at it again ;(

Anyways, Here's a Gaming mouse for you.

![](/commons/posts/2022-07-31-hacky-holidays-unlock-the-city-factory-reset.assets/IMG_20220731_233410.jpg)
_Pro GamingMouse.jpg_

