---
title: Hacky Holidays - Unlock The City - Stop The Heist
author: malformX
date: 2022-07-29 02:0:00 +0530
categories: [CTF, HackyHolidays]
tags: [CTF, IR, Wireshark, Windows]
---

![image-20220728125234500](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728125234500.png)
_Fig 1: Challenge description_


## Gist:

This is an Incident Response challenge. There is a windows system which was exploited by an attacker and we need to trace back what happend and what kind of data was compromised. The zip file contains some important files like logs and powershell history. The pcapng contains the network capture of the system. I didn't even had to open the json file, but seems like it was generated from splunk, but I'm not sure. And the rockyou.txt is a wordlist which we can use to brute force the hashes. This is a three part challenge and each part will help going to the next part of the challenge. So let's start analyzing the data.

## PART 1:

First, let's download all the files that are provided.

![image-20220728131020441](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728131020441.png)
_Fig 2: File list_

Network logs are a great place to start looking. We can use wireshark and see what was going on the system. This is a huge capture. It was roughly 138 MB large and it had hell a lot of packets.

![image-20220728131118224](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728131118224.png)
_Fig 3: Opening pcapng in Wireshark_

With wireshark we can use Protocol Hierarchy option from Statistics menu to see what are all the protocols exist in the capture. Protocol Hierarchy tab is presenting us with all the protocols and other information. We can see HTTP protocol is used in the wireshark. We can also see other protocols like SMB and ssh along with some other protocols.

![image-20220728131445102](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728131445102.png)
_Fig 4: Wiresahrk Protocol Higherarchy_

We do have some HTTP traffic. We can filter the http packets and explore.

![image-20220728141408845](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728141408845.png)
_Fig 5: HTTP traffic_

If we open a packet and observe the `User-Agent`, it says `WinRM Client`. So this is a WinRM traffic.

**"Windows Remote Management (WinRM) is the Microsoft implementation of [WS-Management Protocol](https://docs.microsoft.com/en-us/windows/win32/winrm/ws-management-protocol), a standard Simple Object Access Protocol (SOAP)-based, firewall-friendly protocol that allows hardware and operating systems, from different vendors, to interoperate."**

But unfortunately, we can't see what kind of commands are executed through WinRM session because, it is encrypted using kerberos. If we get the kerberos password we may possibly decrypt this encrypted session. But this challenge won't need to take that path.

![image-20220728141608422](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728141608422.png)
_Fig 6: Encrypted WinRM data_

