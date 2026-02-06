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
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

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
PORT     STATE    SERVICE
21/tcp   open     ftp
22/tcp   open     ssh
80/tcp   open     http
445/tcp  open     microsoft-ds
5355/tcp filtered llmnr
8080/tcp open     http-proxy

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
PORT    STATE SERVICE VERSION

445/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))

|_http-title: Apache2 Ubuntu Default Page: It works

|_http-server-header: Apache/2.4.7 (Ubuntu)

now i know why the samba port wasn't reponsding to my smbclient commands and why there was only one out of the two samba ports open. the port 445 is another http port. lets visit it to find out some more clues.

```bash
http://target_ip:445
```
in the source code wwe find the passphrase of the cool.jpeg but since we used stegseek to authomatically crack the passphrase, we wouldn't need it anyways. lets naviagate to the directory which we found in cool.jpeg.out. at the directory, we find another note for bob which could be another password for something. since we do not know much about where the password is supposed to be used, we will be fuzzing the directories of the port 445 webpage

```bash
gobuster dir -u http://<target_ip>:445 -w/usr/share/wordlists/dirb/common.txt
```
.hta                 (Status: 403) [Size: 285]

.htaccess            (Status: 403) [Size: 290]

.htpasswd            (Status: 403) [Size: 290]

index.html           (Status: 200) [Size: 11596]

server-status        (Status: 403) [Size: 294]

user                 (Status: 200) [Size: 3401]

at the /user directory we find an id_rsa private key for some user. lets paste it into a file named id_rsa. i am guessing that this must be bob's id_rsa but its just a guess

```bash
# give the id_rsa the appropriate permissions
chmod 600 id_rsa
```
after a long time of enumeration, i fell into another rabbit hole. then i realized that the credentials might be encrypted since we never really used them on the login page at port 8080 webpage. lets use this ![Viginere Cipher Decrypter ](https://www.boxentriq.com/analysis/cipher-identifier) to decrypt  the viginere cipher (just an assumption since viginere ciphers are the most common in CTFs). turns out we were right! we get bob's credentials. lets use them on the login page.
                                               
