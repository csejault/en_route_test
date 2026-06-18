This is a list of potential improvement for ansible, docker compose, gatus and the architecture
## Potential improvement
### Proxmox
- disable root login and create another administrator account and only allow a connection with FIDO2 ssh key.
- change the default network port of proxmox
- provision VM with Terraform or ansible or une cloud init
### VM
- chose a better password
- use sudo instead of root
- use lvm and partition the disk
- use cloud init with a clone and then create our vm from the clone (like this we can, set ip and deploy ssh key as soon as the vm start)
- create a script to install and configure the requirment for ansible
- create a gatus user (in docker group) without sudo right to do the docker compose up
- disable ssh (and activate from console if needed)
- start vm when Proxmox reboot
### Gatus
- do a small script that automate the creation of the base64 bacrypt hash.
- Use OIDC for the security
- use yaml anchor to avoid to repeat configuration
- set PostgreSQL to store the logs
- use let's encrypt certificate with a real domain name.
- lauch docker compose up at startup of the VM
- do a doc to update gatus
### Ansible
- use ansible-core and install only the required ansible collection
- use hashicorp vault instead of a local vault
- use a role for the installation of docker-compose to reuse it in other playbook
- use a control node outside of the vm
### Architecture
- use a reverse proxy before Gatus
- put a firewall between vmbr0 and vmbr1 to filter traffic
- create a postgres db to store gatus logs.