We also have SMB protocol on the capture. SMB is a juicy place to look at, as it can contain any sensitive file transfer logs. By using `smb2` filter on wireshark we can see the smb data.(`smb2` filter will filter out SMB v2 packets and `smb` filter will filter out SMB v1 packets. I tried with `smb` filter and didn't find anything meaningfull so i skipped that part.) In the filter we can already see some files are transfered.

![image-20220728135627263](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728135627263.png)
_Fig 7: SMB traffic_

Wireshark has an Option called `Export Objects` under File menu. From there we can export files transfered using protocols like HTTP,SMB,FTP and etc. We can use this option to see all the files that are transfered using SMB and save them on the disk.

![image-20220728135713803](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728135713803.png)
_Fig 8: Exporting SMB objects_

There were quiet some files were exported. But nothing had any meaningful data. The Powershell Transcript file had some command logs, but they were not leading anywhere.

![image-20220728135819086](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728135819086.png)
_Fig 9: Exploring exported files_

The File_Id_* contain one command. The command downloads a powershell script named update.ps1 and run it. Now we do have a domain.

![image-20220728140441422](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728140441422.png)
_Fig 10: Contents of File_Id\_*_

Upon trying to get the file in hopes of examining it, it returns 404.

![image-20220728140549175](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728140549175.png)
_Fig 11: Trying to get update.ps1_

But the domain is active anyways.

![image-20220728140629199](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728140629199.png)
_Fig 12: Checking if whether the domain is active or not_

We can get the IP of this domain and search in wireshark to see if there is any network logs from this domain.

![image-20220728141232752](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728141232752.png)
_Fig 13: Getting the IP of the domain_

But it only had HTTPS packets. And there is no other information we can use from this domain. I looked for any certificate transfer in the capture and also in the zip file so that i can use it to decrypt the HTTPS traffic, but i couldn't find any.

![image-20220728141253294](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728141253294.png)
_Fig 14: HTTPS traffic_

So after a while, i got into enumerating the zip file.

![image-20220728141747220](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728141747220.png)
_Fig 15: Zip file contents_

It was a minimal C volume. It contained some crucial files.

![image-20220728141854416](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728141854416.png)
_Fig 16: Zip file contents_

The users from the User directory had powershell History.

![image-20220728141928725](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728141928725.png)
_Fig 17: Powershell History_

The administrator's powershell history had some group management stuff. 

![image-20220728142158190](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728142158190.png)
_Fig 18: Administrator command history_

That obfuscated string is an [AMSI](https://docs.microsoft.com/en-us/windows/win32/amsi/antimalware-scan-interface-portal) bypass payload. So most probably this is executed by the attacker. Other than this along with some other defender disabling there is nothing interesting here.

![](https://www.zdnet.com/a/img/resize/ff4121453007fe4b5c3328646c90c9633ef884d3/2021/06/01/32459c1f-8d87-4906-8309-386794db9042/screenshot-2021-06-01-at-16-13-35.png?auto=webp&width=1200)

The `unlockthecity` user also has a console history. Here we can see the powershell script getting executed here. Now only an idea bulb lit over my head. We can see that the powershell script is never getting downloaded to the disk. It straightly runs on the memory. So it is a file-less stuff. But, still the powershell script must have a set of commands that it must execute. And this history must be in the event log. 

![image-20220728225401574](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728225401574.png)
_Fig 19: unlockthecity powershell history_

Now, i went and saw if the event log directory exist. Luckily, it was there.

![image-20220728225634580](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728225634580.png)
_Fig 20: checking the event log directory_

Powershell will save all the command history in *-Powershell-Operational.evtx file. 

![image-20220728225654848](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728225654848.png)
_Fig 21: Locating powershell event log_

We can extract it and verify that it indeed is a event log.

![image-20220728225747599](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728225747599.png)
_Fig 22: Extracting the powershell event log_

But however it won't contain any plain data. We need to parse it first.

![image-20220728225831585](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728225831585.png)
_Fig 23: Trying to get the contents_

There are lot of tools for parsing windows event logs. But i find [Evtx Dump](https://github.com/omerbenamram/evtx) convenient. We can use this to parse the log file. After successful parsing, we see plain data. However there was so much data in this logfile. It would take a lot of time to go through all this unless we think smart.

![image-20220728230157159](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728230157159.png)
_Fig 24: Using evtx-dump util_

If we remember from Fig 10 before, we saw the command start time in the beginning from one file on SMB.

![image-20220728235709562](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728235709562.png)
_Fig 25: Getting 'command start time' from SMB export_

We do have the timestamp(SystemTime) of when a command event was run, and also the executed command, in the event log.

![image-20220729010633013](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729010633013.png)
_Fig 26: Identifying SystemTime from the parsed event log_

We can take the Command Start time from that file and format it correctly like in the event log. So `20220715125925` becomes `2022-07-15T12:59:25`.

![image-20220728235827886](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728235827886.png)
_Fig 27: Adapting the time format_

Now, using jq command we can get the events that are started around the target time and get the data we want.

`./evtx_dump-v0.7.2-x86_64-unknown-linux-gnu --dont-show-record-number -o json Microsoft-Windows-PowerShell%4Operational.evtx | jq -r '.Event|select(.System.TimeCreated[].SystemTime|contains("2022-07-15T12:59:25"))|.EventData.Payload'`

![image-20220728235911756](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728235911756.png)
_Fig 28: Extracting the commands executed at the target time_

That results in the first FLAG. 

## PART 2:

For the second flag we need to figure out the files that was taken by the attacker. 

![image-20220728235933513](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220728235933513.png)
_Fig 29: Part 2 description_

From the above image we can see a Powershell reverseshell is executed on the system. It has the attacker IP and PORT. Luckily it is not an encrypted reverse shell. So we can see the data traffic bright and clear.

![image-20220729011228909](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729011228909.png)
_Fig 30: Identifying attacker reverseshell IP and PORT_

Again, we can go to wireshark and filter the traffic that was transmitted on PORT 4444 by the IP 192.168.117.157.

![image-20220729000042829](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729000042829.png)
_Fig 31: Filtering the IP and PORT in wireshark_

We can Follow the stream as TCP stream by right clicking on the first packet and choosing TCP Stream under Follow menu. Then we'll get a nice TCP stream on that session. We can already see all the command executed on that reverse shell session. 

> Note:  mimikatz is downloaded and the output is saved in C:\Temp after executing it.
{: .prompt-tip}

![image-20220729000115480](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729000115480.png)
_Fig 32: Reverseshell session_

If we follow down we can see some rtf file contents are getting posted to a HTTP file server on attacker machine. 

![image-20220729000145934](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729000145934.png)
_Fig 33: File transfer through reverse shell session_

We can go and filter this HTTP port in wireshark. We can also see the GET request to mimikatz. But the file contents doesn't have anything interesting.

![image-20220729012138107](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729012138107.png)
_Fig 34: Filtering the file server port in wireshark_

After spending some quality time, it finally appeared to my eyes. It was the files itself. Their names. Their POST req order was the FLAG. 

![image-20220729000247468](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729000247468.png)
_Fig 35: Finding the flag pattern through transfered file names_

Come to think of it, the challenge description stated that i have to find the exfilterated files only.

> READ THE FRIGGIN DESCRIPTION MANN.
{: .prompt-warning}

After getting all the file names, we get the flag. `CTF{EXFILTRATE_ALL_THE_FILES}`

## PART 3:

Now we need to find the hash and crack it.

![image-20220729000353753](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729000353753.png)
_Fig 36: Part 3 description_

If we remember, mimikatz was executed and the output was saved under C:\Temp. If we visit this location in the zip file and open the dump.txt we can see the hash there.

![image-20220729001020553](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729001020553.png)
_Fig 37: Getting the contents of mimikatz dump_

We already has the format for the hash value from the description. We can use the combinator attack and some hashcat rule to achieve this password format. For cracking NTLM hashes we need to use mode 1000 in hashcat.

`hashcat -m 1000 hashy.txt -a 1 -j '^{ ^F ^T ^C $_' -k '$! $}' rockyou.txt rockyou.txt` 

I tried to crack it in my system. But my potato PC was starting to go nutz. Also i can't wait 19 days.

![image-20220729013754960](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220729013754960.png)
_Fig 38: POTATOO PCC_

So i used [colabcat](https://github.com/someshkar/colabcat). If anyone is poor like me, use it.

![image-20220719093820926](/commons/posts/2022-07-28-hacky-holidays-unlock-the-city-stoptheheist.assets/image-20220719093820926.png)
_Fig 39: Cracking the hash using colabcat_

Finally we get the last FLAG. After this i used to password to try and decrypt the WinRM traffic, but i couldn't get any results.

## Useful Materials

- [https://lzone.de/cheat-sheet/jq](https://lzone.de/cheat-sheet/jq) 
- [Wireshark Filters](https://gist.github.com/githubfoam/08efac0343f98bd727caa32e6c81f655)
- [https://hashcat.net/wiki/doku.php?id=example_hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)