---
title: Hacky Holidays - Protect The Supply
author: malformX
date: 2022-07-28 00:01:00 +0530
categories: [CTF, HackyHolidays]
tags: [Docker, Reversing, Ghidra, CTF]
---

![image-20220712131714334](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712131714334.png)
_Fig 1: The challenge description_

## <u>Gist</u>:

In this challenge, Initially a bad code was introduced to the docker image. Later after a while the bad code was removed from the instance. Our goal is to find, where is that malicious code and what that code did. Solving this challenge involves Identifying this piece and reversing the compiled C shared object.

## <u>Docker</u>:

We can start running the docker and explore around it a bit.

![image-20220712133649310](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712133649310.png)
_Fig 2: Running Docker_

We can also try to see the recently modified files, but nothing interesting shows up.

![image-20220727020748063](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727020748063.png)
_Fig 3: Finding recently modified files_

It just seems like a nice little image that looks fresh. Also, we can explore a bit more and go through important files and directories, see if anything shows up. But i'll save you the time that i lost miserably and get to the business directly. If we try hard and recall 2 seconds ago, we saw that the author used the word `change` 3 times in the description. Well, it turns out, that is a hint. 
> _Remember kids: Unlike me, always read the description._
{: .prompt-tip}

![](https://64.media.tumblr.com/e601cd9f8a0ff41797e3c06ce355d1c8/6461d05495bb4506-40/s540x810/728cd6c8b6ff9a6a92e848d56f7cc56e4d488f69.gif)

Apparently, dockers have history. Let's see what google has to say about this.

**"Docker images are constructed in layers, each layer corresponding to a first approximation to a line in a Dockerfile . `The history command shows these layers, and the commands used to create them`".**

Using the `docker history` command will show the changes that was committed in the image. As we can see there is a bunch of commits there. There is one interesting commit here which copies `/lib/x86_64-linux-gnu` into the docker machine. Also the commit ID is seems to be missing. When trying to locate the file, we can't seem to find that shared object file in the mentioned location. Because in the latest commit `sha256:fc6642d32b03e5c96e96299b64ca355bcfea30ef6ad4dec984f2601129210bbb` it removes that shared object file.

![image-20220712133915012](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712133915012.png)
_Fig 4: 'docker history' output_

![image-20220727023130604](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727023130604.png)
_Fig 5: Looking for libc.so.evil.6_

The command `docker inspect` can be further used to view the information about layers and other stuff. We can see the latest layer from the docker history is the current layer here. But we can't seem to locate the layer which was responsible for copying the shared object file. Matter of fact we can't find any other layer expect the current one.

![image-20220712133734356](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712133734356.png)
_Fig 7: `docker inspect` output_

![image-20220712133808476](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712133808476.png)
_Fig 8: `docker inspect` output_

We can save the whole docker locally using `docker save` command. It'll extract every layer and save it into the tar file. Once extracted we can see there is a bunch of other tar balls inside the parent archive.

![image-20220712134318791](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712134318791.png)
_Fig 9: Executing`docker save` and exploring the archive_

Using any archive viewer, we can explore the archive. And one of the archives has the shared object file.

![image-20220712134500820](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712134500820.png)
_Fig 10: Locating the libc.so.evil.6 in the archive_

We can verify that it indeed is a shared object. Looks like it a 64 bit shared object.

![image-20220712134540468](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712134540468.png)
_Fig 11: File Utility_

## <u>Reversing</u>:

We can try and execute it, as shared objects can also be directly executed on several occasions. But even though i'm in a VM, IM NOT GOING TO DO IT. 

![](https://c.tenor.com/Hin7A6Rp6LEAAAAC/sneaky-sneak.gif) 

Okay, i ended up doing it anyways.

![image-20220728012541715](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220728012541715.png)

We can see that, the binary is trying to execute some commands like apt. But my system doesn't have apt. Because..

<img src="https://i.kym-cdn.com/photos/images/masonry/002/243/395/bcd.jpg" style="zoom: 150%;" />

Further, the file can be analyzed in ghidra. After loading the shared object file in ghidra we see the file entrypoint. There is no main fucntion in the ghidra decompilation but main func can be tracked through the `__libc_start_main` as it is a fucntion call that always calls main func. 

![image-20220712140949593](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220712140949593.png)
_Fig 12: Locating main function in Ghidra_

Now, we're in the main function and lots of variables and weird strings are here. There a long string which looks like the cyclic pattern of pwntools assigned to `local_10` variable.

![image-20220727004521645](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727004521645.png)
_Fig 13: Observation of main function_

Going down, we see, there is a call to one function.

![image-20220727004601254](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727004601254.png)
_Fig 14: First function call inside main function_

This func executes some system commands like installing packages using apt and running linpeas and saving its output in the disk. But one interesting thing here is, it has a domain. The attacker echos a command string, which if executed, will append the nc reverse shell to the bash config. So, each time the victim openes his terminal, the attacker will get the shell. Uhmmm.. that's actually a cool way.

![image-20220727004631155](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727004631155.png)
_Fig 15: Malicious code piece_

![](https://memegenerator.net/img/instances/49677433.jpg) 

But the domain looks dead.

![image-20220727004703050](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727004703050.png)
_Fig 16: pinging the mal domain_

Following down, we see another func.

![image-20220727004816673](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727004816673.png)
_Fig 17: Location another function_

This func also trying to contact that dead domain. And seems like it exfiltrates the linpeas output using ICMP packets.

![image-20220727004829992](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727004829992.png)
_Fig 18: Data exfiltration_

Finally in the bottom, we see another function which takes a parameter.

![image-20220727005847603](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727005847603.png)
_Fig 19: Final fucntion_

This fucntion got some logic going on with it. It takes one argument and there is a loop running until the length of that argument is hit then each character in the arg is XORed with another data in a location. Now, we have already seen the parameter(local_10). So, `param_1[local_1c]`is a string array. And `local_1c + DAT_00104018` hints us that it might also be a typical C array as C arrays work with pointers. In C arrays, may it be a list or a string array, by adding a 4 byte integer value to the array pointer we access the address which contains the next element in the array.

![image-20220727005919487](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727005919487.png)
_Fig 20: Function logic_

As we can see the fucntion parameter `local_10` is the char array.

![image-20220727012210234](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727012210234.png)
_Fig 21: local_10 variable type_

And the data location is a list array. From the `malloc` size we can guess that the size of list array also is going to be 0xa8(168) in length. But `local_10` is 167 charcters long, but it doesn't matter as this first 167 characters are going to be XORed with only the first 167 elements of `DAT_00104018`. Because the while loop in the function stops, once the character count of `local_10` hits. 

[    `sVar1 = strlen(param_1);`
    `if (sVar1 <= (ulong)(long)local_1c) break;` ]

![image-20220727012316620](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727012316620.png)
_Fig 22: Size of DAT_00104018_

So we can simply use python to milk out a program that can do the XOR thing for us. We can copy all the list array definitions and put it in python.

![image-20220727012702933](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727012702933.png)
_Fig 23: Mirroring the XOR login in python_

> make sure to change this `*DAT_00104018 = 4;`  to `DAT_00104018[0] = 4;` as we're using python, we need to specify the initial element position in the list.
{: .prompt-tip}

After copying `local_10` and XOR the whole thing, we get the flag.

![image-20220727032541718](/commons/posts/2022-07-28-hacky-holidays-protect-the-supply/image-20220727032541718.png)
_Fig 24: FLAG_

## <u>Useful Materials</u>:

- [Docker config and Deployment Cheatsheet](https://hacksheets.in/all-categories/container-security-main/docker-cheatsheet/)
- [https://hacksheets.in/all-categories/container-security-main/docker-cheatsheet/](https://hacksheets.in/all-categories/container-security-main/docker-cheatsheet/)
- [https://book.hacktricks.xyz/network-services-pentesting/2375-pentesting-docker](https://book.hacktricks.xyz/network-services-pentesting/2375-pentesting-docker)
- [https://ghidra-sre.org/CheatSheet.html](https://ghidra-sre.org/CheatSheet.html)



