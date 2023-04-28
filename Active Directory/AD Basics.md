## Active Directory Basics

A windows domain is a group of users and computers under the administration of a given business. The main idea behind a domain is to centralise the administration of common components of a Windows computer network in a single repository.

This is called **Activer Directory (AD)**.
The server that runs the AD services is known as a **Domain Controller (DC)**

In a windows domain, credentials are stored in a centralised repository called:
*Active Directory*

The server in charge of running the Active Directory is called:
*Domain Controller*

The core of any Windows Domain is the **Active Directory Domain Service (AD DS)**
This service acts as a catalogue that holds the information of all the "objects" that exists in your network. (Users, groups, machines, printers, shares etc).

**Machine accounts**
For every computer that joins the AD domain a machine object will be created, machines are also considered "security principals" and are assigned an accounts just as any regular user. This account has somewhat limited rights withing the domain itself.
These accounts can be identified via their name, e.g: **A machine called `DC01` will have a machine account called `DC01$`**

**Security Groups**
Security groups are also considered security principals, and therefore, can have privileges over resources on the network.
Groups can have both users and machines as members, If needed, groups can include others group as well.
Several groups are created by default in a domain to grant specific privilege to the users. Heres a table with the most important groups in a domain:
| Security Group     | Description                                                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Domain Admins      | Users of this group have administrative privileges over the entire domain. By default they can administer any computer on the domain, including the DCs |
| Server Operators   | Users in this group can administer Domain Controllers. They cannot change any administrative group memberships.                                         |
| Backup Operators   | Users in this group are allowed to access any file, ignoring their permissions. They are used to perform backups of data on computers.                  |
| Account Operators  | Users in this group can create or modify other accounts in the domain.                                                                                  |
| Domain Users       | Includes all existing user accounts in the domain                                                                                                       |
| Domain Computers   | Includes all existing computers in the domain                                                                                                           |
| Domain Controllers | Includes all existing DCs on the domain                                                                                                                 |

Organizational Units are container objects that allow you to classify users and machines.

````ad-note
OUs are handy for applying policies to users and computers, which include specific configurations that pertain to sets of users depending on their particular role in the enterprise. Remember, a user can only be a member of a single OU at a time, as it wouldn't make sense to try to apply two different sets of policies to a single user.
````

```ad-note
Security Groups, on the other hand, are used to grant permissions over resources. For example, you will use groups if you want to allow some users to access a shared folder or network printer. A user can be a part of many groups, which is needed to grant access to multiple resources.
```

Delegation allowes you to grant users specific privileges to perform advanced tasks on OUs withouth needing a Domain Administrator to step in.

IT support takes advantage of this by having permission to reset other low priviliged users passwords.

1. Workstations

Workstations are one of the most common devices within an Active Directory domain. Each user in the domain will likely be logging into a workstation. This is the device they will use to do their work or normal browsing activities. These devices should never have a privileged user signed into them.

2. Servers

Servers are the second most common device within an Active Directory domain. Servers are generally used to provide services to users or other servers.

3. Domain Controllers

Domain Controllers are the third most common device within an Active Directory domain. Domain Controllers allow you to manage the Active Directory Domain. These devices are often deemed the most sensitive devices within the network as they contain hashed passwords for all user accounts within the environment.