## Why AD enum
Generated credentials 
```
Username: leon.jennings Password: Password!
```
## Credentials injection

### Questions
**What native Windows binary allows us to inject credentials legitimately into memory**
```
runas.exe
```

**What parameter option of the runas binary will ensure that the injected credentials are used for all network connections?**
```
/netonly
```

**What network folder on a domain controller is accessible by any authenticated AD account and stores GPO information?**
```
SYSVOL
```

**When performing dir \\za.tryhackme.com\SYSVOL, what type of authentication is performed by default?**
```
Kerberos Authentication
```

## Enumeration Through Microsoft Management Console

### Drawbacks

- The net commands must be executed from a domain-joined machine. If the machine is not domain-joined, it will default to the WORKGROUP domain.

- The net commands may not show all information. For example, if a user is a member of more than ten groups, not all of these groups will be shown in the output.

### Usefull enumeration commands
```
##Enumerate users
net user /domain
net user [username] /domain

##Enumerate groups
net group /domain
net group [group name] /domain

##Enumerate password policy
net accounts /domain 

```

### Questions

**Apart from the Domain Users group, what other group is the aaron.harris account a member of?**
```
Internet Access
```

**Is the Guest account active? (Yay,Nay)**
```
Nay
```

**How many accounts are a member of the Tier 1 Admins group?**
```
7
```

**What is the account lockout duration of the current password policy in minutes?**
```
30
```

### User Enumeration Through PowerShell

### Usefull PS enum commands
```
##Users
Get-ADUser -Identity [account name] -Server [domain controller] -Properties *

### Using filter parameter for more control of enumeration and format to ###display the results in a clean way
Get-ADUser -Filter 'Name -like "*stevens"' -Server za.tryhackme.com | Format-Table Name,SamAccountName -A

##Groups
Get-ADGroup -Identity Administrators -Server za.tryhackme.com
###Group Membership
Get-ADGroupMember -Identity Administrators -Server za.tryhackme.com

##ADObjects

###Looking for all AD objects that were changed after a specific date:
$ChangeDate = New-Object DateTime(2022, 02, 28, 12, 00, 00)
Get-ADObject -Filter 'whenChanged -gt $ChangeDate' -includeDeletedObjects -Server za.tryhackme.com

### Password spray without locking accounts that have a badpwncount > 0
Get-ADObject -Filter 'badPwdCount -gt 0' -Server za.tryhackme.com

##Domains
Get-ADDomain -Server za.tryhackme.com

##Altering AD Objects
Set-ADAccountPassword -Identity gordon.stevens -Server za.tryhackme.com -OldPassword (ConvertTo-SecureString -AsPlaintext "old" -force) -NewPassword (ConvertTo-SecureString -AsPlainText "new" -Force)

```

### Bloodhound

Overview - Provides summaries information such as the number of active sessions the account has and if it can reach high-value targets.

Node Properties - Shows information regarding the AD account, such as the display name and the title.

Extra Properties - Provides more detailed AD information such as the distinguished name and when the account was created.

Group Membership - Shows information regarding the groups that the account is a member of.

Local Admin Rights - Provides information on domain-joined hosts where the account has administrative privileges.

Execution Rights - Provides information on special privileges such as the ability to RDP into a machine.

Outbound Control Rights - Shows information regarding AD objects where this account has permissions to modify their attributes.
Inbound Control Rights -  Provides information regarding AD objects that can modify the attributes of this account.

**THM86144**