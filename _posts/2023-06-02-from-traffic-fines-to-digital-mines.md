---
title: From Traffic Fines to Digital Mines - Unleashing the secret files of Echallan
author: malformX
date: 2023-06-09 00:28:00 +0530
categories: [Bugs, Findings]
tags: [BUGBOUNTY, SQLI, WEB, INJECTION]
---

![back for once more](https://media.tenor.com/SwQvhUDvuC8AAAAC/adblocker-im-back.gif)

Heyo guys. I'm back from the hibernation one more time and back with a little story. A story of how i got the whole Database of parivahan Echallan for just 1000 rupees. It was a really nice deal.

## Story Time:
A couple of months ago, my goofy friends and I planned to go out for a movie. It was a fun day. After the movie, three of us went to have a cup of coffee. Let’s call my friends Vadai and Sadai. It was a rainy day. The 3 of us were sitting there enjoying the rain and joked about stuff. We only had one bike. Vadai and I shared the same route and so the ride as well. When the rain was calming down its cries, we decided to drop Sadai at the bus stop nearby. It seemed like triples was the optimal choice. So all three of us hopped in and headed towards the bus stop. 

Just near the bus stop there were a dozen of police personnel standing. It was still drizzling a little. So it took some time for us before noting their presence. Before we realised the moment, one of them came running towards us and grabbed our bike key and disappeared into thin rain drops. 

We sighed, pushed our bike to the side and got to the police. We thought we would get a chance to explain our side however, we got challan instead! challans first explanation later! We've got 1000rs fine. We tried to request them to maybe reduce it. But it was the least amount according to one of them.

We took the challan, got to our bike and just stood there in the rain and looked at each other for a brief period of time without uttering a word. We didn't have that much money. We used to forcefully collect coins from each other's pockets just to buy one egg puff. So 1000 was a big deal. Not to mention we already used up our quota on the movie.

Anyways, once more we just emptied our pockets and tried to pay the challan online, but our challan didn’t get updated in the E-Challan website at that time. After a day i  paid the fine. Fortunately my browser was running through burp the whole time.

Sometime later after the payment, i noticed some requests logged in the burp. I was already heart broken and was re-evaluating my life choices after paying 1k fine. With the heart as heavy as the Thor’s hammer, I starred dead into the eyes of my burp proxy window. I noted some unusual behavior there, which turned the nitro boost ON in the motors of my heart.

Enough with the rants and story, let's dive into the technical details now. 

## Detecting Injection:
The site didn’t have much requests to begin with. At first my point of interest was the payment receipt. It did have the receipt filename parameter. I tried to traverse the directory and maybe get a read of internal files. But all i got was some weird errors. I spent some time digging through it, trying to identify any server side issues but luck was laughing at me from a distance.

After spending some time with the sites along with the voices in my head, i noticed one other interesting endpoint. When paying the challan, it’ll ask for a mobile number to send an OTP. Later when the OTP got entered, the respective challan number will get sent along with the OTP in a json array, which will be verified on the server side.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled.png)
_Fig 1: Normal request_

Frustrated me, put a single quote in the challn_no parameter, hoping to not get anything even remotely interesting. But to my surprise, the server cried it’s heart out to me. The SQL query was partially errored out too.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%201.png)
_Fig 2: Tring SQLi_

Now my heart was pumping curiosity to my fingers and brain. I became the happie happie cat for real!

![happiehappie](https://media.tenor.com/arqlNu8gyJYAAAAM/cat-cat-jumping.gif)

I tried to see if otp param is also vulnerable. It was vulnerable indeed. touché!

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%202.png)
_Fig 3: Checking OTP param for SQLi_

I tried some random stuff in hopes of getting a bit more knowledge about the structure of the query. At some point i saw that `WHERE` clause is used and the columns are checked inside brackets. Of course even if i didn't get the structure of the query i would have eventually checked for brackets at this point. 

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%203.png)
_Fig 4: Identifying the structure_

