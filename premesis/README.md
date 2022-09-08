# IaC on Premesis with Ansible and Vagrant

### Spin up multiple virtual machines using Vagrant

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what

# MULTI SERVER/VMs environment 
#
Vagrant.configure("2") do |config|
# creating are Ansible controller
    config.vm.define "controller" do |controller|

     controller.vm.box = "bento/ubuntu-18.04"
    
     controller.vm.hostname = 'controller'
     # synced_folder to run provision.sh to set up ansible controller

     controller.vm.network :private_network, ip: "192.168.33.12"

     #config.hostsupdater.aliases = ["development.controller"] 

    end
# creating first VM called web  
    config.vm.define "web" do |web|

     web.vm.box = "bento/ubuntu-18.04"
     # downloading ubuntu 18.04 image

     web.vm.hostname = 'web'
     # assigning host name to the VM

     web.vm.network :private_network, ip: "192.168.33.10"
     #   assigning private IP

     #config.hostsupdater.aliases = ["development.web"]
     # creating a link called development.web so we can access web page with this link instread of an IP   

    end

# creating second VM called db
    config.vm.define "db" do |db|

     db.vm.box = "bento/ubuntu-18.04"

     db.vm.hostname = 'db'

     db.vm.network :private_network, ip: "192.168.33.11"

     # config.hostsupdater.aliases = ["development.db"]     
    end


end
```

## Provision the Controller

Run updates and upgrades on all machines (may take a while)
SSH into the controller and run the following commands

1. `sudo apt-get install software-properties-common`
2. `sudo apt-add-repository ppa:ansible/ansible`
3. `sudo apt-get update -y`
4. `sudo apt-get install ansible -y`
5. `sudo apt install tree`

- Now edit the ansible.cfg file to include host_key_checking = false under the `[defaults]`

## Edit the Hosts file

```
[defaults]
host_key_checking = false
```

- Now edit the hosts file to include

```
[web]
192.168.33.10 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant

[db]
192.168.33.11 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant
```

default dir /etc/ansible

### Ansible ad-hoc commands

- Copies a file from controller to host `sudo ansible web -m copy -a "src=/etc/ansible/test.txt dest=/home/vagrant"`
- See response time `sudo ansible web -m shell -a "uptime"`
- See if services are running `sudo ansible all -a "systemctl status nginx"`

You can use all to denote for `all` running machines you can also specify delcaring the host name.

## Playbooks (YAML)

### Create provisioning for DB

```
---
- name: configure the db
  hosts: aws-db
  become: true
  tasks:
  - name: update and upgrade
    become: yes
    shell: |
      sudo apt-get update
      sudo apt-get upgrade -y
    
  - name: create file
    file:
      path: /home/ubuntu/provision.sh
      state: touch

  - name: write the script
    become: yes
    blockinfile:
      marker: ""
      dest: /home/ubuntu/provision.sh
      block: |
        #!/bin/bash
        sudo apt-get update
        sudo apt-get upgrade -y
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv D68FA50FEA312927
        echo "deb https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
        sudo apt-get update
        sudo apt-get upgrade -y
        sudo apt-get install -y mongodb-org=3.2.20 mongodb-org-server=3.2.20 mongodb-org-shell=3.2.20 mongodb-org-mongos=3.2.20 mongodb-org-tools=3.2.20
        sudo systemctl restart mongod
        sudo systemctl enable mongod


  - name: make executable
    become: yes
    shell: chmod +x /home/ubuntu/provision.sh

  - name: execute shell
    become: yes
    shell: ./provision.sh

  - name: sort mongod.conf file
    become: yes
    blockinfile:
      dest: /etc/mongod.conf
      marker: ""
      block: |
        # mongod.conf

        # for documentation of all options, see:
        #   http://docs.mongodb.org/manual/reference/configuration-options/

        # Where and how to store data.
        storage:
          dbPath: /var/lib/mongodb
          journal:
            enabled: true


        systemLog:
          destination: file
          logAppend: true
          path: /var/log/mongodb/mongod.log

        net:
         port: 27017
         bindIp: 0.0.0.0

  - name: restart mongodb
    become: yes
    shell: sudo systemctl restart mongod
