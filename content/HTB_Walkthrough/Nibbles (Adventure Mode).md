
# 1. High Level Summary:

<b>Box Name </b>: Nibbles

<b>Level</b>: Easy

<b>About</b>: Nibbles is a fairly simple machine, however with the inclusion of a login blacklist, it is a fair bit more challenging to find valid credentials. Luckily, a username can be enumerated and guessing the correct password does not take long for most.

**Target IP** : 10.129.254.12

**Attack IP**: 10.10.16.24



# 2. Enumeration
### 2.1 Service Scanning

To identify the attack surface, I performed a full port scan of the target.
Command: 'nmap -sV -sS -sC -Pn 10.129.197.242 -p- --min-rate 500'
- -sV /-sC : Service version detection and default script scanning
- -Pn : Skips host discovery (treats hosts as online)
- -p- : scans all 65,535 TCP ports

![[Pasted image 20260529152005.png]]

The scan revealed two fully open TCP ports, a large contingent of closed ports, and a selective list of filtered ports:

1. **Port 22/tcp (Open - SSH):** Identifies as `OpenSSH 7.2p2 Ubuntu 4ubuntu2.2`. The service banner discloses that the underlying operating system is an Ubuntu Linux distribution. Cryptographic host keys were successfully gathered for RSA, ECDSA, and ED25519.
    
2. **Port 80/tcp (Open - HTTP):** Identifies as `Apache httpd 2.4.18 ((Ubuntu))`. The default script engine noted that the webpage lacks a structural HTML title (`Site doesn't have a title`). This marks port 80 as the primary vector for web application enumeration.
    
3. **Closed Ports:** A total of 998 ports responded with a TCP `RST` reset packet, identifying them as closed.

### 2.2 Web Enumeration 

### 2.2.1 Initial Web Inspection

Navigating to the Target Machine's IP address of 10.129.175.46 resulted in a static landing page with two words "Hello world!" as follows:

![[Pasted image 20260529152328.png]]

The lack of visible navigation links, forms, or applications suggests that the actual attack surface is hidden. This requires automated directory and file brute-forcing to uncover unlinked assets. 

### 2.2.2 Directory and File Enumeration (Gobuster)

**Command:** gobuster dir -u http://10.129.254.12 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt

**Findings:** The scan only revealed standard Apache configuration files (`.htaccess`, `.htpasswd`) and `/server-status`, all returning `403 Forbidden` status codes. The only `200 OK` response was `/index.html` (size: 93 bytes), confirming no hidden directories or files exist within the `common.txt` scope.

![[Pasted image 20260529152541.png]]

### 2.2.3 Virtual Host Enumeration (Gobuster VHOST)

**Command:** `gobuster vhost -u http://10.129.175.46 -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt`
    
**Findings:** The virtual host scan returned no results. Note that the target returned a `[WARN]` message regarding the lack of a base domain name to append, meaning VHOST fuzzing directly against an raw IP without a configured domain (e.g., `target.htb`) is highly likely to fail or yield false negatives.

![[Pasted image 20260529152647.png]]

### 2.2.4 HTTP Header Inspection

**Command:** curl -I http://10.129.254.12

**Findings:** The headers confirmed the web server is Apache/2.4.18 (Ubuntu). The Last-Modified header dates back to December 28, 2017, and the Content-Length is tiny (93 bytes), matching the minimal "Hello World!" text. No custom headers or tracking cookies of note were present.

![[Pasted image 20260529152722.png]]

### 2.2.5 Source Code Inspection

During the initial reconnaissance phase, I performed a source code review of the landing page. A comment in the HTML source provided a direct lead:

`<!-- /nibbleblog/ directory. Nothing interesting here! -->`
![[Pasted image 20260529170153.png]]

Navigating to `http://10.129.254.12/nibbleblog/` revealed a web interface. 
![[Pasted image 20260529170142.png]]

I then used Burpsuite to intercept the data packets when attempting to access this new webpage. Two data packets were observed, with one having a status code of 200 and php extension while the other has a status code of 404 and jpg extension.
Zooming in on the jpg extension data packet revealed a url with the term private which could contain useful information - `/nibbleblog/content/private/plugins/my_image/image.jpg`. 
![[Pasted image 20260529174535.png]]

Accessing the website `10.129.254.12/nibbleblog/content/private` gave me a landing page that consists of a list of directories. 
![[Pasted image 20260529174709.png|584]]

