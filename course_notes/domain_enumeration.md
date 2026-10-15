# Domain Enumeration

## Topics Overview

- Domain Enumeration Introduction
	- Domain Enumeration Tools
- ADModule and PowerView
- PowerHuntShares
- BloodHound and SOAPHound
	- BloodHound
	- Stealthy Data Collection in BloodHound
	- SOAPHound
- Access Control List (ACL)
	- Access Control Model
	- Enumerating ACLs
- Group Policy
	- Group Policy Objects (GPO)
	- Organizational Units (OU)
	- Enumerating GPOs
- Trusts
	- Trust Directions
	- Trust Transitivity
	- Default/Automatic Trusts
	- External Trusts
	- Enumerating Trusts
- Forests
	- Enumerating Forests in Active Directory
	- Enumerating Global Catalogs
	- Enumerating Forest Trusts
- User Hunting
	- Identifying Local Admin Access
	- Finding Domain Admin Sessions
	- Checking for Domain Admin Access
	- Enumerating Remote User Sessions

---

## Domain Enumeration Introduction

Domain enumeration is a crucial phase in penetration testing and Red Teaming in Active Directory environments. It allows the collection of information on users, groups, computers, and domain policies.

### Domain Enumeration Tools

To gather domain information, we can use the following tools:

1. **Active Directory PowerShell Module**

