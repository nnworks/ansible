
# Ansible playbooks


## Prerequisites for running ansible playbooks:

## **System (local):**

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
 

## Python modules:
(or run playbook setup-local.yml)

**passlib (local):**

pip install --user passlib



## **Remote host:**

**sshd**

**Proxy**
If needed a proxy should be installed and configured.

(http_proxy, https_proxy)

**CA certificates**
If needed additional CA certificates should be installed and configured.


## CA certificates
If other than standard CA certificates need to be installed, put them (in crt format with extention .crt) in the **certs** folder in the inventories/[develop|...|production] directory.

**LVM2**


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

`ansible-playbook playbooks/setup-remote-user.yml -i inventories/[develop|...|production] --vault-id secrets-ssh@prompt`

- Creates an ansible user 'ansible'
- Copy the public key from ~/ansible/keypairs/[develop|...|production]/ssh-key-from-vault.pub to the authorized_keys set of the ansible user.
- **TODO:** Disables logging in with username / password for the ansible user.

After this step, you can login to the hosts with `ssh ansible@<host> -i ~/ansible/keypairs/develop/ssh-key-from-vault.priv` 

`ansible-playbook playbooks/setup-remote.yml -i inventories/[develop|...|production] --vault-id secrets-ssh@prompt`

- Install CA certificates (*.crt) present in resources/develop/certs/
- Upgrades the system (and restarts when packages are updated)
- Installs a basic set of packages needed for docker etc, like aptitude, python3, some pips, docker-compose and docker-ce


# Help, it doesn't work

## **Hanging in the [Gathering Facts] task**
Sometimes after changing hosts in one of the hosts files, I experienced ansible-playbook to hang on gathering facts. What helped me was removing the .ansible folder in my home directory:

rm -rf ~/.ansible

## **error message like: no vault secrets found**
When the create-vaults.yml is run and this message appears normally means that there is already a vault-ssh.yml and/or vault-secrets.yml present because the playbook already had ran already before. Remove the files manually to generate them again.