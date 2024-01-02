# Lab_ansible-9.0
Ansible Automation Platform Playbook



_____________________________________________________________________________________________________________________________________________________________________________________

AAP - ANSIBLE AUTOMATION PLATFORM PLAYBOOK TEMPLATES

ANSIBLE NAVIGATOR:

setting up the ansible-navigator:
#sudo dnf install python3-pip
# python3 -m pip install ansible-navigator --user ---------- to install the ansible-navigator
# echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.profile  --- Add the installation path to the user shell initialization file (e.g.):
# source ~/.profile --------- refresh the path
# ansible-navigator ------- lunch the ansible-navigator



##Ansible NAVIGATOR:
- sudo dnf install ansible-navigator
- ansible-navigator --version
- podman login registry.redhat.io or podman login utility.lab.example.com ---- to login to the container registry
- ansible-navigator images 

- ansible-navigator inventory -i inventory -m stdout --graph [hostname] ----- to a host
- ansible-navigator inventory -i inventory -m stdout --list ------ to list all managed host
- ansible-navigator inventory -i inventory

##setting the ansible container: 
ansible-navigator:
  execution-environment: 1
    image: utility.lab.example.com/ee-supported-rhel8:latest 2
    pull:
      policy: missing 3
  playbook-artifact:
    enable: false 4
	
---
- name: Add users
  hosts: dev

  tasks:

    - name: Add the users joe and sam
      ansible.builtin.user:
        name: "{{ item }}"
      loop:
        - joe
        - sam	
---
- name: Install packages
  hosts: dev
  vars:
    packages:
      - httpd
      - mariadb-server

  tasks:

    - name: Install the required packages
      ansible.builtin.dnf:
        name: "{{ packages }}"
        state: latest
		
    - name: Install redis
      ansible.builtin.dnf:
        name: redis
        state: latest
      when: ansible_facts['swaptotal_mb'] > 10
	  
Correct the spelling:	  
---
- name: Verify the sam user was created
  hosts: dev

  tasks:

    - name: Verify the sam user exists
      ansible.builtin.user:
        name: sam
      check_mode: yes
      register: sam_check

    - name: Sam was created
      ansible.builtin.debug:
        msg: "Sam was created"
      when: sam_check['changed'] == false

    - name: Output sam user status to file
      ansible.builtin.lineinfile:
        path: /home/student/verify.txt
        line: "Sam was created"
        create: yes
      when: sam_check['changed'] == false
	   	  
review-cr2:
---
- name: Install and configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Install httpd package
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: Start httpd service
      ansible.builtin.service:
        name: httpd
        state: started

    - name: Deploy configuration template
      ansible.builtin.template:
        src: templates/vhost.conf.j2
        dest: /etc/httpd/conf.d/vhost.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart httpd

    - name: Copy index.html
      ansible.builtin.copy:
        src: files/
        dest: "/var/www/vhosts/{{ ansible_facts['hostname'] }}/"
        owner: root
        group: root
        mode: '0644'

    - name: Ensure web server port is open
      ansible.posix.firewalld:
        state: enabled
        permanent: true
        immediate: true
        service: http

  handlers:
    - name: Restart httpd
      service:
        name: httpd
        state: restarted
		
		
get_web_content:
---
- name: Test web content
  hosts: workstation
  become: true

  tasks:
    - name: Retrieve web content and write to error log on failure
      block:
        - name: Retrieve web content
          ansible.builtin.uri:
            url: http://servera.lab.example.com
            return_content: yes
          register: content
      rescue:
        - name: Write to error file
          ansible.builtin.lineinfile:
            path: /home/student/review-cr2/error.log
            line: "{{ content }}"
            create: true
			
site.yml:
---
# Deploy web servers
- import_playbook: dev_deploy.yml
 
# Retrieve web content
- import_playbook: get_web_content.yml



review-cr3:
storage.yml:
-
- name: Configure storage on webservers
  hosts: webservers

  roles:
    - name: redhat.rhel_system_roles.storage
      storage_pools:
        - name: vg_web
          type: lvm
          disks:
            - /dev/vdb
          volumes:
            - name: lv_content
              size: 128m
              mount_point: "/var/www/html/content"
              fs_type: xfs
              state: present
            - name: lv_uploads
              size: 256m
              mount_point: "/var/www/html/uploads"
              fs_type: xfs
              state: present
			  			  
