In this document, we will see how to deploy Gatus on Proxmox with Ansible and Docker, based on [those requirements](https://bitbucket.org/enroute-mobi/test-sysadmin/src/main/)
## Table of content
#todo to update
- [[#Technologies used|Technologies used]]
- [[#Create and configure a debian VM with Proxmox|Create and configure a debian VM with Proxmox]]
	- [[#Create and configure a debian VM with Proxmox#VM creation with Graphical proxmox web interface|VM creation with Graphical proxmox web interface]]
		- [[#VM creation with Graphical proxmox web interface#Requirements|Requirements]]
		- [[#VM creation with Graphical proxmox web interface#Steps|Steps]]
- [[#Deploy Gatus on the VM|Deploy Gatus on the VM]]
	- [[#Deploy Gatus on the VM#Generate a bcrypt hash to store the password|Generate a bcrypt hash to store the password]]
	- [[#Deploy Gatus on the VM#Generate selfsigned certificate for Gatus|Generate selfsigned certificate for Gatus]]
- [[#Create health checks to monitor a website|Create health checks to monitor a website]]
- [[#Open a public https access to the Gatus interface|Open a public https access to the Gatus interface]]
- [[#Potential improvment|Potential improvment]]
	- [[#Potential improvment#Proxmox|Proxmox]]
	- [[#Potential improvment#VM|VM]]
	- [[#Potential improvment#Gatus|Gatus]]
	
--- 
## Technologies used
Here is links for the documentation of the technologie used in this project :

| Technology                                                             | Description                                        |
| ---------------------------------------------------------------------- | -------------------------------------------------- |
| [Ansible](https://docs.ansible.com/projects/ansible/latest/index.html) | Agentless IT automation via YAML playbooks         |
| [Proxmox](https://pve.proxmox.com/pve-docs/)                           | Virtualization platform                            |
| [Gatus](https://github.com/TwiN/gatus)                                 | Self-hosted  health dashboard                      |
| [Docker](https://docs.docker.com/)                                     | Build and run apps in portable containers          |
| [Docker Compose](https://docs.docker.com/compose/)                     | Multi-container Docker apps via a single YAML file |

---
## Create and configure a debian VM with Proxmox

### VM creation with Graphical proxmox web interface
#### Requirements
- Log in to the Proxmox server https://IP:PORT. (The default port is 8006).
- import the Debian image you want to install in Proxmox and verify the checksum to ensure you are using the official image.
#### Steps
- Create the VM with a right click on the Proxmox node, select your image and settings for this VM. enable

---
## Deploy Gatus on the VM
### Generate a bcrypt hash to store the password
```shell
htpasswd -nBC 12 {USERNAME}
echo -n {HASH} | base64 -w0  
```

---
## Create health checks to monitor a website

---
## Open a public https access to the Gatus interface

---
## Potential improvment
### Proxmox
- disable root login and create another administrator account
- change the default network port of proxmox
- provision VM with Terraform or ansible
### VM
- 
### Gatus
- do a small script that automate the creation of the base64 bacrypt hash.
- Use OIDC for the security
- use yaml anchor to avoid to repeat configuration