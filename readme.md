
# Ansible playbooks


## Prerequisites for running ansible playbooks:

## **System (local):**

**Ansible >v2.9**

**Python preferably v3**

**setuptools**

sudo apt-get install python-setuptools

For osx:

`easy_install setuptools`

**pip or pip3**
sudo apt-get install python-pip

or

sudo apt-get install python3-pip

For osx:

`easy_install pip`

**sshpass:**

Mostly default present on linux.

For osx: 

`brew install http://git.io/sshpass.rb`

or   

`brew create https://sourceforge.net/projects/sshpass/files/sshpass/1.06/sshpass-1.06.tar.gz --force`
     `brew install sshpass`
 

## **Remote host:**

**sshd**

**Proxy**
If needed a proxy should be installed and configured:

```
proxy_settings:
  https_proxy: http://127.0.0.1:3128
  http_proxy: http://127.0.0.1:3128
  no_proxy: localhost, 127.0.0.1
```

## CA certificates
If other than standard CA certificates need to be installed, put them (in crt format with extention .crt) in the **certs** folder in the inventories/[develop|...|production] directory.

# Playbooks

## 1) Local preparation:

`ansible-playbook playbooks/create-vaults.yml -i inventories/[develop|...|production]`

(This step is only needed when the ssh-vault.yml or secrets-vault.yml files are not present.)

- Creates an ssh keypair in ~/ansible/keypairs/[develop|...|production], and creates an ssh-vault.yml, encrypted with the given password. This ssh-vault will be put in the inventories/local[develop|...|production]/group_vars/local/. Will ask for the vault password.
- Generates a random password, and creates an secrets-vault.yml, encrypted with the given password. This secrets-vault will be put in the inventories/[develop|...|production]/group_vars/all_servers/. Will ask for the vault password.

`ansible-playbook playbooks/setup-local-user.yml -i inventories/[develop|...|production] --vault-id develop-ssh@prompt`

(This step is only needed when the ~/ansible/keypairs/[develop|...|production]/ssh-key-from-vault.priv / .pub files are not present.)

- Creates ssh key files ~/ansible/keypairs/[develop|...|production]/ssh-key-from-vault.priv / .pub, that will be used for configuration of the remote user's authorized_keys set.
- Installs packages locally like pip module passlib

## 2) Remote user / authentication / basic

### Set up remote user for ansible deployments:

`ansible-playbook playbooks/setup-remote-user.yml -i inventories/[develop|...|production] --vault-id secrets-ssh@prompt`

- Creates a deployment user user defined by 'deployment_user' (for example: ansible)
- Copy the public key from ~/ansible/keypairs/[develop|...|production]/ssh-key-from-vault.pub to the authorized_keys set of the deployment user.
- Disables logging in with username / password for the deployment user.

After this step, you can login as ansible (or another user specified as 'deployment_user') to the hosts with `ssh ansible@<host> -i ~/ansible/keypairs/[develop|...|production]/ssh-key-from-vault.priv` 

### Install basic packages on the remote host:

`ansible-playbook playbooks/setup-remote-base.yml -i inventories/[develop|...|production] --vault-id secrets-ssh@prompt`

- Install CA certificates (*.crt) present in resources/develop/certs/
- Upgrades the system (and restarts when packages are updated)
- Installs a basic set of packages needed for docker etc, like aptitude, python3, some pips, LVM
- Installs docker-compose and docker-ce. Mounts specified LVM volume as the docker directory. The docker/volumes directory is also a mount point for a specified LVM logical volume (see settings in *all-servers.yml*). The idea is that `docker volume` should be used to create persistant data volumes.

## 3) Remote installation of a PostgreSQL database.

`ansible-playbook playbooks/setup-db.yml -i inventories/[develop|...|production] --vault-id secrets-ssh@prompt`

- Installs PostgreSQL
- Requests whether the container should be recreated. If 'no', no update is done.
- Requests whether the postgres-data volume for persistant data should be recreated a.k.a. deleted and created again.

# Help, it doesn't work

## **Hanging in the [Gathering Facts] task**
Sometimes after changing hosts in one of the hosts files, I experienced ansible-playbook to hang on gathering facts. What helped me was removing the .ansible folder in my home directory:

rm -rf ~/.ansible

## **error message like: no vault secrets found**
When the create-vaults.yml is run and this message appears normally means that there is already a vault-ssh.yml and/or vault-secrets.yml present because the playbook already had ran already before. Remove the files manually to generate them again.