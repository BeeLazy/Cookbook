# SSH Access

## About <a id="about"></a>
This document describes how to add users and SSH keys with **cloud-init**, and how to configure a client how to use them.  

## Description <a id="description"></a>
We will generate SSH keys with **ssh-keygen** and add them to our VMs with **cloud-init**.  

## Table of contents <a id="table-of-contents"></a>
1. [About](#about)
2. [Description](#description)
3. [Table of contents](#table-of-contents)
4. [Create the test environment](#create-the-test-environment)
5. [Generate SSH keys](#generate-ssh-keys)
6. [Add SSH key to VM](#add-ssh-key-to-vm)
    1. [Add SSH key with cloud-init](#add-ssh-key-with-cloud-init)
    2. [Add SSH key to existing VM](#add-ssh-key-to-existing-vm)
7. [Login with SSH Key](#login-with-ssh-key)
8. [Related links](#related-links)

## Generate SSH keys <a id="generate-ssh-keys"></a>
If you don't already have a SSH keypair you want to use, we will need to generate one. We can use **OpenSSH** for that on both Windows and Linux.  

Generate keys for user **myuser**. I will use an empty passphrase for my Lab. [Probably not](NIST.IR.7966.pdf) a good idea in production.
```powershell
ssh-keygen -C myuser -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (C:\Users\bee/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in C:\Users\bee/.ssh/id_ed25519
Your public key has been saved in C:\Users\bee/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:xRiwHugjY41jHDnNWlTps96cr3ogbkWv0PA8relChoY myuser
The key's randomart image is:
+--[ED25519 256]--+
|    ..oo.        |
|   = ... +       |
|  + =.o . o      |
| . O..+. .       |
| .X.+*.=S        |
|Eoo+=.O o        |
| . + = O .       |
|    + = =        |
|   . ooo.o.      |
+----[SHA256]-----+
```

Use a strong algorithm like **ed25519** unless you have a reason not to. By default, without any arguments, this command creates a key pair of a 
private key **id_rsa** and a public key **id_rsa.pub**. Other types of keys you can generate, and their filenames
```console
ssh-keygen -t rsa -b 4096
/.ssh/id_rsa
/.ssh/id_rsa.pub

ssh-keygen -t dsa
/.ssh/id_dsa
/.ssh/id_dsa.pub

ssh-keygen -t ecdsa -b 521
/.ssh/id_ecdsa
/.ssh/id_ecdsa.pub

ssh-keygen -t ed25519
/.ssh/id_ed25519
/.ssh/id_ed25519.pub
```

If you already have a key, you will need to copy it into your keyfiles.

On **Windows** the keyfiles are saved into:
```powershell
$env:USERPROFILE\.ssh\
```

On **Linux** they end up in:
```console
/home/$USER/.ssh/
```

## Add SSH key to VM <a id="add-ssh-key-to-vm"></a>
Now that we have a SSH key we can use. We need to get it added to our VMs. 

### Add SSH key with cloud-init <a id="add-ssh-key-with-cloud-init"></a>
The easiest way to add SSH keys, is to add them with **cloud-init** when the VM is created.  

Get the publickey you made earlier. I made a **ed25519** on Windows, and my username is **bee**, so my publickey 
is in **C:\Users\bee/.ssh/id_ed25519.pub**
```powershell
PS C:\Users\bee> cat C:\Users\bee/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDOUfb4ie1YyGfBJyOC3XzQAn+QeGnepdijc5n/RW50f myuser
```

Copy the entire line, and place it in your cloud-init yaml like this:

Contents of **sshuser.yaml** for a user called **myuser**
```yaml
users:
  - default
  - name: myuser
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
    - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDOUfb4ie1YyGfBJyOC3XzQAn+QeGnepdijc5n/RW50f myuser
```

We can now launch VMs, and have our SSH keys added automatically:
```console
multipass launch -n sshtestvm --cloud-init sshuser.yaml
Launched: sshtestvm
```

### Add SSH key to existing VM <a id="add-ssh-key-to-existing-vm"></a>
If you did not add the SSH keys to the server when it was created, you can add it at a later time too. 

**Standard user** needs to be added to the **authorized_keys** file in the **\\.ssh\\** directory (se above).  

> :warning: **Note:** On Windows **administrative user** should be added to **administrators_authorized_keys** in **C:\ProgramData\ssh\\**

To remotely add **standard user** SSH keys to a Windows Server
```powershell
# Get the public key file generated previously on your client
$authorizedKey = Get-Content -Path $env:USERPROFILE\.ssh\id_ed25519.pub

# Generate the PowerShell to be run remote that will copy the public key file generated previously on your client to the authorized_keys file on your server
$remotePowershell = "powershell New-Item -Force -ItemType Directory -Path $env:USERPROFILE\.ssh; Add-Content -Force -Path $env:USERPROFILE\.ssh\authorized_keys -Value '$authorizedKey'"

# Connect to your server and run the PowerShell using the $remotePowerShell variable
ssh username@domain1@contoso.com $remotePowershell
```

And for **administrative user**
```powershell
# Get the public key file generated previously on your client
$authorizedKey = Get-Content -Path $env:USERPROFILE\.ssh\id_ed25519.pub

# Generate the PowerShell to be run remote that will copy the public key file generated previously on your client to the authorized_keys file on your server
$remotePowershell = "powershell Add-Content -Force -Path $env:ProgramData\ssh\administrators_authorized_keys -Value '$authorizedKey';icacls.exe ""$env:ProgramData\ssh\administrators_authorized_keys"" /inheritance:r /grant ""Administrators:F"" /grant ""SYSTEM:F"""

# Connect to your server and run the PowerShell using the $remotePowerShell variable
ssh username@domain1@contoso.com $remotePowershell
```

## Login with SSH Key <a id="login-with-ssh-key"></a>
Now that everything is setup, it's ready to test a login with our custom **myuser** user.  

Find the VMs IP and SSH to it
```powershell
PS C:\Users\bee> multipass list
Name                    State             IPv4             Image
sshtestvm               Running           172.30.196.20    Ubuntu 22.04 LTS

PS C:\Users\bee> ssh myuser@172.30.196.20 -o StrictHostKeyChecking=no
Warning: Permanently added '172.30.196.20' (ED25519) to the list of known hosts.
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-71-generic x86_64)

$ uname -a
Linux sshtestvm 5.15.0-71-generic #78-Ubuntu SMP Tue Apr 18 09:00:29 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

$ printenv USER
myuser
```

We can now connect to our VMs from our host with tools that use SSH. This opens a new world when we can connect **VS Code** and many other tools, and not only 
the **multipass shell**.  

## Related links <a id="related-links"></a>
[Multipass docs - multipass.run](https://multipass.run/docs)  
[Key-based authentication in OpenSSH for Windows - microsoft-com](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)  
