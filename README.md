ssh-ca

How to use SSH Certificate Authority to centralize management of host keys with Ansible


Introduction

When using ssh to connect to hosts, whether we know it or not, the use of some asymmetric key pair is behind the scene for encryption and authentication purposes. The key pair can be used for user authentication (user key) or machine authentication (host key). But what might not be obvious to most of us is that, as early as 2010, openSSH has added certificate authority (CA) support as part of the bundle. Utilizing CA support in ssh is especially beneficial to automating provisioning using ansible which relies on ssh.    

When configuring key-based authentication in ssh, the authorized_keys file (as in /home/ansible/.ssh) on the remote machine handles user key authentication while the known_hosts file (as in /home/ansible/.ssh) on the local ansible server where you initiate ssh connection handles host key authentication. While CA can be used for either or both user and host authentication as mentioned, we will be focusing only on the use of host key authentication, as there are many existing articles talking about user key management in great length using different approaches. Adding CA into ansible makes it even more compelling when provisioning a cluster of hosts at scale, especially when they come and go often reusing the same IP address. Ansible users or any long time ssh users would not have to be reminded the annoyance of host key-related errors and warnings interrupting their workflow.

Instead of trusting the host key one by one individually and maintaining the keys in known_hosts file, we can get the trusted certificate authority to handle all the trusting of the host keys representing the remote hosts in one place, in a centralized manner. The result is the much simplified host key management: We trust only the host key of the authority, namely the CA. This works exactly the same way as the SSL certificates that we have been used to over the Internet. When we go to amazon, paypal, or our online banks online via https, we don't need to directly trust their individual certificate. We only need to trust the entity that issues them the certificates - think Verisgn or Thawte. So let's get started.


Prerequisite and assumptions

* download a copy of this repo:

      git clone git://github.com/sc-sf/ssh-ca.git

* centos 6.5
* hostnames can be resovled by either dns or hosts file
* ansible version 1.7.2 or above
* the "ansible" user is already there on all machines
* the hosts defined in inventory file should be fqdn; you must edit this file to reflect your hosts

     # file: inventory_file

     [myservers]
     server1.my.local
     server2.my.local


The playbooks included here actually contain mostly ad-hoc shell commands. It's imaginable that it won't be long before someone come up with a full blown ansible module for key signing purpose.


Step 1: Create key and certificate for the CA and make ansible server trust any keys signed by CA

The ssh_ca_signning.yml playbook is run for creating and signing of the host key. But before we can sign keys, we need to create the key pair for the CA server itself. This is where the ssh_create_ca_key.yml playbook comes in handy. You need to run this only the first time. It also copies the content of the newly generated host_ca.pub into ansible user's known_hosts file.
Notice the string "@cert-authority * " is inserted right in the beginning. 


     # file: ssh_create_ca_key.yml
     ---
     - hosts: localhost
       sudo: yes
       vars:
         ca_key_name: host_ca
  
       tasks:
       - name: turn off the host key checking only once for initial key deployment
         local_action: shell export ANSIBLE_HOST_KEY_CHECKING=False

       - name: creating a host key on the ansible server
         local_action: shell chdir=/etc/ssh ssh-keygen -f {{ca_key_name}} -q -N ""

       - name: copy root ca's pub key to ansible's known_hosts file
         local_action: shell echo "@cert-authority * $(cat /etc/ssh/{{ca_key_name}}.pub)" \
           > /home/ansible/.ssh/known_hosts



We are now ready to create and sign keys. Because ssh_create_ca_key.yml disables host key checking in ansible, we won't be getting the ignoring warnings the first time when trying to get the host keys created and signed. Ansible allows the ssh's host key checking mechanism be disabled by either in ansible.cfg or exporting the environment variable like in the above playbook. It's wise setting it in ansible.cfg as this would make it too easy to just leave it turned off permanently, creating a security concern.


Step 2: Create and sign host key for each remote machine

Because we will name each host key after the target host's fqdn, and initially store in the cert_dir, we make sure that there's no pre-existing keys with the same name by clearing the dir. Then the fqdn (the inventory_hostname variable) will be used as the name of the host key and cert. This assumes the hosts entries in ansible inventory file is fqdn, as mentioned above, specifying an empty passphrase. We then sign the key using the host_ca key specified by the -s switch. The result would also generate the certificate file ending with -cert.pub. Please note the use of -Z switch in here. It specifies the names of the principles that can use this certificate. For example, you can specify that server1 and server1.my.local as principles. What this means is that it will not work if this cert is used by say server12 or server12.my.local. Also note centos (or at least version 6.5) mistakenly use -Z to designate principles while all other distributions use -n for the same purpose. Please keep this in mind when applying to distributions
other than centos.


     # file: ssh_ca_signning.yml
     ---
     - hosts: all
       remote_user: ansible
       sudo: yes
       vars:
         cert_dir: /tmp
         ca_key_dir: /etc/ssh
         certlogger: cert-signing.log
         sshd_cfg: sshdconfig.j2

       tasks:
       - name: make sure no preexisting keys conflicting with new ones
         local_action: shell rm -f {{cert_dir}}/{{inventory_hostname}}*

       - name: generate host key pair # -q=silience; -N ""=no passphrase
         local_action: shell ssh-keygen -f {{cert_dir}}/{{inventory_hostname}} -q -N "" 

       - name: sign key and generate certificate  
         local_action: shell ssh-keygen -s {{ca_key_dir}}/host_ca -I {{inventory_hostname}} \ 
           -Z {{ansible_hostname}},{{inventory_hostname}} \
           -h -V +104w {{cert_dir}}/{{inventory_hostname}}.pub
         register: eventlog

       - name: log the key signing event
         local_action: shell echo {{eventlog}} >> {{certlogger}}


So now we are ready to copy the newly generated key and certificate over. There's a bug in ansible (the current version in use) where the copy module (even with sudo) is given permission denied error message and thus prevented access to file having 600 permission of another owner. The temp fix here is to change permission from 600 to 644 before copying and then change it back afterward.
The bug is described in here - https://github.com/ansible/ansible/issues/6948


       - name: change permission of private key before copying
         local_action: shell chmod 644 {{cert_dir}}/{{inventory_hostname}} 

       - name: copying private key and change permission back to 600
         copy: src={{cert_dir}}/{{inventory_hostname}} dest=/etc/ssh force=yes mode=600

       - name: copying certificate and leave permission as 644
         copy: src={{cert_dir}}/{{inventory_hostname}}-cert.pub dest=/etc/ssh force=yes

       - name: updating sshd_config 
         template: src={{sshd_cfg}} dest=/etc/ssh/sshd_config owner=0 group=0 mode=600
         notify:
         - restart sshd

       handlers:
         - name: restart sshd
           service: name=sshd state=restarted


Make sure we make changes to the /etc/ssh/sshd_config file only after the copying or else it would interrupt or terminate the current ssh connection. In sshd_config file, the two added new lines are:

     
     HostKey /etc/ssh/{{inventory_hostname}}
     HostCertificate /etc/ssh/{{inventory_hostname}}-cert.pub


With all remote hosts configured with this playbook and the properly formatted host_ca pub key in your local known_hosts, you will connect to any remote hosts without getting host authentication prompt at all. This is true whether you are reissuing certificates to the existing hosts or new hosts, as long as their keys are signed with the same CA key.