dev-users.yml:
---
- name: Create local users
  hosts: webservers
  vars_files:
    - pass-vault.yml
  tasks:
    - name: Add webdev group
      ansible.builtin.group:
        name: webdev
        state: present

    - name: Create user accounts
      ansible.builtin.user:
        name: webdev
        groups: webdev
        password: "{{ pwhash }}"

    - name: Modify sudo config to allow webdev members sudo without a --
      ansible.builtin.lineinfile:
        path: /etc/sudoers.d/webdev
        state: present
        create: yes
        mode: 0440
        line: "%webdev ALL=(ALL) NOPASSWD: ALL"
        validate: /usr/sbin/visudo -cf %s
		
ansible-navigator run -m stdout --playbook-artifact-enable false dev-users.yml --vault-id @prompt


for network.yml:
mkdir -pv group_vars/webservers
---
network_connections:
  - name: eth1
    type: ethernet
    ip:
      address:
        - 172.25.250.45/24
		

cron-job:
---
- name: Recurring cron job
  hosts: webservers
  become: true

  tasks:
    - name: Crontab file exists
      ansible.builtin.cron:
        name: Rotate HTTPD logs
        minute: "0"
        hour: "0"
        weekday: "*"
        user: devops
        job: "logrotate -f /etc/logrotate.d/httpd"
        cron_file: rotate_web
        state: present
		
to verify the cron job: ssh devops@servera "cat /etc/cron.d/rotate_web"

To import the playbooks: site.yml

---
- import_playbook: storage.yml
- import_playbook: dev-users.yml
- import_playbook: network.yml
- import_playbook: log-rotate.yml

to run site.yml:
ansible-navigator run -m stdout --playbook-artifact-enable false site.yml --vault-id @prompt

 {% for host in groups['all'] %}
 {{ hostvars[host]['ansible_facts']['default_ipv4']['address'] }} {{ hostvars[host]
 ['ansible_facts']['fqdn'] }} {{ hostvars[host]['ansible_facts']['hostname'] }}
 {% endfor %}



LABS: 
- name: Run the /opt/bin/makedb.sh command
  ansible.builtin.command:
    cmd: /opt/bin/makedb.sh
	
- name: Initialize the database
  ansible.builtin.command:
    cmd: /opt/bin/makedb.sh
    creates: /opt/db/database.db
	
Playbook: 
- name: /etc/hosts is up-to-date
  hosts: datacenter-west
  remote_user: automation
  become: true

  tasks:
    - name: server.example.com in /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: '192.0.2.42 server.example.com server'
        state: present
_______________________________________________________________________________________________________________________
ch3-lab:
>> lab data-review start
>> cd ~/data-review
>> touch playbook.yml
vim playbook.yml
---
- name: install and configure webserver with basic auth
  hosts: webserver
  vars:
    firewall_pkg: firewalld
    firewall_svc: firewalld
    web_pkg: httpd
    web_svc: httpd
    ssl_pkg: mod_ssl
    httpdconf_src: files/httpd.conf
    httpdconf_dest: /etc/httpd/conf/httpd.conf
    htaccess_src: files/.htaccess
    secrets_dir: /etc/httpd/secrets
    secrets_src: files/htpasswd
    secrets_dest: "{{ secrets_dir }}/htpasswd"
    web_root: /var/www/html
	
  tasks:
    - name: latest version of neccessary package installed
      yum: 
         name: 
             - "{{firewall_pkg}}"
             - "{{web_pkg}}"
             - "{{ssl_pkg}}"
         state: latest 
		
	- name: configure web service
      copy:
        src: "{{ httpdconf_src }}"
        dest: "{{ httpdconf_dest }}"
        owner: root
        group: root
        mode: 0644
		
	- name: secrets directory exists
      file:
        path: "{{ secrets_dir }}"
        state: directory
        owner: apache
        group: apache
        mode: 0500
		
	 - name: htpasswd file exists
       copy:
        src: "{{ secrets_src }}"
        dest: "{{ secrets_dest }}"
        owner: apache
        group: apache
        mode: 0400
		
   - name: .htaccess file installed in docroot
      copy:
        src: "{{ htaccess_src }}"
        dest: "{{ web_root }}/.htaccess"
        owner: apache
        group: apache
        mode: 0400
		
	- name: create index.html
      copy:
        content: "{{ ansible_facts['fqdn'] }} ({{ ansible_facts['default_ipv4']['address'] }}) has been customized by Ansible.\n"
        dest: "{{ web_root }}/index.html"
		
	- name: firewall service enabled and started
      service:
        name: "{{ firewall_svc }}"
        state: started
        enabled: true
		
	- name: open the port for the web server
      ansible.posix.firewalld: #the firewalld services
        service: https
        state: enabled
        immediate: true
        permanent: true
		
		
    - name: web service enabled and started
      service:
        name: "{{ web_svc }}"
        state: started
        enabled: true
		
