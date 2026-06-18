# Configure Proxmox and the VM

This guide covers preparing the infrastructure that will host Gatus: configuring a Proxmox node, setting up an isolated NAT network, and creating and configuring the Debian VM, including exposing it to the internet over HTTPS. Once the VM is ready, head over to the main README for the Ansible and Docker Compose steps that actually deploy Gatus on it.
## Table of Contents

- [Prerequisites](#prerequisites)
- [Configure Proxmox](#configure-proxmox)
    - [Import debian image](#import-debian-image)
    - [Setup the network and active NAT](#setup-the-network-and-active-nat)
- [Create and configure a Debian VM with Proxmox](#create-and-configure-a-debian-vm-with-proxmox)
    - [VM creation with Graphical Proxmox web interface](#vm-creation-with-graphical-proxmox-web-interface)
        - [Creation](#creation)
    - [VM configuration](#vm-configuration)
        - [configure the network](#configure-the-network)
        - [remove de sources from the cdrom](#remove-de-sources-from-the-cdrom)
        - [add the source-list repository](#add-the-source-list-repository)
        - [Install qemu-guest-agent](#install-qemu-guest-agent)
        - [active qemu agent on Proxmox](#active-qemu-agent-on-proxmox)
        - [Install git and ansible and download the repository](#install-git-and-ansible-and-download-the-repository)
- [Activate PAT on Proxmox to expose Gatus](#activate-pat-on-proxmox-to-expose-gatus)

---
## Prerequisites
- A Proxmox VE node already installed and reachable.
- One network interface on the Proxmox host with a public ip.

---
## Configure Proxmox
- Log in to the Proxmox server https://IP:PORT. (The default port is 8006).
### Import debian image
- import the Debian image you want to install in Proxmox and verify the checksum to ensure you are using the official image.
### Setup the network and active NAT
**The following step will allow us to pass by Proxmox internet connection with our VM. Be really careful with those the next steps. Any wrong configuration can block the internet connection of our Proxmox at the next reboot.**: 
*In this exemple we will use the card name **`vmbr1`** with the ip 192.168.100.1/24*.
- Add a virtual bridge card vmbr1 to Proxmox.
- on the shell console, edit `/etc/network/interfaces` and add to the existing configuration the following lines(replace vmbr0 by the name of the network card that possess the public IP) :
```shell
...
auto vmbr1
        iface vmbr1 inet static
        address 192.168.100.1
        netmask 255.255.255.0
        bridge-ports none
        bridge-stp off
        bridge-fd 0

#NAT SETTINGS
post-up echo 1 > /proc/sys/net/ipv4/ip_forward
post-up iptables -t nat -A POSTROUTING -s '192.168.100.0/24' -o vmbr0 -j MASQUERADE
post-down iptables -t nat -D POSTROUTING -s '192.168.100.0/24' -o vmbr0 -j MASQUERADE
```
- use `ifreload -a` to reload the network configuration and apply the configuration.

---
## Create and configure a Debian VM with Proxmox

### VM creation with Graphical Proxmox web interface
#### Creation
- Create the VM with a right click on the Proxmox node, select your image and settings for this VM. Use the network card we use in  [Setup the network and active NAT](#setup-the-network-and-active-nat)
### VM configuration
#### configure the network
- on the shell console, edit `/etc/network/interfaces` and add to the existing configuration the following lines. `eth0` is the name of the network card but can sometimes differ.
```
...
auto eth0
iface eth0 inet static
address 192.168.100.10
netmask 255.255.255.0
gateway 192.168.100.1
nameserver 8.8.8.8
nameserver 1.1.1.1
```
- restart the service
```
sudo systemctl restart networking
```

>**At this point our VM is connected to internet . Is is possible to deploy ssh key with cloud init to use a remote ssh connection. Note that you need to have [Activated the PAT on Proxmox](#activate-pat-on-proxmox-to-expose-gatus)**
#### remove de sources from the cdrom
remove the line in starting with `deb cdrom` in /etc/apt/sources.list
#### add the source-list repository
add the http Debian sources as recommended by the [Debian documentation](https://wiki.debian.org/SourcesList)
To use this format, create a file called like /etc/apt/sources.list.d/debian.sources :
```
Types: deb deb-src
URIs: http://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

#### Install qemu-guest-agent
this is optional but this program facilitate the console integration in the web browser and the communication between the VM and Proxmox
```shell
sudo apt update
sudo apt install -y qemu-guest-agent
```
#### active qemu agent on Proxmox
Shutdown the VM and activate the qemu agent in the VM option in Proxmox the start the vm again
#### Install git and ansible and download the repository
This step is to download the git repo and start using ansible for the configuration
```shell
sudo apt update
sudo apt install -y git ansible
```


---
## Activate PAT on Proxmox to expose Gatus
- **You will need a rule to allow https and another rule to allow ssh In this project it is not recommended to do so because there is no firewall to filter the incoming traffic.**
- To allow our VM to be accessible from outside we need to create a PAT rule.
 - on the shell console, edit `/etc/network/interfaces` and add to the existing configuration the following lines:
	 - replace vmbr0 by the name of the network card that possess the public IP
	 - `OUTSIDE IP/PORT` are the ip/port reachable
	 - `DEST IP/PORT` are the ip/port you want to give access to.
```shell

#PAT SETTINGS
post-up iptables -t nat -A PREROUTING -p tcp -d {OUTSIDE IP} --dport {OUTSIDE PORT} -i vmbr0 -j DNAT --to-destination {DEST IP}:{DEST PORT}
post-down iptables -t nat -D PREROUTING -p tcp -d {OUTSIDE IP} --dport {OUTSIDE PORT} -i vmbr0 -j DNAT --to-destination {DEST IP}:{DEST PORT}

```
- use `ifreload -a` to reload the network configuration and apply the configuration.