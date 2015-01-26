How to use SSH Certificate Authority to centralize management of host keys with Ansible

Download a copy of this repo via this command:

     git clone git://github.com/sc-sf/ssh-ca.git



Create key and certificate for the CA and make ansible server trust any keys signed by CA

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
Create and sign host key for each remote machine

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

   - name: generate host key pair # -q = silience; -N ""= no passphrase
     local_action: shell ssh-keygen -f {{cert_dir}}/{{inventory_hostname}} -q -N "" 

   - name: sign key and generate certificate  
     local_action: shell ssh-keygen -s {{ca_key_dir}}/host_ca -I {{inventory_hostname}} \ 
        -h -V +104w {{cert_dir}}/{{inventory_hostname}}.pub \
        #-Z {{ansible_hostname}},{{inventory_hostname}} 
     register: eventlog

   - name: log the key signing event
     local_action: shell echo {{eventlog}} >> {{certlogger}}

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


Changes to the /etc/ssh/sshd_config file

 HostKey /etc/ssh/{{inventory_hostname}}
 HostCertificate /etc/ssh/{{inventory_hostname}}-cert.pub



