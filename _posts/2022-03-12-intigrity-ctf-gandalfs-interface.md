---
title: IntigrityCTF - Gandalf's Interface
author: malformX
date: 2022-03-12 23:28:00 +0530
categories: [CTF, Intigrity]
tags: [Reversing, CTF, Android]
---

This is one of the simple challenges that i solved during intigrity 1337 CTF. 

![image-20220312214556212](/commons/posts/intigrity-gandalfs-interface/image-20220312214556212.png)

The challenge provided an apk file which we can download locally. Whenever dealing with apk files, i always decompile them to see if it uses kotlin or any other framework instead of typical java. 
![image-20220312215122152](/commons/posts/intigrity-gandalfs-interface/image-20220312215122152.png)

If kotlin is used, we would've seen a folder named `kotlin` and also in assets we would see some files or folders named `flutter`. But we don't see anything like that sort.
Unlike C/C++, java language runs on Java Virtual Machine (JVM). When compiled, java programs will be converted into java bytecode, which will then run on top of the JVM. But one interesting thing about this javabytecode is, it can easily converted back into it's source code. But yeah, if kotlin is used, it will obfuscate the code into some blob file or will extract it on memory when it is being run. But we don't have to go through all this for now.

To get the source code of the apk we can first convert it into a jar file using `dex2jar` and use `jd-gui` to view the source.

![image-20220312220217269](/commons/posts/intigrity-gandalfs-interface/image-20220312220217269.png)

Now we can browse through the code.

![image-20220312221447268](/commons/posts/intigrity-gandalfs-interface/image-20220312221447268.png)

`com` is mostly where all the packages start. And in subdirectory we see `example.webview`. So the app will probably run as `com.example.webview` process. We can try to run it in Genymotion emulator and further do a dynamic analysis, but this challenge can be solved with static analysis itself.

If we see the code, it is pretty much self explanatory. `Mainactivity` is the main fucntion here. It is the main class which will load all the sub classes and functions. We can see, on click event it calls `WebViewActivity`. And this event is binded to a button. So when clicking a button this class gets called. 

![image-20220312221748593](/commons/posts/intigrity-gandalfs-interface/image-20220312221748593.png)

We can explore what this does.

![image-20220312222000934](/commons/posts/intigrity-gandalfs-interface/image-20220312222000934.png)

It calls another class. Let's follow it up.

![image-20220312222102268](/commons/posts/intigrity-gandalfs-interface/image-20220312222102268.png)

This one is sort of got a  big loggic. And we see some pre defined objects and also something what looks like a JWT. We'll see what this is in a moment.

![image-20220312222717827](/commons/posts/intigrity-gandalfs-interface/image-20220312222717827.png)

Following up the code we see there is an `if` condition. And it checks if the input of `bitcoincrypt.getsomesalt` is equal to `AUTH_TOKEN`. We already seen `AUTH_TOKEN` before. If we convert it to ASCII, we see gibberish only.

![image-20220312223123919](/commons/posts/intigrity-gandalfs-interface/image-20220312223123919.png)

 If we can see how `getsomesalt` function works, we can reverse it and get our input.

![image-20220312225433202](/commons/posts/intigrity-gandalfs-interface/image-20220312225433202.png)

`getsomesalt` function gets two arguments. One is our input and another one is `salt` as byte array. We can easily recreate this function in python.

```python
def getsalt(inp):
  salt = [ord(i) for i in "RheO5PB6mfL5N3YBH45e5XuCEaWpvWUFESqTYnZk"]
  arrayOfByte2 = inp
  arrayOfByte1 = [0 for i in range(len(arrayOfByte2))]
  for b in range(len(arrayOfByte2)):
    arrayOfByte1[b] =  chr(arrayOfByte2[b] ^ salt[b % len(salt)])
  return "".join(arrayOfByte1)

AUTH_TOKEN = [38, 0, 0, 60, 93, 49, 50, 95, 25, 22, 45, 71, 47, 94, 60, 54, 45, 70]
print(getsalt(AUTH_TOKEN)) #theshapitparameter
```

After running the script we get `theshapitparameter` as our output.

![image-20220312230047037](/commons/posts/intigrity-gandalfs-interface/image-20220312230047037.png)

Now we see there is a string being built. First, one weird value is appended to the empty str array. This value is hardcoded constant. And there is a cool trick to get the value of this constant.

For this, we need convert the number from decimal to hex. hex for `2131689512` is `0x7f0f0028`. Then we can go to the `apktool` decompilation directory. Under that directory we need to follow `res/values` directory. In that directory there will be a file called `public.xml`. If we search for the hex value in this file we will get a string reference which is `groot`.

![image-20220312230651736](/commons/posts/intigrity-gandalfs-interface/image-20220312230651736.png)

Now we can grep for this string in `strings.xml` and we'll get the corresponding value.

![image-20220312230823097](/commons/posts/intigrity-gandalfs-interface/image-20220312230823097.png)

If we visit this URL it just tells us to check our location or params. Also something about DirectAccess.

![image-20220312232742115](/commons/posts/intigrity-gandalfs-interface/image-20220312232742115.png)

This URL is getting appended to out String first. Then it appends a question mark `(?)` to our string. Which means now we have `https://teambounters.com/shapa.php?`. 

Followed by that, we see the paramstring is getting appended. And we already found our param string value using `getsalt` function, which is `theshapitparameter`. Now our string becomes `https://teambounters.com/shapa.php?theshapitparameter`. Finally it appends `?=givemeflag`. So the final string is `https://teambounters.com/shapa.php?theshapitparameter?=givemeflag`.

![image-20220312231413208](/commons/posts/intigrity-gandalfs-interface/image-20220312231413208.png)

At this point it is obvious that a url is getting called. But if we call it in curl or browser, it is just same as before, when we opened the url in browser, without any params.

![image-20220312231627043](/commons/posts/intigrity-gandalfs-interface/image-20220312231627043.png)

The reason for this is, if we note, some HTTP headers are being set before the url is called.

![image-20220312231731275](/commons/posts/intigrity-gandalfs-interface/image-20220312231731275.png)

If `User-Agent` and `Accept-Language` is set to those specific value only it will succeed. `Accept-Language` seems pretty straight forward. But there is some other stuff going on with `User-Agent`. Like before, we can follow the code and reverse it to get the value.

![image-20220312232009123](/commons/posts/intigrity-gandalfs-interface/image-20220312232009123.png)

The function flow is nearly as same as the `getsalt` function here. So we can modify our python script to get the value again.

```python
def getheader():
  arrayOfByte1 = [ord(i) for i in "RheO5PB6mfL5N3YBH45e5XuCEaWpvWUsdgdedgdrddf"]
  arrayOfByte2 = [ord(i) for i in "Thisnewuseragent"]
  arrayOfByte3 = [0 for i in range(len(arrayOfByte2))]

  for b in range(len(arrayOfByte2)):
    arrayOfByte3[b] = chr(arrayOfByte2[b] & arrayOfByte1[b % len(arrayOfByte1)])
  return "".join(arrayOfByte3)

print(getheader()) #PhaC$@B4ad@!F!H@
```

After running the script we get the output `PhaC$@B4ad@!F!H@`. Now that we have everything we need, we can start requesting the url with all the required Headers.

![image-20220312232337407](/commons/posts/intigrity-gandalfs-interface/image-20220312232337407.png)

> Note i escaped special characters in `PhaC$@B4ad@!F!H@` because bash/zsh will misinterpret them.
{: .prompt-tip}

And that's it. We got the flag.