Going through the different sites revealed the following - 

1. keys.php and shadow.php revealed nothing. Even though burpsuite revealed a 200 OK message, the site was completely empty. 
2. users.xml revealed a username "admin"
![[Pasted image 20260529175000.png]]
3. Returning to the Parent Directory and accessing the following webpage  `http://10.129.254.12/nibbleblog/content/public/upload/` showed a list of images, and selecting the one titled nibbles_0_o.jpg gave me the following image:
![[Pasted image 20260529180201.png|346]]

I noted that the word Nibbles appears multiple times in the text body, which I interpreted as having some sort of significance, likely a password, since I already have a username.
Logical guesswork prompted me to access the following webpage `http://10.129.254.12/nibbleblog/admin.php
![[Pasted image 20260529180457.png]]

An attempt to login was then made with the following credentials:
- username = admin
- password = nibbles



## 3. Exploit (Initial Access)

Success! I got in!
![[Pasted image 20260529180620.png]]

After exploring the website, I noted one place where I can upload malicious payloads - and that is in the Plugin sections -> My Image.
As a test run, I uploaded a jpeg file titled `Dragon.jpeg` to determine whether the extension is accepted. And it was. Changes has been saved successfully. 
![[Pasted image 20260529181155.png]]

Accessing the Plugins in the directory page revealed that even though I originally named the file as `dragon.jpeg`, the system renamed it to `image.jpeg`. This will be useful information for when I upload my extra payload. 
*Note: Always verify if the server renames or processes your uploaded files, as this changes your final execution path.*
![[Pasted image 20260531151625.png]]

For the exploit, I copied the a reverse shell exploit from my linux machine `usr/share/webshells/php/php-reverse-shell.php` into a file called shell.php using the `cp` command; before running `ls -l` to confirm that the file creation was a success. I then uploaded it into the same webpage : Plugins > My image
![[Pasted image 20260531152638.png]]

![[Pasted image 20260529182413.png]]

And as expected, my shell.php file was renamed to image.php. Knowing the name of my uploaded shell and its location therefore allows me to run the exploit and create a shell using netcat on my attack machine. 

![[Pasted image 20260529182519.png]]

The exploit was successful and a shell was created. I ran whoami to check my identity (nibbler) and `sudo -l` to confirm what permissions have been assigned to the current logged in user. In this scenario, I can run the following command on Nibbles: `/home/nibbler/personal/stuff/monitor.sh`; Knowing this could be a possible path to privilege escalation, I took note of this before proceeding to find my current user flag. 

![[Pasted image 20260529182532.png]]
![[Pasted image 20260531153000.png]]

Running commands `ls -l` and `cd` eventually led me to the user flag which was inside a file called user.txt. The `cat` command was used to reveal the user's flag as :
8e0e842754a30754f883772636b7b129

![[Pasted image 20260529182707.png]]

![[Pasted image 20260529182725.png]]



## 4. Privilege Escalation

Upon obtaining the user flag, I can now proceed with Privilege Escalation, which we will refer to our previously found command to proceed with. 
It was noted that the `home/nibbler/personal/stuff/monitor.sh` did not exist within the system. So I carried out the following steps in order:
1. created the folder which the `monitor.sh` file was supposed to be stored in using `mkdir` 
2. copied the data from `bin/sh` into the folder under the file name `monitor.sh`
3. ran chmod +x to add execute permissions to said file
4. and ran it with the sudo command since my current logged in user nibbles has permission to execute this specific file. `.../monitor.sh`

![[Pasted image 20260529183028.png]]

*Note: if nothing appears after the sudo command is run, do a simple test command such as whoami or id. If something appears, it could be a non-interactive shell. To confirm, I ran the python command to perform a TTY spawn. After which, root@Nibbles... appeared which means I have successfully converted it into a fully interactive TTY shell.*

![[Pasted image 20260531154635.png]]



## 5. Post Exploitation

From here on out, it is simply a case of running the same set of commands of `ls -l`, `cd` and `cat`; since root texts are usually titled root.txt for Hack The Box Linux boxes, I used logical deduction to guess its location and name which turns out to just be root.txt in root's home folder. 
The flag is revealed to be: 
6efff93c6f4c69118fa025424639abe5

![[Pasted image 20260529183242.png]]
