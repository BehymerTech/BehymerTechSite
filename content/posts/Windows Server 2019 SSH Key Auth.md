---
menu: "main"
weight: 50
---
## Introduction:
The OpenSSH server Microsoft has added for Windows Server 2019 has come a long way since its inception. The last time I tried the service, I had a lot of problems using key-based authentication. I was complaining about this online and @CyborgEvilham  envouraged me to give it another shot.

{{ < tweet 1430481859646038019 >}}

After playing around with it for a few minutes this morning, I was able to successfully use SSH key authentication from my WSL Ubuntu instance back to my primary desktop without issue. There are a few additional steps and I do not think the instructions are very clear so I am trying to combine instructions from a couple of sources here. One of the big things that I missed was that you have to set the ACL 


## Change Windows default shell to powershell:
``` powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```

## Install OpenSSH server, client, and start services:
``` powershell
# Install the OpenSSH Client
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0

# Install the OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Start the sshd service
Start-Service sshd

# OPTIONAL but recommended:
Set-Service -Name sshd -StartupType 'Automatic'

# Confirm the firewall rule is configured. It should be created automatically by setup.
Get-NetFirewallRule -Name *ssh*

# There should be a firewall rule named "OpenSSH-Server-In-TCP", which should be enabled
# If the firewall does not exist, create one
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
``` 

## Copy your SSH key from the remote system 

Either SCP your public key (example: ~/.ssh/id_rsa.pub) or copy the contents to the correct .ssh/authorized_keys file on the server. 
- If you adding key for a standard user, you can just place it in C:\Users\Your-UserName\.ssh\ 
- If your user is an administrator on the system, you will copy your public key contents to C:\ProgramData\ssh\administrator_authorized_keys

### Set the correct permissions on the administrator_authorized_keys file:
If you added the administrator_authorized_keys file, you will need to set it to where only Administrators and SYSTEM have access and both have full control. You can do this through the UI or using the script below sourced from https://www.concurrency.com/blog/may-2019/key-based-authentication-for-openssh-on-windows:

``` powershell
Set the ACL on the administrator_authorized_keys file:
$acl = Get-Acl C:\ProgramData\ssh\administrators_authorized_keys
$acl.SetAccessRuleProtection($true, $false)
$administratorsRule = New-Object system.security.accesscontrol.filesystemaccessrule("Administrators","FullControl","Allow")
$systemRule = New-Object system.security.accesscontrol.filesystemaccessrule("SYSTEM","FullControl","Allow")
$acl.SetAccessRule($administratorsRule)
$acl.SetAccessRule($systemRule)
$acl | Set-Acl
```
## Connecting
Connect from your remote client the same way you would to any ssh server: 
``` bash
ssh user@hostname
```

If you encounter any issues, remember to use the "-v" option to get verbose output. 


## Sources
- https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse
- https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement
- https://www.concurrency.com/blog/may-2019/key-based-authentication-for-openssh-on-windows