- name: test web server with basic auth
  hosts: localhost
  become: no
  vars:    #for variables within 
    web_user: guest
  vars_files:    #to call in external variables
	 - vars/secret.yml
  tasks:
    - name: connect to web server with basic auth
      uri:
        url: https://serverb.lab.example.com
        validate_certs: no
        force_basic_auth: yes
        user: "{{ web_user }}"
        password: "{{ web_pass }}" #the "{{ web_pass }}" is a variable in the secret.yml
        return_content: yes
        status_code: 200
      register: auth_test
    - debug: 
	     var: auth_test.content
		 
# mkdir vars/secret.yml
# ansible-vault create vars/secret.yml --- to create an encrypted variable file

ansible-navigator run -m stdout playbook.yml --vault-id @prompt --- this prompts for the password


lab4:
---
- name: Playbook Control Lab
  hosts: webservers
  vars_files: vars.yml
  tasks:
    #Fail Fast Message
    - name: Show Failed System Requirements Message
      ansible.builtin.fail:
        msg: "The {{ inventory_hostname }} did not meet minimum reqs."
      when: >
        ansible_facts['memtotal_mb'] < min_ram_mb or
        ansible_facts['distribution'] != "RedHat"

    #Install all Packages
    - name: Ensure required packages are present
      ansible.builtin.dnf:
        name: "{{ packages }}"
        state: latest

    #Enable and start services
    - name: Ensure services are started and enabled
      ansible.builtin.service:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop: "{{ services }}"

    #Block of config tasks
    - name: Setting up the SSL cert directory and config files
      block:
        - name: Create SSL cert directory
          ansible.builtin.file:
            path: "{{ ssl_cert_dir }}"
            state: directory

        - name: Copy Config Files
          ansible.builtin.copy:
            src: "{{ item['src'] }}"
            dest: "{{ item['dest'] }}"
          loop: "{{ web_config_files }}"
          notify: restart web service

      rescue:
        - name: Configuration Error Message
          ansible.builtin.debug:
            msg: >
              One or more of the configuration
              changes failed, but the web service
              is still active.

    #Configure the firewall
    - name: ensure web server ports are open
      ansible.builtin.firewalld:
        service: "{{ item }}"
        immediate: true
        permanent: true
        state: enabled
      loop:
        - http
        - https

  #Add handlers
  handlers:
    - name: restart web service
      ansible.builtin.service:
        name: "{{ web_service }}"
        state: restarted
	
vars.yml:
min_ram_mb: 256

distribution: RedHat

packages:
 - httpd
 - firewalld
 - apache
 
services: 
 - httpd
 - firewalld
 - apache
 
ssl_cert_dir: /usr/bin

web_config_files:
 - src: /home
 - dest: /etc/home 
 
web_service:
 - httpd


ansible-navigator run -m stdout playbook.yml

lab5: 
Deploying files to managed hosts

ansible_facts:
 - memtotal_mb: 960
 - processor_count: 1



vim serverb_facts.yml
- name: dispaly ansible facts
  hosts: serverb.lab.example.com
  tasks: 
   - name: Display facts
     debug: 
	   var: ansible_facts
	   
#make a template directory and create a motd.j2	   
mkdir templates

vi motd.j2
System total memory: {{ ansible_facts['memtotal_mb'] }} MiB.
System processor count: {{ ansible_facts['processor_count'] }}

vi motd.yml
---
- name: configure
  hosts: all
  remote_user: devops
  become: true
  tasks: 
   - name: Configure a custom /etc/motd
      ansible.builtin.template:
        src: templates/motd.j2
        dest: /etc/motd
        owner: root
        group: root
        mode: 0644
	
	- name: Check file exists
      ansible.builtin.stat: ##the stat module verifies the path /etc/motd exits
        path: /etc/motd
      register: motd

    - name: Display stat results
      ansible.builtin.debug: #to display the info in the register variable 
        var: motd
		
	- name: Copy custom /etc/issue file
      ansible.builtin.copy:
        src: files/issue
        dest: /etc/issue
        owner: root
        group: root
        mode: 0644
		
	- name: Ensure /etc/issue.net is a symlink to /etc/issue
      ansible.builtin.file:
        src: /etc/issue
        dest: /etc/issue.net
        state: link
        owner: root
        group: root
        force: yes
		
		
