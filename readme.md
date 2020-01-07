
# Ansible playbooks


## Prerequisites for running ansible playbooks:

## **System (local):**

**SSH**

**OpenSSL**

**Ansible >v2.9**

**Python v3**

For OSX set the param interpreter_python in the ansible.cfg to other than 'auto' if needed.

**setuptools**

sudo apt-get install python-setuptools

For osx:

`easy_install setuptools`

**pip3**
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
 

**openssl**

On OSX the default (outdated) openssl/libressl is very slow in generating dhparams. 
As an alternative you can install a more recent version with homebrew:

`brew install openssl`

`brew link openssl --force`

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
If other than standard CA certificates need to be installed on the remote hosts, put them (in crt format with extention .crt) in the **resources/[develop|...|production]/cacerts** directory.


# Playbooks

## 1) Local preparation:

`ansible-playbook playbooks/create-vaults.yml -i inventories/[develop|...|production]`

(This step is only needed when the ssh-vault.yml, secrets-vault.yml or vault-ca-certs.yml files are not present.)

- Creates an ssh keypair in ~/ansible/keypairs/[develop|...|production], and creates an ssh-vault.yml, encrypted with the given password. This ssh-vault will be put in the inventories/local[develop|...|production]/group_vars/local/. Will ask for the vault password.
- Generates a random password, and creates an secrets-vault.yml, encrypted with the given password. This secrets-vault will be put in the inventories/[develop|...|production]/group_vars/all_servers/. Will ask for the vault password.
- generates a self signed CA certificate (for internal use) and a cetrtificate signed by the this CA. The keys and certificates are stored in the vault-ca-certs.yml file in the inventories/[develop|...|production]/group_vars/local directory. For the configuration of the generated certificates, please edit the conf files in /resources/[develop|...|production]/csr-configs directory to match your domain, names etc!

`ansible-playbook playbooks/setup-local-user.yml -i inventories/[develop|...|production] --vault-id [develop|...|production]-ssh@prompt`

(This step is only needed when the ~/ansible/keypairs/[develop|...|production]/ssh-key-from-vault.priv / .pub files are not present.)

- Extracts the internal CA certificate from vault-ca-certs.yml to the resources/[develop|...|production]/cacerts/
- Extracts the internal CA signed certificate from vault-ca-certs.yml to the resources/[develop|...|production]/certs/
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

- Install CA certificates (*.crt) present in resources/[develop|...|production]/cacerts/.
- Upgrades the system (and restarts when packages are updated)
- Installs a basic set of packages needed for docker etc, like aptitude, python3, some pips, LVM
- Installs docker-compose and docker-ce. Mounts specified LVM volume as the docker directory. The docker/volumes directory is also a mount point for a specified LVM logical volume (see settings in *all-servers.yml*). The idea is that `docker volume` should be used to create persistant data volumes.


## 3) Remote installation of an OpenLDAP service
Image used:
https://github.com/osixia/docker-openldap

For more about LDAP info see :
https://www.golinuxcloud.com/install-and-configure-openldap-centos-7-linux/
https://www.digitalocean.com/community/tutorials/how-to-change-account-passwords-on-an-openldap-server
https://www.digitalocean.com/community/tutorials/how-to-manage-and-use-ldap-servers-with-openldap-utilities
http://techiezone.rottigni.net/2011/12/change-root-dn-password-on-openldap/


`sudo docker exec openldap ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config olcDatabase=\*`
`sudo docker exec --interactive openldap ldapsearch -H ldapi:// -LLL -Q -Y EXTERNAL -b "cn=config"`

Some LDAP basic info: 

DIT: Directory Information Tree, root of a tree of entries stored in ldap

Show hashed passwords for the admin and config DIT:

`sudo docker exec openldap ldapsearch -H ldapi:// -LLL -Q -Y EXTERNAL -b "cn=config" "(olcRootDN=*)" dn olcRootDN olcRootPW`


Authentication to LDAP:
- Anonymous (no auth), normally read-only for public-facing DIT. (use ldapsearch ... -x)
- Simple: username is DN (distinguished name) + password
- SASL: Simple Authentication and Security Layer: Several mechanisms like *EXTERNAL*

Find the DIT root entry (using the anonymous binding):

`sudo docker exec openldap ldapsearch -H ldap://localhost -ZZ -x -LLL -s base -b "" namingContexts`

Will return something like:
```  
dn
namingContexts: dc=default,dc=nl
```
Find an simpleSecurityObject (used for uname/password config) entry to bind with under the DIT looked up above (use -b to bind to a specific DIT dn). Change the dc= parts to the domain parts given during the openLDAP container installation.

`sudo docker exec openldap ldapsearch -H ldap://localhost -ZZ -x -LLL -b "dc=example,dc=com" "(objectClass=simpleSecurityObject)" dn`

Will return something like:
```
dn: cn=admin,dc=example,dc=com
```

Performing actions using the SASL/EXTERNAL authentication method (only possible from within the container):

`sudo docker exec openldap ldapsearch -Y EXTERNAL -H ldapi:///`

Authentication on openLDAP with username (dn) and password:

`sudo docker exec --interactive openldap ldapsearch -H ldap://localhost -ZZ -x -D "cn=admin,dc=example,dc=com" -W`
    
or

`sudo docker exec openldap ldapsearch -H ldap://localhost -ZZ -x -D "cn=admin,dc=example,dc=com" -w admin`


Querying the current TLS certificate / key etc:
`sudo docker exec --interactive openldap ldapsearch -H ldapi:// -Y EXTERNAL -b cn=config '(|(olcTLSCertificateFile=*)(olcTLSCertificateKeyFile=*)(olcTLSCipherSuite=*))' olcTLSCertificateFile olcTLSCertificateKeyFile olcTLSCipherSuite olcTLSProtocolMin`

Modifying openLDAP settings in the Config DIT:
TODO


## 3) Remote installation of a PostgreSQL database.

`ansible-playbook playbooks/setup-db.yml -i inventories/[develop|...|production] --vault-id secrets-ssh@prompt`

- Requests whether the container should be recreated. If 'no', no update is done.
- Requests whether the postgres-data volume for persistant data should be recreated a.k.a. deleted and created again.
- Install PostgreSQL client (psql) on the host
- Installs PostgreSQL docker container

Get the version of the installed PostgreSQL:

`psql --host=localhost --username=postgres`

Set up LDAP using TLS as means to authenticate / authorize

Therefore several steps are needed:
- Copy CA certificate to docker data volume
- Copy CA certificate to file specified by TLS_CACERT in /etc/ldap/ldap.conf (assumed to be /etc/ssl/certs/ca-certificates.crt), else TLS errors will occur because the certificate of the ldap server will not be trusted.
- modify pg_hba.conf: add something like:

```
host    all     all     0.0.0.0/0       ldap    ldapserver="openldap-server.test.internal.nnworks.nl" ldapprefix="cn=" ldapsuffix=",dc=example,dc=com" ldaptls=1
```


# Help, it doesn't work

## **Hanging in the [Gathering Facts] task**
Sometimes after changing hosts in one of the hosts files, I experienced ansible-playbook to hang on gathering facts. What helped me was removing the .ansible folder in my home directory:

rm -rf ~/.ansible

## **error message like: no vault secrets found**
When the create-vaults.yml is run and this message appears normally means that there is already a vault-ssh.yml and/or vault-secrets.yml present because the playbook already had ran already before. Remove the files manually to generate them again.

## **pip3 installing modules gives a failure**
Sometimes installing the passlib or cryptography modules fail (seg fault of pip3). What helped me was removing the `~/.local/lib/python3.x` directory. 