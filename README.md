# README
This project automates the deployment of [Gatus](https://github.com/TwiN/gatus), a lightweight self-hosted health-check dashboard, on a Debian OS. It follows the requirements described in [this specification](https://bitbucket.org/enroute-mobi/test-sysadmin/src/main/), and uses Ansible to prepare the host while Docker Compose runs Gatus in a container, secured with TLS and basic authentication.
> **In this documentation we will talk about the usage of ansible and docker-compose to run Gatus. To see how to configure Proxmox and the vm that run Ansible see [this documentation](./Configure%20Proxmox%20and%20the%20VM.md)**
## Table of content

- [Technologies used](#technologies-used)
- [Usage](#usage)
    - [Prerequisites](#prerequisites)
    - [Clone the repository](#clone-the-repository)
    - [Import the inventory and secret file or create it](#import-the-inventory-and-secret-file-or-create-it)
    - [Configuration](#configuration)
        - [secret.yml](#secretyml)
            - [How to Generate a bcrypt base64 hash for `vault_gatus_user_password`](#how-to-generate-a-bcrypt-base64-hash-for-vault_gatus_user_password)
        - [inventory.yml](#inventoryyml)
        - [Configure Gatus monitoring](#configure-gatus-monitoring)
    - [Install Gatus](#install-gatus)
    - [Launch Gatus with docker-compose](#launch-gatus-with-docker-compose)
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

___
## Usage
### Prerequisites
**You need access to a Debian host with admin rights and an internet connection.**
### Clone the repository
in the user home that admin the server, clone the following repository:
```shell
git clone https://github.com/csejault/en_route_test.git
```
### Import the inventory and secret file or create it
For security reason those files are not in the repository because it contain sensitive data. You can import it. If you don't have them you can create it by copying the data from the [configuration section](#configuration).
### Configuration
**Please restrict the access of secret.yml and inventory.yml**
#### secret.yml
**For security reason this file must be edited with `ansible-vault`. See [Ansible vault documentation](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html)**. 
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
### Install Gatus
Inside the ansible directory (cloned in this [section](#clone-the-repository)), launch the following command :
```shell
ansible-playbook playbook.yml -i inventory.yml --ask-vault-pass
```
Running the Ansible playbook will: 
- install Docker and Docker Compose on the target host,
- create the directories and files used by Docker Compose and Gatus, and set the correct permissions on them.

### Launch Gatus with docker-compose
Your Gatus is ready to lauch. You can go in the `vm_compose_dir` to lauch the following command :
```
sudo docker compose up -d
```

