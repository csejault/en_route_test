In this document, we will see how to deploy Gatus on Proxmox with Ansible and Docker, based on [those requirements](https://bitbucket.org/enroute-mobi/test-sysadmin/src/main/)
## Table of content
#todo to update
	
--- 
## Technologies used
Here is links for the documentation of the technologie used in this project :

| Technology                                                             | Description                                        |
| ---------------------------------------------------------------------- | -------------------------------------------------- |
| [Ansible](https://docs.ansible.com/projects/ansible/latest/index.html) | Agentless IT automation via YAML playbooks         |
| [Debian](https://wiki.debian.org/)                                     | Operating system used in this project              |
| [Docker](https://docs.docker.com/)                                     | Build and run apps in portable containers          |
| [Docker Compose](https://docs.docker.com/compose/)                     | Multi-container Docker apps via a single YAML file |
| [Gatus](https://github.com/TwiN/gatus)                                 | Self-hosted  health dashboard                      |
| [Proxmox](https://pve.proxmox.com/pve-docs/)                           | Virtualization platform                            |

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
## Create and configure a debian VM with Proxmox

### VM creation with Graphical proxmox web interface
#### Creation
- Create the VM with a right click on the Proxmox node, select your image and settings for this VM. Use the network card we use in  [Setup the network and active NAT](#setup-the-network-and-active-nat)
### VM configuration
#### configure the network
- on the shell console, edit `/etc/network/interfaces` and add to the existing configuration the following lines. `eth0` is the name of the network card but can sometimes differ.
```
...
auto eth0
iface eth0 inet static
address 192.168.100.10/24
netmask 255.255.255.0
gateway 192.168.100.1
nameserver 8.8.8.8
nameserver 1.1.1.1
```
- reboot or reload the service
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

>**At this point our VM is connected to internet and can install programs from Debian repository**
#### Install qemu-guest-agent
this is optional but this program facilitate the console integration in the web browser and the communication between the VM and Proxmox
```shell
sudo apt update
apt install -y qemu-guest-agent
```
#### active qemu agent on Proxmox
Shutdown the VM and activate the qemu agent in the VM option in Proxmox the start the vm again

#### Install git and ansible and download the repository
This step is to download the git repo and start using ansible for the configuration
```shell
sudo apt update
apt install -y git ansible
```

#todo clone repo
git clone git@github.com:csejault/en_route_test.git

## Lauch ansible
```shell
ansible-playbook playbook.yml -i hosts.ini --ask-vault-pass
```

## Launch docker-compose
sudo docker-compose up #todo 

---
## Deploy Gatus on the VM
### Generate a bcrypt hash to store the password
```shell
htpasswd -nBC 12 {USERNAME}
echo -n {HASH} | base64 -w0  
```
You can use this Hash in the security field of the [config_file](./config/config.yaml)

---
## Create health checks to monitor a website

---
## Open a public https access to the Gatus interface
### Activate PAT on Proxmox
- To allow our VM to be accessible from outside we need to create a PAT rule. On Proxmox shell, launch the following command where `OUTSIDE IP/PORT` are the ip/port reachable and `DEST IP/PORT` are the ip/port you want to give access to :
```shell
iptables -t nat -A PREROUTING -p tcp -d {OUTSIDE IP} --dport {OUTSIDE PORT} -i vmbr0 -j DNAT --to-destination {DEST IP}:{DEST PORT}
```
- **You will need a rule to allow https and another rule if you want to give access to ssh. In this project it is not recommended to do so because there is no firewall to filter the incoming traffic.**
Dont forget to save the new configuration. Otherwise it will be lost at reboot time :
```shell
iptables-save > /etc/iptables.conf
```


---
## Potential improvment
### Proxmox
- disable root login and create another administrator account and only allow a connection with FIDO2 ssh key.
- change the default network port of proxmox
- provision VM with Terraform or ansible
### VM
- chose a better password
- use lvm and partition the disk
- use cloud init with a clone and then create our vm from the clone (like this we can, set ip and deploy ssh key as soon as the vm start)
- Disable root SSH login
### Gatus
- do a small script that automate the creation of the base64 bacrypt hash.
- Use OIDC for the security
- use yaml anchor to avoid to repeat configuration
- set PostgreSQL to store the logs
- use let's encrypt certificate with a real domain name.
### Ansible
use ansible-core and install only the required ansible collection
use hashicorp vault instead of a local vault
### Architecture
- use a reverse proxy before Gatus
- put a firewall between vmbr0 and vmbr1

### This doc
follow guidelines for table of content : https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax