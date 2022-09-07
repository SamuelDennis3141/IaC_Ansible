# Ansible on AWS

## Creating Controller Node on AWS

This can be created by creating an EC2 instances and running the appropriate provisions inside the instance.

## Provisioning the Control Node

```
sudo apt update
sudo apt upgrade -y
sudo apt install python3-pip
pip3 install awscli
pip3 install boto boto3
```

## Creating a Key Pair (name your key appropriately)

```
cd ~/.ssh/
ssh-keygen -t rsa -b 4096
```

## Creating folder for encrypted AWS secret and access keys
``` 
cd /etc/ansible/
mkdir group_vars
cd group_vars/
mkdir all/
cd all
sudo ansible-vault create pass.yml
sudo chmod 600 pass.yml
```

#### When inside the pass.yml file enter

```
access_key_id: xxxx
secret_key_id: xxxx
```

NOTE to exit .vi files enter `esc wq! enter`

## Create EC2 Instances (playbook)

# AWS playbook
---

- hosts: localhost
  connection: local
  gather_facts: False

  vars:
    ansible_python_interpreter: /usr/bin/python3
    key_name: eng122_samuel
    region: eu-west-1
    image: ami-07b63aa1cfd3bc3a5
    id: "db-ansible"
    sec_group: "{{ id }}-sec"


  tasks:

    - name: Facts
      block:

      - name: Get instances facts
        ec2_instance_facts:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          region: "{{ region }}"
        register: result

      - name: Instances ID
        debug:
          msg: "ID: {{ item.instance_id }} - State: {{ item.state.name }} - Public DNS: {{ item.public_dns_name }}"
        loop: "{{ result.instances }}"

      tags: always


    - name: Provisioning EC2 instances
      block:

      - name: Upload public key to AWS
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '/home/ubuntu/.ssh/{{ key_name }}.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"

      - name: Create security group
        ec2_group:
          name: "{{ sec_group }}"
          description: "Sec group for app {{ id }}"
          # vpc_id: 12345
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          rules:
            - proto: tcp
              ports:
                - 22
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on ssh port
        register: result_sec_group

      - name: Provision instance(s)
        ec2:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          key_name: "{{ key_name }}"
          id: "{{ id }}"
          group_id: "{{ result_sec_group.group_id }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: true
          count: 1
          # exact_count: 2
          count_tag:
            Name: eng122_sam_db_ansible
          instance_tags:
             Name: eng122_sam_db_node

      tags: ['never', 'create_ec2']

## Alter Hosts Folder

```
[aws-app]
ec2-app-instance ansible_host=ec2-ip_public ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/file  

[aws-db]
ec2-db-instance ansible_host=ec2-ip_public ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/file  
```

> Use file created in key generation step (the one used in creating the ec2 instance)
> Run `sudo ansible aws-app -m ping` to ensure connection
> Change ansible.cnfg to include hosts_key_checking=false if issues remain. (NOT TO BE DONE IN PRODUCTION).


## Playbooks used to deploy app with database 

#### NOTE PLAYBOOKS PLACED IN THE /etc/ansible LOCATION FOR EASE OF USE

### Provisions for the database node

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

### Provision the App node

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

### Launch the app
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

### Contents of nodeapp.sh

```
#!/bin/bash

cd /home/ubuntu/app/app

export DB_HOST=mongodb://172.31.50.248:27017/posts

node seeds/seed.js

npm start
```
