# What Ansible is good for?
## `Code driven deployments and operations and complicated set of actions`
  - configuration automation by hand was not a good idea
  - infra provisioning take delays as well
  - on-demand infra in cloud computing
  - ansible use for orchestration performance
    - complicated actions
    - with specific order
  - create new hosts file
  - vim webapp
  ```
  [web]
  host1
  host2
  host3
  
  [lb]
  lb1
  lb2
  
  [all:vars]
  ansible_connection=local
  # to connect locally to all these fake hosts
  ```
  - vim webdeply.yaml
  ```
  ---
  - name: Deploy
    hosts: web
    serial: 1
    # so that each host is executed through all the tasks , before moving to next hosts
    
    tasks:
      # disable the node
      - name: disable node
        debug:
          msg: "disbale {{ inventory_hostname }}"
       # delegate use for redirect connection, below addressing the first node of the lb group
        delegate_to: "{{ groups['lb'][0] }}"
        
       # upgrade the software
       - name: upgrade web
         debug:
           msg: "upgrade software"
           
       # reenable the node
       - name: enable node
         debug:
           msg: "enable {{ inventory_hostname }}"
         delegate_to: "{{ groups['lb'][0] }}"
  ```
  - `ansible-playbook -i webapp webdeply.yaml`
  - so the task are made with only one lb and if serial is not there then it would had skipped, `[host1 -> lb1]`
## Manage system configuration
  - configure the appln and make the service started
  - vim web-app.yaml
  ```
  ---
  - name: configure web app
    hosts: web
    # git source code repo and the ver to checkout
    vars:
      repo: myrepo.com/repo.git
      version: 8
     
    tasks:
      - name: install nginx
        debug:
          msg: "dnf install nginx"
          
      # create a dir to hold the web content
      - name: ensure web dir
        debug: 
          # we can use of file module if the dir is already there
          msg: "/mkdir /webapp"
          
      - name: get content
        debug:
          msg: "git clone --branch {{ version }} {{ repo }} /webapp"
      
      - name: nginx config
        debug:
          msg: "put nginx config  in place"
      
      # check the service is running, use service module
      - name: ensure nginx running
        debug:
          msg: "service nginx start"     
  ```
  - `ansible-playbook -i webapp web-app.yaml`
## React to configuration changes
  - what if we want to change the config of nginx above and then we need to restart it
  - we can use `notify` directive in any task, which instruct to notify handler on change and after notify add `changed_when` to true to reflect the change
  - and we need to define handler section in the play
  - handlers are just special task, with special meaning and only ran when required
  ```
  ---
  - name: configure web app
    hosts: web
    # git source code repo and the ver to checkout
    vars:
      repo: myrepo.com/repo.git
      version: 8
     
    tasks:
      - name: install nginx
        debug:
          msg: "dnf install nginx"
          
      # create a dir to hold the web content
      - name: ensure web dir
        debug: 
          # we can use of file module if the dir is already there
          msg: "/mkdir /webapp"
          
      - name: get content
        debug:
          msg: "git clone --branch {{ version }} {{ repo }} /webapp"
      
      - name: nginx config
        debug:
          msg: "put nginx config  in place"
        notify: restart nginx
        changed_when: True
      
      # check the service is running, use service module
      - name: ensure nginx running
        debug:
          msg: "service nginx start" 
          
    handlers:
      - name: restart nginx
        debug:
          msg: "service nginx restart"
  ```
  - `ansible-playbook -i webapp web-app.yaml -vv`
## Infra Management
  - create new docker containers and play around with those containers
  - vim docker.yaml
  ```
  ---
  - name: make dockers
    hosts: localhost
    
    # create 3 containers using busybox image
    tasks:
      - name: make containers
        docker_container:
          image: busybox
          # have the containers sleep for a day
          command: sleep 1d
          # name the container with_seq loop
          name: "busy{{ item }}"
        with_sequence: count=3
      
      - name: add hosts
        add_host:
          # same name as the container above
          name: "busy{{ item }}"
          # adding to group
          groups: dockers
          # setting connection method
          ansible_connection: docker
        with_sequence: count=3
        
  # adding the second play for inside the containers
  - name: hi from docker
    hosts: dockers
    # they do not have the python built ins, so we need to tell the ansible not to gather facts
    gather_facts: False
    
    tasks:
      - name: ping
      # use of raw module, which doesnot require python on hosts
        raw: exho $HOSTNAME
  ```
  - `ansible-playbook -i webapp docker.yaml -vv`
## Repeat a task across a fleet / Ad Hoc Tasks
  - eg 
    - `ansible -i webapp web -m debug -a "msg='shutdown -r now'"`
    - `ansible -i webapp web -m command -a "rpm -q ansible"`
    - `ansible-doc copy` - to check the documentation of any module
    - `ansible -i webapp web -m dnf -a "name=libselinux-python state=present" -f 1 -b`
    - -f respond to `forks` which is the max number of concurrent hosts and -b is used for `priviledge excalation`
    - `ansible -i webapp web -m copy -a "src=file1 dest=/tmp/file1" -f 1`
## challenge
  - ad-hoc task to check the kernel release of local sys
    - `ansible -i hosts host1 -m command -a "uname -r"`
