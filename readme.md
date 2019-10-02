
## Ansible playbooks



# prerequisites for running ansible playbooks:

## System (local):

**Python preferably v3**

**setuptools**

sudo apt-get install python-setuptools

For osx:

`easy_install setuptools`

**pip**
sudo apt-get install python-pip

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




## Remote host:

**sshd**

**Proxy**
If needed a proxy should be installed and configured.

(http_proxy, https_proxy)

**CA certificates**
If needed additional CA certificates should be installed and configured.