[**ADModule**](https://github.com/samratashok/ADModule) is the official Microsoft module, digitally signed, and usable even in **Constrained Language Mode (CLM)**.
Official documentation: [Microsoft Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/?view=windowsserver2022-ps).

```powershell
Import-Module C:\AD\Tools\ADModule-master\Microsoft.ActiveDirectory.Management.dll
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1
```

2. **BloodHound (C# and PowerShell Collectors)**

[**BloodHound**](https://github.com/BloodHoundAD/BloodHound) is a tool for visualizing relationships between Active Directory objects.

3. **PowerView (PowerShell)**

[**PowerView**](https://github.com/ZeroDayLab/PowerSploit/blob/master/Recon/PowerView.ps1) is part of the **PowerSploit** suite, used for domain enumeration via PowerShell.

```powershell
. C:\AD\Tools\PowerView.ps1
```

4. **SharpView (C#)**

[**SharpView**](https://github.com/tevora-threat/SharpView/) is an alternative to PowerView written in C#. It does not support pipeline filtering.

---

## ADModule and PowerView

### Basic Domain Enumeration Commands

**Get Active Directory Domain**

```powershell
# For the current domain
Get-Domain # PowerView
Get-ADDomain # ActiveDirectory Module

# For another domain
Get-Domain -Domain moneycorp.local
Get-ADDomain -Identity moneycorp.local
```

**Get Domain SID**

```powershell
# For the current domain
Get-DomainSID
(Get-ADDomain).DomainSID
```

**Get Domain Policy**

```powershell
# For the current domain
Get-DomainPolicyData
(Get-DomainPolicyData).systemaccess

# For another domain
(Get-DomainPolicyData -domain moneycorp.local).systemaccess
```

**Get Domain Controllers**

```powershell
# For the current domain
Get-DomainController
Get-ADDomainController

# For another domain
Get-DomainController -Domain moneycorp.local
Get-ADDomainController -DomainName moneycorp.local -Discover
```

### User Enumeration

**Get Users**

```powershell
# Get a list of users in the current domain
Get-DomainUser
Get-ADUser -Filter * -Properties *

# Get a specific user
Get-DomainUser -Identity student1
Get-ADUser -Identity student1
```

**List Properties for Users**

```powershell
# Get list of all properties for a specific user in the current domain
Get-DomainUser -Identity student1 -Properties *
Get-ADUser -Identity student1 -Properties *

# Filter on user properties
Get-DomainUser -Properties samaccountname,logonCount
Get-ADUser -Filter * -Properties * | select -First 1 | Get-Member -MemberType *Property | select Name
Get-ADUser -Filter * -Properties * | select name,logoncount,@{expression={[datetime]::fromFileTime($_.pwdlastset)}}

# Search for a particular string in a user's attributes
Get-DomainUser -LDAPFilter "Description=*built*" | Select name,Description
Get-ADUser -Filter 'Description -like "*built*"' -Properties Description | select name,Description
```

**Get Computers**

```powershell
# Get all the computers in the current domain
Get-DomainComputer | select Name
Get-ADComputer -Filter * | select Name

# Get all the computers running Windows Server 2022
Get-DomainComputer -OperatingSystem "*Server 2022*"
Get-ADComputer -Filter 'OperatingSystem -like "*Server 2022*"' -Properties OperatingSystem | select Name,OperatingSystem

# Check if a machine is reachable via ICMP
Get-ADComputer -Filter * -Properties DNSHostName | %{Test-Connection -Count 1 -ComputerName $_.DNSHostName}
Get-DomainComputer -Ping
```

**Get Domain Groups**

```powershell
# Get all the groups in the current domain
Get-DomainGroup | select Name
Get-ADGroup -Filter * | select Name

# Get all the groups for another domain
Get-DomainGroup -Domain <targetdomain>
Get-ADGroup -Filter * -Server <targetdomain>

# Filter groups containing "admin" in the group name
Get-DomainGroup *admin*
Get-ADGroup -Filter 'Name -like "*admin*"' | select Name
```

**Get Members of Domain Groups**

```powershell
# Get all the members of the Domain Admins group
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-ADGroupMember -Identity "Domain Admins" -Recursive

# Get the group membership for a user
Get-DomainGroup -UserName "student1"
Get-ADPrincipalGroupMembership -Identity student1
```

**Get Local Groups** (Needs Admin Privileges on non-DC Machines)

```powershell
# Get all the local groups on a machine
Get-NetLocalGroup -ComputerName dcorp-dc

# Get members of the local group "Administrators" on a machine
Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators
```

**Get Logged-in Users** (Needs Specific Rights on the Target Machine)

```powershell
# Get actively logged users on a computer (needs local admin rights on the target)
Get-NetLoggedon -ComputerName dcorp-adminsrv

# Get locally logged users on a computer (needs remote registry on the target machine)
Get-LoggedonLocal -ComputerName dcorp-adminsrv

# Get the last logged user on a computer (needs administrative rights and remote registry on the target machine)
Get-LastLoggedOn -ComputerName dcorp-adminsrv
```

**Get Shares and Sensitive Files**

```powershell
# Find shares on hosts in current domain
Invoke-ShareFinder -Verbose

# Find sensitive files on computers in the domain
Invoke-FileFinder -Verbose

# Get all fileservers of the domain
Get-NetFileServer
```

---

### PowerHuntShares

For enumerating shares, we can also use [**PowerHuntShares**](https://github.com/NetSPI/PowerHuntShares) that is an advanced tool for SMB share enumeration.

**Key Features**

- **Identifies shared folders**: scans the network to find all accessible shares.
- **Finds sensitive files**: searches for sensitive information within shares.
- **Analyzes ACLs (Access Control Lists)**: checks permissions on shared folders.
- **Generates an HTML report**: provides a detailed report with a clear view of share relationships.
- **Includes "ShareGraph"**: a visual representation of relationships between shares.

To execute PowerHuntShares and collect information about SMB shares, use the following command:
```powershell
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```

- `-NoPing`: skips host pings to reduce the risk of detection (**better OPSEC**).
- `-OutputDirectory C:\AD\Tools`: specifies the directory where the HTML report will be saved.
- `-HostList C:\AD\Tools\servers.txt`: specifies the list of servers to scan.

The **`servers.txt`** file **excludes the domain controller (DC)** to avoid unnecessary noise and detection. Indeed, PowerHuntShares allows **low-profile enumeration** by avoiding excessive queries in the DC logs.

---

## BloodHound and SOAPHound

### BloodHound

BloodHound is a powerful tool for **Active Directory (AD) enumeration**, providing a graphical representation of AD entities and relationships. It helps security professionals map **attack paths** to high-value targets, such as **Domain Admins**, using **graph theory**.

**Key Features**

- **Graph-based visualization** of AD relationships.
- **Built-in queries** for commonly used actions.
- **Custom Cypher query support** for advanced analysis.
- **Shortest path mapping** to privileged accounts (e.g., Domain Admins).

There are two free versions of BloodHound:
1. [**BloodHound Legacy**](https://github.com/BloodHoundAD/BloodHound)
2. [**BloodHound CE (Community Edition)**](https://github.com/SpecterOps/BloodHound)

#### BloodHound Legacy

To collect data for BloodHound Legacy, run the following command:

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\BloodHound-master\BloodHound-master\Collectors\SharpHound.exe -args --collectionmethods All
```

**SharpHound.exe** is the official data collector for BloodHound.
The collected data can be uploaded to **BloodHound Legacy** for analysis.

#### BloodHound CE (Community Edition)

To collect data for BloodHound CE, run the following command:

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Sharphound\SharpHound.exe -args --collectionmethods All
```

The data gathered can be uploaded to **BloodHound CE** for visualization and analysis.

### Stealthy Data Collection in BloodHound

By default, **SharpHound** collects extensive data, which can generate noticeable network activity. To minimize detection:

Remove collection methods that generate excessive logs:
- **RDP**
- **DCOM**
- **PSRemote**
- **LocalAdmin**

To avoid detection by **Microsoft Defender for Identity (MDI)**, use the following command:

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SharpHound\SharpHound.exe -args --collectionmethods Group,GPOLocalGroup,Session,Trusts,ACL,Container,ObjectProps,SPNTargets,CertServices --excludedcs
```

`-ExcludeDCs` option is for stealth.

### SOAPHound

[**SOAPHound**](https://github.com/FalconForceTeam/SOAPHound) is an alternative enumeration tool designed for **stealthy Active Directory enumeration**.
It talks to Active Driectory Web Services (on port 9389) in place of sending LDAP queries, just like the ADModule. So, almost no network-based detection (like MDI).

**Build a Cache of Domain Objects**

```powershell
SOAPHound.exe --buildcache -c C:\AD\Tools\cache.txt
```

This builds a **local cache** containing **basic domain object information**.

**Collect BloodHound-Compatible Data**

```powershell
SOAPHound.exe -c C:\AD\Tools\cache.txt --bhdump -o C:\AD\Tools\bloodhound-output --nolaps
```

This exports **BloodHound-compatible data** in a stealthy manner.

### BloodHound and SOAPHound Summary

**Key Takeaways**

✔ **BloodHound provides a graphical view of AD relationships and attack paths.**
✔ **BloodHound CE is the modern version, while BloodHound Legacy is still widely used.**
✔ **Stealthy enumeration should exclude noisy methods like RDP, DCOM, and LocalAdmin.**
✔ **Using `-ExcludeDCs` helps prevent detection by Microsoft Defender for Identity.**
✔ **SOAPHound is a stealth alternative that avoids traditional LDAP queries for enumeration.**

---

## Access Control List (ACL)

**Access Control Lists (ACLs)** are fundamental to **Active Directory (AD) security**, as they define and regulate access to domain objects. Understanding how ACLs function is essential for both **defensive security (hardening AD)** and **offensive operations (enumeration and privilege escalation)**.

An **Access Control List (ACL)** is a collection of **Access Control Entries (ACE)** that specify **permissions and auditing rules** for an object.

**Types of ACLs**
1. **DACL (Discretionary ACL)** → Defines which users or groups have access to an object.
2. **SACL (System ACL)** → Logs audit events for successful or failed access attempts.

ACLs play a **critical role in Active Directory security** by controlling permissions on domain objects.

**Importance of SACL in Auditing**

**SACL entries** allow tracking of successful and failed access attempts.
These events can be analyzed in **Windows Event Logs** (`Security` log under **Event ID 4662** for object access).
Monitoring SACL entries helps **detect unauthorized access attempts** and **identify potential privilege escalation paths**.

### Access Control Model

Active Directory uses an **Access Control Model** to regulate access to objects and resources based on:
- **Access Tokens**. Represent the security context of a process, including the identity and privileges of a user.
- **Security Descriptors**, that contain:
  - **SID of the owner**.
  - **Discretionary ACL (DACL)**: defines access permissions.
  - **System ACL (SACL)**: used for auditing access attempts.

**How ACLs Work**

1. **An object has a DACL that defines permissions**.
2. **Each entry (ACE) within the DACL specifies access rules** for users or groups.
3. **Access tokens** from the requesting process determine whether access is granted or denied.
4. **Precedence is given to explicit deny rules** over allow rules.

**Example Scenario**

**Thread A** (user: Andrew, member of Group A, Group B, Group C) attempts access.
The DACL contains the following ACEs:
1. **ACE 1 (Explicit Deny for Andrew)**: Access is denied immediately.
2. **ACE 2 (Allow for Group A - Write Access)**.
3. **ACE 3 (Allow for Everyone - Read and Execute)**.

Because **deny rules take precedence**, Andrew is denied access.

### Enumerating ACLs

**Get ACLs for a Specific Object**

```powershell
Get-DomainObjectAcl -SamAccountName student1 -ResolveGUIDs
```

**Search for ACLs by LDAP Path**

```powershell
Get-DomainObjectAcl -SearchBase "LDAP://CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local" -ResolveGUIDs -Verbose
```

**Enumerate ACLs Using Active Directory Module**

```powershell
(Get-Acl 'AD:\CN=Administrator,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local').Access
```

**Search for Interesting ACEs**

```powershell
Find-InterestingDomainAcl -ResolveGUIDs
```

**Get ACLs for a Specific Network Path**

```powershell
Get-PathAcl -Path "\\dcorp-dc.dollarcorp.moneycorp.local\sysvol"
```

### Access Control List (ACL) Summary

**Key Takeaways**

✔ **ACLs control access to objects in Active Directory based on Access Tokens and Security Descriptors.**
✔ **DACLs define access permissions, while SACLs handle auditing.**
✔ **Deny ACEs take precedence over Allow ACEs.**
✔ **PowerShell commands like `Get-DomainObjectAcl`, `Get-Acl`, and `Find-InterestingDomainAcl` help enumerate ACLs.**
✔ **Auditing ACLs is crucial for identifying misconfigurations and privilege escalation paths.**

---

## Group Policy

**Group Policy** provides a centralized way to manage **configuration settings** in an **Active Directory (AD) environment**.

A policy setting can apply to:
- **Computers**: security settings, startup and shutdown scripts, assigned applications, and more.
- **Users**: security settings, logon and logoff scripts, assigned applications, and more.

### Group Policy Objects (GPO)

**Group Policy settings** are stored in a **Group Policy Object (GPO)**.

**What is a GPO?**

A **GPO (Group Policy Object)** is a virtual collection of **policy settings, security permissions, and scope of management (SOM)** used to **enforce security and configuration settings** on users and computers in an Active Directory environment.

**GPOs can be linked to:**
- **Domains**
- **Sites**
- **Organizational Units (OUs)**

**Security Risks**

**Overly permissive GPOs** can be abused for:
- **Privilege escalation**
- **Backdoors**
- **Persistence mechanisms**
- **Misconfigurations that allow lateral movement**

### Organizational Units (OU)

Organizations use **Organizational Units (OUs)** to **delegate administration** within Active Directory.

An **OU** is the **smallest container** in AD to which a **GPO can be applied**.

**Common Attack Surface:**
- **Misconfigured or overly permissive GPOs at the OU level** are a common security risk in enterprise environments.
- Attackers often target **OUs with weak access control settings** to gain further control over the domain.

### Enumerating GPOs

**List All GPOs in the Current Domain**

```powershell
Get-DomainGPO
Get-DomainGPO -ComputerIdentity dcorp-student1
```

**Find GPOs Using Restricted Groups or Groups.xml**

```powershell
Get-DomainGPOLocalGroup
```

**Mapping Group Memberships via GPO**

```powershell
# Get users in a local group of a machine using GPO
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity dcorp-student1

# Find machines where a user is a member of a specific group
Get-DomainGPOUserLocalGroupMapping -Identity student1 -Verbose
```

**Enumerating Organizational Units (OUs)**

```powershell
# Get all OUs in a domain
Get-DomainOU
Get-ADOrganizationalUnit -Filter * -Properties *

# Find GPOs applied to an OU
Get-DomainGPO -Identity "{0D1CC23D-1F20-4EEE-AF64-D99597AE2A6E}"
```

The **GPOs applied to a specific Organizational Unit (OU)** can be identified using the `gplink` attribute (from `Get-NetOU`), which contains references to linked GPOs.

### Group Policy Summary

**Key Takeaways**

✔ **Group Policy (GPO) provides centralized configuration management for users and computers in Active Directory.**
✔ **GPOs are applied at the domain, site, and OU levels and can enforce security policies, scripts, and application settings.**
✔ **Misconfigured GPOs can be exploited for privilege escalation, persistence, and lateral movement.**
✔ **Organizational Units (OUs) structure Active Directory and define policy inheritance.**
✔ **PowerShell commands like `Get-DomainGPO`, `Get-DomainOU`, and `Get-DomainGPOLocalGroup` help enumerate GPOs and analyze security risks.**

---

## Trusts

In an **Active Directory (AD) environment**, a **trust** is a relationship between **two domains or forests** that allows users from one domain or forest to access **resources in another domain or forest**.

Trusts can be:
- **Automatic**: established by default (e.g., **parent-child trusts, intra-forest trusts**).
- **Manually established**: configured by administrators (e.g., **forest trusts, external trusts**).

**Trusted Domain Objects (TDOs)** represent trust relationships in a domain.

### Trust Directions

Trust relationships can be either **one-way** or **two-way**.

1. **One-Way Trust**

**Unidirectional** → Users from the **trusted** domain can access resources in the **trusting** domain, but not the other way around.

Trusted (Account) Domain → Users can authenticate ↓ Trusting (Resource) Domain → Grants access to resources

2. **Two-Way Trust**

**Bidirectional** → Users from **both domains** can access each other's resources.
This is often used for **forest-wide** or **intra-forest** relationships.

Domain A ↔ Domain B → Mutual access

### Trust Transitivity

Trusts in AD can be **transitive** or **nontransitive**.

1. **Transitive Trust**

Can be **extended** to establish trust relationships with **other domains**.
**Default for intra-forest trusts** like:
- **Tree-root trust**
- **Parent-child trust**

2. **Nontransitive Trust**

**Limited to two domains** and **cannot extend further**.
Can be **one-way or two-way**.
**Default for external trusts** (between different forests without an implicit trust).

### Default/Automatic Trusts

Some **trusts are created automatically** when a new domain or tree is added.

1. **Parent-Child Trust**

Created **automatically** between a **new domain** and its **parent** in the namespace hierarchy.

Example:

dollarcorp.moneycorp.local (child) ↳ moneycorp.local (parent)

**Always** two-way and transitive.

2. **Tree-Root Trust**

Created **automatically** when a **new domain tree** is added to a **forest root**.
**Always** two-way and transitive.

### External Trusts

Used when two domains **from different forests** need to communicate but **do not share a forest-wide trust**.
**Can be one-way or two-way** but is **always nontransitive**.

1. Tree-Root Trust

Created **automatically** when a **new domain tree** is added to a **forest root**.
**Always** two-way and transitive.

2. **External Trusts**

Used when two domains **from different forests** need to communicate but **do not share a forest-wide trust**.
**Can be one-way or two-way** but is **always nontransitive**.

Forest 1 ↔ Forest 2 → External Trust

3. Forest Trusts

Established **between the root domains of two forests**.
**Cannot extend to a third forest** (no implicit trust).
**Can be one-way or two-way transitive**.

Forest 1 ↔ Forest 2 → Direct Trust Forest 2 ↔ Forest 3 → Direct Trust Forest 1 🚫 Forest 3 (No implicit trust)

### Enumerating Trusts

To enumerate trust relationships in a domain, use the following PowerShell commands.

**Get a List of Domain Trusts**

```powershell
Get-DomainTrust
Get-DomainTrust -Domain us.dollarcorp.moneycorp.local

Get-ADTrust
Get-ADTrust -Identity us.dollarcorp.moneycorp.local
```

### Trusts Summary

**Key Takeaways**

✔ **Trusts allow authentication between different domains and forests in Active Directory.**
✔ **Trusts can be one-way (unidirectional) or two-way (bidirectional).**
✔ **Transitive trusts allow authentication to be extended across multiple domains, while nontransitive trusts do not.**
✔ **Automatic trusts include parent-child and tree-root trusts, which are always two-way and transitive.**
✔ **External trusts are nontransitive and used when two forests do not have an implicit trust.**
✔ **Forest trusts allow communication between forests but cannot be extended beyond the direct relationship.**
✔ **PowerShell commands like `Get-DomainTrust` and `Get-ADTrust` help enumerate trust relationships.**

---

## Forests

A **forest** in Active Directory is the highest level of organization, containing **multiple domains** that share a common schema, configuration, and global catalog.

### Enumerating Forest

**Get Details About the Current Forest**

```powershell
Get-Forest
Get-Forest -Forest eurocorp.local
Get-ADForest
Get-ADForest -Identity eurocorp.local
```

**Get All Domains in the Current Forest**

```powershell
Get-ForestDomain
Get-ForestDomain -Forest eurocorp.local
(Get-ADForest).Domains
```

### Enumerating Global Catalogs

**Get All Global Catalogs in the Current Forest**

A **Global Catalog (GC)** contains a partial, read-only replica of all objects in the forest and is used for searching AD objects across domains.

```powershell
Get-ForestGlobalCatalog
Get-ForestGlobalCatalog -Forest eurocorp.local
Get-ADForest | select -ExpandProperty GlobalCatalogs
```

### Enumerating Forest Trusts

**Map Forest Trusts**

Trusts at the **forest level** define relationships between **different forests**, allowing users from one forest to authenticate in another

```powershell
Get-ForestTrust
Get-ForestTrust -Forest eurocorp.local
Get-ADTrust -Filter 'msDS-TrustForestTrustInfo -ne "$null"'
```

### Forests Summary

**Key Takeaways**

✔ A forest is the highest organizational unit in Active Directory, containing multiple domains.
✔ The `Get-Forest` and `Get-ADForest` commands provide forest-level information.
✔ Domains within a forest can be listed using `Get-ForestDomain`.
✔ Global Catalog servers, which store partial replicas of AD objects, can be found using `Get-ForestGlobalCatalog`.
✔ Forest trust relationships can be enumerated using `Get-ForestTrust` and `Get-ADTrust`.

---

## User Hunting

### Identifying Local Admin Access

**Find Machines Where the Current User Has Local Admin Access**

The following command scans all machines in the domain and identifies where the **current user** has **local administrator privileges**.

```powershell
Find-LocalAdminAccess -Verbose
```

**How It Works**

- Queries the **Domain Controller (DC)** for a **list of machines** (`Get-NetComputer`).
- Uses **multi-threading** to check local admin rights on each machine (`Invoke-CheckLocalAdminAccess`).
- Can also be performed using **remote administration tools** such as **WMI** or **PowerShell Remoting**:
    - **Find-WMILocalAdminAccess.ps1**
    - **Find-PSRemotingLocalAdminAccess.ps1**

### Finding Domain Admin Sessions

**Find Computers Where a Domain Admin (or Specified User/Group) Has Active Sessions**

```powershell
Find-DomainUserLocation -Verbose
Find-DomainUserLocation -UserGroupIdentity "RDPUsers"
```

**How It Works**

- Queries the **DC** to retrieve members of the **specified group** (`Get-DomainGroupMember`).
- Retrieves a list of **computers in the domain** (`Get-DomainComputer`).
- Lists **logged-on users and active sessions** (`Get-NetSession` / `Get-NetLoggedon`).
- **⚠️ On Windows Server 2019+**, local administrator privileges are required to **list sessions**.

### Checking for Domain Admin Access

**Find Computers Where the User Has Admin Rights**

```powershell
Find-DomainUserLocation -CheckAccess
```

Uses `Test-AdminAccess` to verify whether the user has administrative access to a machine.

**Find File Servers and DFS Servers Where a Domain Admin Session Exists**

```powershell
Find-DomainUserLocation -Stealth
```

### Enumerating Remote User Sessions

**List Sessions on Remote Machines Without Admin Access**

- Uses **Remote Registry** to query `HKEY_USERS` without requiring **admin privileges**.
- Fetches **active user sessions** from remote machines.

```powershell
Invoke-SessionHunter -FailSafe
```

**OPSEC-Friendly Enumeration**

- Avoid scanning **all** machines to reduce detection risk.
- Target only specific hosts using a predefined list.

```powershell
Invoke-SessionHunter -NoPortScan -Targets C:\AD\Tools\servers.txt
```

### User Hunting Summary

**Key Takeaways**

✔ **User hunting helps identify where a user (or admin) has active sessions, aiding lateral movement strategies.**
✔ **`Find-LocalAdminAccess` identifies machines where the user has local admin privileges.**
✔ **`Find-DomainUserLocation` can find logged-on users, including domain admins, across the network.**
✔ **Remote enumeration can be performed stealthily using `Invoke-SessionHunter` and `Find-WMILocalAdminAccess.ps1`.**
✔ **Using `-CheckAccess` and `-Stealth` modes can improve operational security (OPSEC).**

---
---
