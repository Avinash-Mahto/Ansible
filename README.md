---
- hosts: openstack
  sudo: yes

  tasks:
  - name: "Install NTP service"
    yum: name=chrony state=present

  - name: "copy /etc/chrony.conf file"
    copy:
     src: "/root/openstack/chrony.conf"
     dest: "/etc/chrony.conf"

  - name: "Start NTP service"
    service: name=chronyd state=started    

  - name: " Install Openstack Package"
    yum: name=centos-release-openstack-mitaka state=present
    
  - name: "Upgrade all packages"
    yum: name=* state=latest update_cache=yes
  - name: "Install the openstack client"
    yum: name=python-openstackclient state=present
  - name: "Install the openstack-selinux package"
    yum: name=openstack-selinux state=present

  - name: "Install mariadb"
    yum: name={{ item }} state=present
    with_items:
      - mariadb
      - mariadb-server
      - python2-PyMySQL
    tags: mariadb
 
  - name: "create and edit the /etc/my.cnf.d/openstack.cnf"
    copy:
     src: /root/openstack/openstack.cnf 
     dest: /etc/my.cnf.d/openstack.cnf

  - name: "Start the database service and configure it to start"
    service: "name=mariadb state=started enabled=yes"
    tags: mariadb1

  - name: " Start installation of expect"
    yum: name=expect state=present

  - name: "copy mysql script to destination server"
    copy:
     src: /root/openstack/mariadb
     dest: /root/
     mode: 0755 
     
  - name: "running the mysql_secure_installation script"
    command: >
     expect /root/mariadb

  - name: " Install mongo db"
    yum: name={{ item }} state=present
    with_items:
      - mongodb-server
      - mongodb
  - debug: var=item.stdout
    with_items: echo.results


  - name:  "/etc/mongod.conf file" 
    lineinfile:
      dest: "/etc/mongod.conf"
      regexp: "{{ item.regexp }}"
      line: "{{ item.line }}"
      state:  "{{ present }}"
    with_items:
      - { regexp: "^bind_ip = 127.0.0.1", line: "bind_ip = 10.0.0.17" }
      - { regexp: "^#smallfiles = true", line: "smallfiles = true" }

  - name: "start mongodb service"
    service: name=mongod state=started enabled=yes

  - name: "Install rabbitmq package"
    yum: name=rabbitmq-server state=present

  - name: " start rabbitmq services"
    service: name=rabbitmq-server state=started enabled=yes

 - name: "create openstack user"
    command: >
     rabbitmqctl add_user openstack redhat
  - name: "Permit configuration, write, and read access for the openstack user"
    command:
     rabbitmqctl set_permissions openstack ".*" ".*" ".*"

  - name: "Install memcached packge"
    yum: name={{ item }} state=present
    with_items:
      - memcached
      - python-memcached

  - name: "Start memcache service"
    service: name=memcached state=started

  - name: root password is present
    mysql_user: 
      user: root 
      login_host: localhost 
      login_password: openstack 
      login_port: 3306
      state: present


  - mysql_user: 
      user: keystone
      login_password: openstack
      priv: '*.*:ALL'
      state: present

  - name: "Create the Keystone database"
    mysql_db: 
      login_user: root
      login_password: openstack
      db: keystone 
      state: present

  - shell: 
      openssl rand -hex 10 >> /root/pass.txt

  - name: "Install keystone, httpd & mod_wsgi package"
    yum: name={{ item }} state=present
    with_items:
      - openstack-keystone
      - httpd
      - mod_wsgi

  - name:  Edit "/etc/keystone/keystone.conf"
    lineinfile:
      dest: "/etc/keystone/keystone.conf"
      regexp: "{{ item.regexp }}"
      line: "{{ item.line }}"
    with_items:
      - { regexp: "^#admin_token = d5c53221cbaea647d5a0u", line: "admin_token = d5c53221cbaea647d5a0" }
      - { regexp: "^#connection = mysql://keystone:openstack@openstack/keystone", line: "connection = mysql://keystone:openstack@10.0.0.17/keystone" }
      - { regexp: "^#provider = fernet", line: "provider = fernet" }


  - name: "copy database populate script"
    copy:
     src: /root/openstack/a.sh
     dest: /root/
     mode: 0755

  - name: "Populate database service"
    command: >
    ./root/a.sh

  - name: "Initialize Fernet keys"
    command: >
      keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
  - name:  Edit "/etc/httpd/conf/httpd.conf"
    lineinfile:
      dest: "/etc/httpd/conf/httpd.conf"
      regexp: "^#ServerName www.example.com:80"
      line: "ServerName openstack"

  - name: "Create /etc/httpd/conf.d/wsgi-keystone.conf file"
    copy:
     src: /root/openstack/wsgi-keystone.conf
     dest: /etc/httpd/conf.d/

  - name: " Start Apache HTTP service"
    service: name=httpd state=restarted

  - name: "Set source file to create keystone endpoints"
    script:  /etc/ansible/source.sh

  - name: "Create keystone endpoints"
    shell:  >
      `openstack service create --name keystone --description "OpenStack Identity" identity`
