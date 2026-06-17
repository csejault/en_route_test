This project automates the deployment of [Gatus](https://github.com/TwiN/gatus), a lightweight self-hosted health-check dashboard, on a Debian virtual machine hosted on Proxmox. It follows the requirements described in [this specification](https://bitbucket.org/enroute-mobi/test-sysadmin/src/main/), and uses Ansible to prepare the host while Docker Compose runs Gatus in a container, secured with TLS and basic authentication.
## Table of content
#todo to update
- [Technologies used](#technologies-used)
- [Usage](#usage)
  - [Clone the repository](#clone-the-repository)
  - [Import the inventory and secret file or create it](#import-the-inventory-and-secret-file-or-create-it)
  - [Configuration](#configuration)
    - [secret.yml](#secretyml)
      - [How to Generate a bcrypt base64 hash for `vault_gatus_user_password`](#how-to-generate-a-bcrypt-base64-hash-forvault_gatus_user_password)
    - [inventory.yml](#inventoryyml)
    - [Configure Gatus monitoring](#configure-gatus-monitoring)
  - [Deploy Gatus](#deploy-gatus)
  - [Launch Gatus with docker-compose](#launch-gatus-with-docker-compose)
- [Configure Proxmox](#configure-proxmox)
  - [Import debian image](#import-debian-image)
  - [Setup the network and active NAT](#setup-the-network-and-active-nat)
- [Create and configure a debian VM with Proxmox](#create-and-configure-a-debian-vm-with-proxmox)
  - [VM creation with Graphical proxmox web interface](#vm-creation-with-graphical-proxmox-web-interface)
    - [Creation](#creation)
  - [VM configuration](#vm-configuration)
    - [configure the network](#configure-the-network)
    - [add the source-list repository](#add-the-source-list-repository)
    - [Install qemu-guest-agent](#install-qemu-guest-agent)
    - [active qemu agent on Proxmox](#active-qemu-agent-on-proxmox)
    - [Install git and ansible and download the repository](#install-git-and-ansible-and-download-the-repository)
- [Open a public https access to the Gatus interface](#open-a-public-https-access-to-the-gatus-interface)
  - [Activate PAT on Proxmox](#activate-pat-on-proxmox)
- [Potential improvment](#potential-improvment)
  - [Proxmox](#proxmox)
  - [VM](#vm)
  - [Gatus](#gatus)
  - [Ansible](#ansible)
  - [Architecture](#architecture)
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

___
## Usage
### Clone the repository
in the user home that admin the server, clone the following repository:
```shell
git clone git@github.com:csejault/en_route_test.git
```
### Import the inventory and secret file or create it
For security reason those files are not in the repository because it contain sensitive data. You can import it. If you don't have them you can create it by copying the data from the [configuration section](#configuration).
### Configuration
#### secret.yml
**For security reason this file must be edited with `ansible-vault`. See [Ansible vault documentation](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html)**
*location of the file : inside the ansible dir*
```yaml
# password of the admin user
ansible_sudo_pass:

# password for the gatus user. This must be bcrypt and base64 encoded
vault_gatus_user_password: 

# Webhook used to send alert via discord
vault_gatus_alert_discord_webhook_url: 

# Private key used by gatus to enable tls
vault_gatus_tls_private_key: |
  -----BEGIN PRIVATE KEY-----
  ...
  -----END PRIVATE KEY-----

# Certificate used by gatus to enable tls
vault_gatus_tls_certificate: |
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----

```
##### How to Generate a bcrypt base64 hash for`vault_gatus_user_password`
I used `htpasswd` commands by running a docker container and install apache2-utils
```shell
htpasswd -nBC 12 {USERNAME}
echo -n {HASH} | base64 -w0  
```

---
#### inventory.yml
*location of the file : inside the ansible dir*
```yaml
all:
  hosts:
    localhost:
	  # Type of connection (local if run ansible on the server to configure)
      ansible_connection: local
	  # Tls certificate name for gatus
      gatus_cert_name: XXX
	  # port exposed by the server
      gatus_ext_port: XXX 
	  # port exposed by gatus(inside docker)
      gatus_int_port: XXX 
	  # Tls private key name for gatus
      gatus_private_name: XXX
	  # gatus username (used to connect to gatus)
      gatus_user: XXX
	  # architecture of the server
      vm_architecture: XXX
	  # Directory where docker compose dir and gatus dir will be store
      vm_app_dir: XXX
	  # Directory where the docker-compose files will be store
      vm_compose_dir: XXX
	  # Directory where the gatus files will be store
      vm_gatus_dir: XXX
	  # Username of the admin user on the server
      vm_user: XXX
```
#### Configure Gatus monitoring
It is possible to change the monitoring by modifying the [Gatus configuration file](./ansible/files/gatus_config.yaml). See [Gatus documentation](#technologies-used).
### Deploy Gatus
Inside the ansible directory (cloned in this [section](#clone-the-repository)), launch the following command :
```shell
ansible-playbook playbook.yml -i inventory.yml --ask-vault-pass
```
Running the Ansible playbook will: 
- install Docker and Docker Compose on the target host,
- create the directories and files used by Docker Compose and Gatus, and set the correct permissions on them.

### Launch Gatus with docker-compose

#todo reverify playbook to check if we put the user in sudo group
```
sudo docker-compose up #todo 
```

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
- create a script to install and configure the requirment for ansible
- create a gatus user (in docker group) without sudo right to do the docker compose up
- disable ssh (and activate from console if needed)
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
- put a firewall between vmbr0 and vmbr1 to filter traffic
- create a postgres db to store gatus logs.