lab6: 



lab7: 

##to install collections:
#ansible-galaxy collection install -p collections/ redhat-rhel_system_roles-1.19.3.tar.gz
#ansible-galaxy collection list


NB: ansible-galaxy init [role-name]
cd ~/role-review

vim web_dev_server.yml
---
- name: Configure Dev Web Server
  hosts: dev_webserver
  force_handlers: yes
  
> mkdir -v roles
> vi roles/requirements.yml: 
- name: infra.apache
  src: git@workstation.lab.example.com:infra/apache
  scm: git
  version: v1.4
  
next: install the project dependencies on the role-review directory:
#ansible-galaxy install -r roles/requirements.yml -p roles

- install RHEL system roles:
#sudo yum install rhel-system-roles

create a role skeleton:
# cd roles
# ansible-galaxy init apache.developer_configs
# cd role-review

- update the roles/apache.developer_configs/meta/main.yml of the apache.developer_configs role

dependencies:
  - name: infra.apache
    src: git@workstation.lab.example.com:infra/apache
    scm: git
    version: v1.4

- overwrite the tasks/main.yml with the developer_tasks.yml
# mv -v developer_tasks.yml roles/apache.developer_configs/tasks/main.yml

- place the developer.conf.j2 file in the role's templates 
# mv -v developer.conf.j2 roles/apache.developer_configs/templates/

vim web_developers.yml
---
web_developers:
 - username: ffrank
   name: fred frank
   user_port: 9081
 - username: ffrank2
   name: freddie frank
   user_port: 9082
 
[role-review]$ mkdir -pv group_vars/dev_webserver 
# mv -v web_developers.yml group_vars/dev_webserver

- add the roles: apache.developer_configs to the play web_dev_server.yml
vim web_dev_server.yml 
---
- name: Configure Dev Web Server
  hosts: dev_webserver
  force_handlers: yes
  roles: 
   - apache.developer_configs  
  pre_tasks:
    - name: Check SELinux configuration
      block:
        - include_role:
            name: redhat.rhel_system_roles.selinux
      rescue:
        # Fail if failed for a different reason than selinux_reboot_required.
        - name: Check for general failure
          fail:
            msg: "SELinux role failed."
          when: not selinux_reboot_required

        - name: Restart managed host
          reboot:
            msg: "Ansible rebooting system for updates."

        - name: Reapply SELinux role to complete changes
          include_role:
            name: redhat.rhel_system_roles.selinux	  


vi selinux.yml #contains some variables 
---
# variables used by rhel-system-roles.selinux

selinux_policy: targeted
selinux_state: enforcing

selinux_ports:
  - ports:
      - "9081"
      - "9082"
    proto: 'tcp'
    setype: 'http_port_t'
    state: 'present'

[role-review] curl servera
[role-review] curl servera:9081
[role-review] curl servera:9082

LAB9: 

this installs redhat-rhel_system_roles collection from the redhat-rhel_system_roles-1.19.3.tar.gz file into the collection directory
#ansible-galaxy collection install ./redhat-rhel_system_roles-1.19.3.tar.gz -p collections

#storage_var.yml ---- a variable file containing storage
---
storage_pools:
  - name: apache-vg
    type: lvm
    disks:
      - /dev/vdb
    volumes:
      - name: content-lv
        size: 64m
        mount_point: "/var/www"
        fs_type: xfs
        state: present
      - name: logs-lv
        size: 128m
        mount_point: "/var/log/httpd"
        fs_type: xfs
        state: present
		
vi storage.yml
---
- name: Configure storage on webservers
  hosts: webservers

  roles:
    - name: redhat.rhel_system_roles.storage
nb: run the storage.yml 



#using network system-roles
---
- name: NIC Configuration
  hosts: webservers

  roles:
    - redhat.rhel_system_roles.network


crontab: 
---
- name: Recurring cron job
  hosts: webservers

  tasks:
    - name: Crontab file exists
      ansible.builtin.cron:
        name: Add date and time to a file
        minute: "*/2"
        hour: 9-16
        weekday: 1-5
        user: devops
        job: df >> /home/devops/disk_usage
        cron_file: disk_usage
        state: present