```

### Provision App

```
---
# add hosts or name of the host server
- hosts: aws-app

# indentation is extremely important
# gather live information
  gather_facts: yes

# we need admin permissions
  become: true

# add the instructions
  tasks:
  - name: update + upgrade
    become: yes
    shell: |
      sudo apt update
      sudo apt upgrade -y
  
  - name: Install Nginx
    apt: pkg=nginx state=present

  - name: Install NodeJS
    apt: pkg=nodejs

  - name: Install NPM
    apt: pkg=npm


  - name: download the files for the nodeapp
    become: true
    copy:
      src: /home/ubuntu/eng122_ci_cd/
      dest: /home/ubuntu/app

  
  - name: install npm in the directory
    shell: |
      cd /home/ubuntu/app/app
      npm install
      
---
- name: launch the node app
  hosts: aws-app
  become: true
  tasks:
    - name: make shell executable
      shell: chmod +x /usr/bin/nodeapp.sh

    - name: enable new service
      shell: sudo systemctl daemon-reload

    - name: enable nodeapp service
      shell: sudo systemctl enable nodeapp.service

    - name: start nodeapp service
      shell: sudo systemctl start nodeapp.service

    - name: check status
      shell : sudo systemctl status nodeapp.service
      
---
- name: do reverse proxy
  hosts: aws-app
  tasks:

  - name: Disable NGINX Default Virtual Host
    become: yes
    ansible.legacy.command:
      cmd: unlink /etc/nginx/sites-enabled/default

  - name: Create NGINX conf file
    become: yes
    file:
      path: /etc/nginx/sites-available/node_proxy.conf
      state: touch

  - name: Amend NGINX Conf file
    become: yes
    blockinfile:
      path: /etc/nginx/sites-available/node_proxy.conf
      marker: ""
      block: |
        server {
            listen 80;
            location / {
                proxy_pass http://localhost:3000;
            }
        }

  - name: Link NGINX Node Reverse Proxy
    become: yes
    command:
      cmd: ln -s /etc/nginx/sites-available/node_proxy.conf /etc/nginx/sites-enabled/node_proxy.conf

  - name: Make sure NGINX service is running
    become: yes
    service:
      name: nginx
      state: restarted
      enabled: yes
      
      
  - name: Create nodeapp service file
    become: yes
    file:
      path: /etc/systemd/system/nodeapp.service
      state: absent


  - name: Create nodeapp service file
    become: yes
    file:
      path: /etc/systemd/system/nodeapp.service
      state: touch

  - name: Amend nodeapp service file
    become: yes
    blockinfile:
      path: /etc/systemd/system/nodeapp.service
      marker: ""
      block: |
        [Unit]
        Description=Launch Node App

        [Service]
        Type=simple
        Restart=always
        user=root
        ExecStart=/usr/bin/nodeapp.sh

        [Install]
        WantedBy=multi-user.target

  - name: Create nodeapp service file
    become: yes
    file:
      path: /usr/bin/nodeapp.sh
      state: absent

  - name: Create nodeapp service file
    become: yes
    file:
      path: /usr/bin/nodeapp.sh
      state: touch

  - name: download prov script
    become: true
    copy:
      src: /home/ubuntu/nodeapp.sh
      dest: /usr/bin/nodeapp.sh
```

### The Nodeapp.sh was created outside of the script because I did not know how to configure #!/bin/bash at the top. The nodeapp.sh is below.

```
#!/bin/bash

cd /home/ubuntu/app/app

export DB_HOST=mongodb://172.31.50.248:27017/posts

node seeds/seed.js

npm start
```

### Launch app 

```
---
- name: launch the node app
  hosts: aws-app
  become: true
  tasks:
    - name: make shell executable
      shell: chmod +x /usr/bin/nodeapp.sh

    - name: enable new service
      shell: sudo systemctl daemon-reload

    - name: enable nodeapp service
      shell: sudo systemctl enable nodeapp.service

    - name: start nodeapp service
      shell: sudo systemctl start nodeapp.service

    - name: check status
      shell : sudo systemctl status nodeapp.service

```
