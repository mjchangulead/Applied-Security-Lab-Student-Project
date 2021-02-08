# ASL20 Group5

Our solution to the course project of the Applied Security Lab 2020 @ ETHZ

Requirements: vagrant and virtualbox

## Vagrant

1. To start the machines use: vagrant up
2. To restart the machines after a configuration change, use: vagrant destroy -f & vagrant up
3. In case the change is in the playbook or the SHELL, there is no need to destroy but just to re-execute the provisions, you can use: vagrant up --provision
4. To connect to a machine use: vagrant ssh hostname
5. To suspend the machines, save the state / shut them down in virtualbox or use: vagrant halt / vagrant suspend

All machines are spawn and can be managed in virtualbox's panel

## Ansible

Ansible allows the installation and managing of machines automatically. Ansible uses .yml files called playbook to do so. Check shared/ansible/list.yml for an example for this setup. The requirements are preinstalled according to requirements.yml

MySQL installed using ansible role: https://github.com/idealista/mysql_role

Vars folder contains imports for list.yml

## Manage a system component

To ssh and configure a certain host with sudo capabilities:

1. First, ssh to the ansibleadmin node using: vagrant ssh ansibleadmin
2. Set the admin user with password "CKLwhksWtn": su admin
3. Change the user to ansible by executing with the same password "CKLwhksWtn": sudo su ansible
4. a) Then ssh to a host by using: sshpass -v -P "Enter" -p $(cat /home/passphrase) ssh ansible@hostname
4. b) Additionally you can introduce the passphrase manually: sshpass -v -P "Enter" -p "UURf_Uko5s0qDxEkhKkX0A6lGAJO1WzWWy2XJMJd" ssh ansible@hostname

Additionally, we can ssh to admin@ansibleadmin from the GUI Client by:
1. First set user to admin with password "admin": su admin
2. Then ssh to the ansibleadmin by executing: ssh -i /home/vagrant/sshkey_store/client/imovies_ansibleadmin admin@ansibleadmin
3. Follow steps 3 to 4 in the previous list

Additionally, you can execute playbooks from ansible@ansibleadmin:
1. ansible-playbook -e 'FORCE_ROOT_CA_CERT_REGEN=true' -i inventory list.yml

Configuration can be found in Vagrantfile constants are in /shared/ansible/ansible_passphrase.txt

## Frontend

The frontend source can be found in the frontend folder. The packed files as .html, .js and .css can be found in the frontend/dist folder and can be served directly. 

## Database

The database is created, executed, and dumped by ansible and runs as a daemon using the default MySQL port "3306"

The MySQL DB accepts connections from the private network (Ips "192.168.33.*"). Right now there is a unique DB called imovies with a unique table called users.

The user to configure this DB is "imovies_user_private" with password "mysecret_private"

Additionally, the root user is "root" with password "IqTFqZbhbY"

You can try it by executing: 
1. mysql -u root -D imovies -p'IqTFqZbhbY'
2. mysql\> SELECT * FROM users; 

From webserver host (needs the mysql-client installed):
1. mysql -u imovies_user_private -p'mysecret_private' -h dbhost -D imovies -P 3306
2. mysql\> SELECT * FROM users;

The node can be accessed as dbhost or 192.168.33.25. 

The node only currently accepts remote accesses from the webserverhost using imovies_user_private user and mysecret_private password. Or by all private remote nodes using imovies_user_all user and mysecret_all password.

The DB as an additional column for CA admins called 'admin' that is 1 for CA Admins and 0 for the rest.

Configuration can be found in shared/ansible/list.yml and shared/ansible/vars/db.yml

The loggs can be found in /var/log/mysql

**Install Webserver at webserverhost machine**

Ansible is configured to deploy webserver on webserverhost.
Currently on port :80 (currently using a private network ip for testing purposes)
The node can be accessed as webserverhost or 192.168.33.35 or from the client by imovies.ch or www.imovies.ch

You can test it using: curl 192.168.33.35 

1. login webserverhost: $vagrant ssh webserverhost
2. get the source code: $git clone https://gitlab.ethz.ch/jasonf/asl20-group5.git
3. Install packages: 
   1) $sudo apt-get update -y
   2) $sudo apt-get upgrade -y
   3) $sudo apt-get install python3 python3-pip tree -y
   4) $pip3 install Django
   5) $django-admin --version
4. go to folder ./webserver
5. run server: $ python3 manage.py runserver 192.168.33.35:8000

If at step 3.5, django-admin is not found, please add django PATH which is shown after Django package installed: $echo 'export PATH="$PATH":/home/vagrant/.local/bin' >>  ~/.profile

## Core Ca Rest Api

### Deploy

Ansible is configured to deploy core_ca on certhst.
Currently on 192.168.33.15:8000 

Production environment is served by nginx server which is itself served by 
gunicorn wsgi server.

shell script that starts the wsgi server and flask application is located in:
`core_ca/core_ca.sh`

The config file for the wsgi server is located at `core_ca/gunicorn.conf.py`

### Local Test Setup

To run the ca rest API locally for testing 
To start the core CA rest API service run (Starting the service like this will adopt existing ca, existing certificates and private keys): 
`python ca_api.py`

To create new root and intermediate ca and certificates (clean start): 

1. delete the `/core_ca/certs`dir
2. run `python setup.py`
3. start core CA rest API `python ca_api.py`

Start Testrun to see if the local setup works:

`python test.py`

### API
For example, requests look at `core_ca/test.py`

CRL: new_cert and revoke_cert return also the certificate revocation list crl as a pem file
this is just in case you want to be able to verify certificates in the webserver directly without
consulting the Core CA.

ARCHIVE: PKCS12 archive encrypted with the passphrase holding the ca trust chain, the user certificate, the user privatekey. It is 
base64 encoded b.c. we cannot send bytearrays in a json object. 
To get recreate the PKCS12 archive object from the base64 string use the
code for that in `core_ca/test.py`.

* `/new_cert` issue new certificate. POST Args: `user_id, mail_addr, passphrase`
   returns: a base64 encoded PKCS12 file containing CA chain of trust, user certificate and user private key.
   Instructions on how we can decode this file back to a pkcs12 object are given in test.py
* `/revoke_cert` revoke some certificate. POST Args: `user_id` returns: status.
* `/verify_cert` check if a certificate is valid. POST Args: `certificate` returns: certificate status.
* `/ca_stats`get admin information on status of ca. GET 

### Logs

Gunicorn wsgi server logger is configured to take in also the logs of the flask app. 
They are stored in `core_ca/log/`.

**Backup**: the dirs `core_ca/log`and `core_ca/certs` should be backed up.