So now all i had to do was close the bracks and comment out the rest to complete the query and the server no longer errored.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%204.png)
_Fig 5: Completing the full query_

For some time i tried to guess how things are configured. For non existent challan it said invalid otp. The challan_no is checked before otp.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%205.png)
_Fig 6: Playing with the query_

Now the injection point is really simple. I can append the query after brackets. If i get “Your OTP is successfully verified.” in the response, then the query is successful.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%206.png)
_Fig 7: Successful response_

Or if i get “invalid otp” the query is failed.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%207.png)
_Fig 8: Unsuccessful response_

We can use the above information to extract the db using true or false (conditional sqli) condition. But the server is already erroring. So i thought why not use the error based injection to extract info. Maybe I should've followed the traditional flow for SQLi.

As i didn’t know the db used in the backend. I just started with the default mysql extractvalue() function to error out the query response. But i got some weird error. 

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%208.png)
_Fig 9: Trying to use extarctvalue() to extract DB_

I listened to the server's complaint and tried to remove the hex character. Now it told me, there is no function named database(). huhh!!!

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%209.png)
_Fig 10: Non existent database() function_

Okay vro! i will just select an integer. What do you got to say now? It said, there are no function named extractvalue() at all.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2010.png)
_Fig 11: Non-existent extractvalue()_

Now i understood, this isn’t mysql at all. I randomly tried functions from different db vendors.

sqlite? : nope!

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2011.png)
_Fig 12: Trying sqlite function_

mssql? : nope!

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2012.png)
_Fig 13: Trying mssql function_

now now! the version() function seems to exist. And i got a crucial information saying it’s output is of type text.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2013.png)
_Fig 14: Finding the function that exist_

Now let’s try a conditional check. The version is probably not going to be ‘1’ in any case. So return true if version not eq 1, should obviously return true. And it returned true. But now, version() exist both in mysql and postgres. We already know that extractvalue() func didn’t exist in our server. It exist only on mysql. This thought made me conclude that i'm dealing with postgres.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2014.png)
_Fig 15: Concluding the DB to be Postgres_

I can verify this conclusions with other postgres functions.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2015.png)
_Fig 16: Confirming it is Postgres_

In postgres there exist a common function CAST() which is used to convert types of a data. And when i tried to convert an unconvertible data type it died with an error. I can use this function to our benefit. If I try to cast a sql query’s output as an int it’ll error the output.

Let’s try this out.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2016.png)
_Fig 17: Successfully extracting information from the DB_

As seen above, now i was able to extract query output. It is also confirmed that postgres is our db.

At this point i hacked a little python script to extract query of my choice using python. I was able to see DL numbers of users, access cctv challans and what not.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2017.png)
_Fig 18: Extraction using python script_

But the story doesn’t end here. In postgres there are some functions which can be used to list and read dirs and files. I was able to get nearly all of the logins of traffic police. With this one could’ve logged in as any traffic police and remove any Echallan of anyone or put fine to any vehicle.

![Untitled](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/Untitled%2018.png)
_Fig 19: File_Read using SQLi_

So in conclusion i held the whole database of traffic challan website for the cost of 1000rs.

![](https://media.tenor.com/2GBd-X60Tl8AAAAd/ramdev-baba-to-indian-government-pawpaw.gif)

## Timeline

> I've reported it on the end of march. They fixed the vulnerabilty later on. I kept asking them for any update, but i haven't got any so far other than the auto generated mail.
{: .prompt-tip}

![](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/2023-06-12_15-55.png)
_Fig 20: Overall Timeline_

I also got a incident id from CERT-IN, but don't know what they meant by that. They treated it as incident i guess.

![](/commons/posts/2023-06-02-from-traffic-fines-to-digital-mines.assets/2023-06-12_15-57.png)
_Fig 21: CERT-IN Incident ID_