# Install wordpress using Ansible

Installing and setting up the WordPress website is a very difficult task for any beginner user. You will need to perform a lot of things to set up WordPress website including, installing and configuring Apache, installing PHP, installing and configuring databases.

You can automate the WordPress deployment process with the help of the Ansible. Server automation is an essential role for any system administrator. Ansible offers a simple architecture to automate the server deployment on hundreds or thousands of servers at a time.

## Create an Inventory File

Create an inventory file to define your Target host. You can create it with the following command:

```sh
vi hosts

[amazon]
172.31.47.215  ansible_user="ec2-user" ansible_port="22" ansible_private_key_file="ansible.pem"
```

Where:

- ansible_user is the root user of the Target host.
- ansible_port is the SSH port of Target host.
- ansible_private_key_file is the key file to acccess the target


### Create an Ansible Variable
Define a variable to store the information about common things including, MySQL user, Apache host, Port, Password, PHP extensions, etc. 

```sh
vim apache.vars
---
httpd_port: "8080"  
httpd_user: "apache"
httpd_group: "apache"

vim domain.vars
---
httpd_domain: "www.example.com"
    
vim mariadb.vars
---
sql_root_password: "root@12345"
sql_extra_user: "wordpress"
sql_extra_user_password: "wordpress@123"
sql_extra_database: "wordpress"
```

### Create an Apache Virtual Host Template File

Create a template for Apache virtual host configuration. Ansible will copy this template file to the Target host.

```sh
vi virtualhost.conf.tmpl

<virtualhost *:{{ httpd_port }}>
  
  servername {{ httpd_domain }}
  documentroot /var/www/html/{{ httpd_domain }}
  directoryindex index.html index.php
  <directory /var/www/html/{{ httpd_domain }}>
     allowoverride all
  </directory>
    
</virtualhost>
```

### Create a sample test page

```sh
# echo "<?php phpinfo(); ?>" > test.php
# echo "<h1><center> apache working... </center></h1>" > test.html
```

### Create a Playbook for Apache and  MariaDB
Create a playbook for the Apache to install and configure Apache and MariaDB on the Target host. This playbook will do the following things:

- Install the Apache package.
- Start the Apache service and enable it to start at boot.
- Create an Apache web root directory.
- Copy the Apache virtual host configuration template file from the Ansible control machine to the Ansible Target host.
- Install MySQL and other packages on target host
- Start the MySQL service and enable it to start at boot.
- Set the MySQL root password.
- Create a database for WordPress.
- Create a database user for WordPress.


```sh
vi lamp-Installation.yml
---
- name: "Wordpress Installation On LampStack"
  hosts: amazon
  become: true
  vars_files:
    - apache.vars
    - domain.vars
    - mariadb.vars
  tasks:
    
    - name: "Apache - Package Installation"
      yum:
        name: httpd
        state: present
    
    - name: "Apache - Enabling PhP Support"
      shell: amazon-linux-extras install php7.4 -y
        
    
    - name: "Apache - Generating httpd.conf from template"
      template:
        src: httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
    
    - name: "Apache - Generating Vitualhost from template"
      template:
        src: virtualhost.conf.tmpl
        dest:  "/etc/httpd/conf.d/{{ httpd_domain }}.conf"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
    
    - name: "Apache - Creating DocumentRoot for virtualhost"
      file:
        path: "/var/www/html/{{ httpd_domain }}"
        state: directory
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}" 
            
            
    - name: "Apache - Creating Sample test pages"
      copy:
        src: "{{ item }}"
        dest: "/var/www/html/{{ httpd_domain }}/"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
      with_items:
        - test.php
        - test.html
            
    - name: "Apache - Restarting/Enabling httpd Service"
      service:
        name: httpd
        state: restarted
        enabled: true
            
    - name: "Mariadb-Server - Package Installation"
      yum:
        name: 
          - mariadb-server
          - MySQL-python
        state: present
    
    - name: "Mariadb-Server - Service Restart/Enable"
      service:
        name: mariadb
        state: restarted
        enabled: true
            
    - name: "Mariadb-Server - Reset Root Password"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        user: "root"
        password: "{{ sql_root_password }}"
        host_all: true
            
    
    - name: "Mariadb-Server - Removing Anonymous Users"
      mysql_user:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        user: ""
        state: absent
            
    - name: "Mariad-Server - Removing test database"
      mysql_db:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        name: "test"
        state: absent
    
    - name: "Mariad-Server - Creating extra database {{ sql_extra_database }}"
      mysql_db:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        name: "{{ sql_extra_database }}"
        state: present
            
            
    - name: "Mariad-Server - Creating Extra user {{ sql_extra_user }}"
      mysql_user:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        user: "{{ sql_extra_user }}"
        password: "{{ sql_extra_user_password }}"
        state: present
        priv: '{{ sql_extra_database }}.*:ALL'
```        

### Create a Playbook for WordPress

Create a playbook for WordPress role to download and configure WordPress on the Target host. This playbook will do the following things:

- Download and extract WordPress to the Apache web root directory.
- Set ownership for WordPress.
- Rename WordPress sample configuration file.
- Define database settings in the WordPress configuration file.


```sh

vim wordpress-Installation.yml

---
- name: "Wordpress Installation On LampStack"
  hosts: amazon
  become: true
  vars_files:
    - apache.vars
    - domain.vars
    - mariadb.vars
  vars:
    wordpress_url: "https://wordpress.org/wordpress-5.7.3.tar.gz"
  tasks:
    
    - name: "Apache - Package Installation"
      yum:
        name: httpd
        state: present
    
    - name: "Apache - Enabling PhP Support"
      shell: amazon-linux-extras install php7.4 -y
        
    
    - name: "Apache - Generating httpd.conf from template"
      template:
        src: httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
    
    - name: "Apache - Generating Vitualhost from template"
      template:
        src: virtualhost.conf.tmpl
        dest:  "/etc/httpd/conf.d/{{ httpd_domain }}.conf"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
    
    - name: "Apache - Creating DocumentRoot for virtualhost"
      file:
        path: "/var/www/html/{{ httpd_domain }}"
        state: directory
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}" 
            
            
    - name: "Apache - Creating Sample test pages"
      copy:
        src: "{{ item }}"
        dest: "/var/www/html/{{ httpd_domain }}/"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
      with_items:
        - test.php
        - test.html
            
    - name: "Apache - Restarting/Enabling httpd Service"
      service:
        name: httpd
        state: restarted
        enabled: true
            
    - name: "Mariadb-Server - Package Installation"
      yum:
        name: 
          - mariadb-server
          - MySQL-python
        state: present
    
    - name: "Mariadb-Server - Service Restart/Enable"
      service:
        name: mariadb
        state: restarted
        enabled: true
            
    - name: "Mariadb-Server - Reset Root Password"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        user: "root"
        password: "{{ sql_root_password }}"
        host_all: true
            
    
    - name: "Mariadb-Server - Removing Anonymous Users"
      mysql_user:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        user: ""
        state: absent
            
    - name: "Mariad-Server - Removing test database"
      mysql_db:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        name: "test"
        state: absent
    
    - name: "Mariad-Server - Creating extra database {{ sql_extra_database }}"
      mysql_db:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        name: "{{ sql_extra_database }}"
        state: present
            
            
    - name: "Mariad-Server - Creating Extra user {{ sql_extra_user }}"
      mysql_user:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        user: "{{ sql_extra_user }}"
        password: "{{ sql_extra_user_password }}"
        state: present
        priv: '{{ sql_extra_database }}.*:ALL'
            
            
    - name: "Wordpress - Downloading Archive File {{ wordpress_url }}"
      get_url:
        url: "{{ wordpress_url }}"
        dest:  "/tmp/wordpress.tar.gz"
     
    
    - name: "Wordpress - Extracting Archive File /tmp/wordpress.tar.gz"
      unarchive:
        src: "/tmp/wordpress.tar.gz"
        dest: "/tmp/"
        remote_src: true
            
    
    - name: "Wordpress - Copying Contents From /tmp/wordpress/ to /var/www/html/{{ httpd_domain }}/"
      copy:
        src: "/tmp/wordpress/"
        dest: "/var/www/html/{{ httpd_domain }}/"
        remote_src: true
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
            
    - name: "Wordpress - Generating wp-config.php from template"
      template:
        src: wp-config.php.tmpl
        dest: "/var/www/html/{{ httpd_domain }}/wp-config.php"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
            
            
    - name: "Post-Installation - Clean Up"
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - "/tmp/wordpress"
        - "/tmp/wordpress.tar.gz"
            
    - name: "Post-Installation - Restart"
      service:
        name: "{{ item }}"
        state: restarted
        enabled: true
      with_items:
        - httpd
        - mariadb

```
### Run the Ansible Playbook


```sh
ansible-playbook -i hosts wordpress-Installation.yml
```

Once the playbook has been executed successfully, you should get the following output:

```sh
PLAY [lamp install] **********************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************
[WARNING]: Platform linux on host 172.31.47.215 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python
interpreter could change this. See https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [172.31.47.215]

TASK [apache install] ********************************************************************************************************************************************
ok: [172.31.47.215]

TASK [enable php] ************************************************************************************************************************************************
changed: [172.31.47.215]

TASK [httpd.conf generate] ***************************************************************************************************************************************
ok: [172.31.47.215]

TASK [DocumentRoot] **********************************************************************************************************************************************
ok: [172.31.47.215]

TASK [sample tset pages] *****************************************************************************************************************************************
ok: [172.31.47.215] => (item=test.php)
ok: [172.31.47.215] => (item=test.html)

TASK [apache restart] ********************************************************************************************************************************************
changed: [172.31.47.215]

TASK [mariadb install] *******************************************************************************************************************************************
ok: [172.31.47.215]

TASK [mariadb service] *******************************************************************************************************************************************
changed: [172.31.47.215]

TASK [mariadb server reset root password] ************************************************************************************************************************
[WARNING]: Module did not set no_log for update_password
fatal: [172.31.47.215]: FAILED! => {"changed": false, "msg": "unable to connect to database, check login_user and login_password are correct or /root/.my.cnf has the credentials. Exception message: (1045, \"Access denied for user 'root'@'localhost' (using password: NO)\")"}
...ignoring

TASK [Removing Anonymous Users] **********************************************************************************************************************************
ok: [172.31.47.215]

TASK [Removing test database] ************************************************************************************************************************************
ok: [172.31.47.215]

TASK [Creating extra database] ***********************************************************************************************************************************
ok: [172.31.47.215]

TASK [Creating Extra user] ***************************************************************************************************************************************
ok: [172.31.47.215]

TASK [download wordpress] ****************************************************************************************************************************************
changed: [172.31.47.215]

TASK [extract downloaded wordpress] ******************************************************************************************************************************
changed: [172.31.47.215]

TASK [Copying Contents] ******************************************************************************************************************************************
ok: [172.31.47.215]

TASK [Wordpress - Generating wp-config.php from template] ********************************************************************************************************
ok: [172.31.47.215]

TASK [post -installation clean up] *******************************************************************************************************************************
changed: [172.31.47.215] => (item=/tmp/wordpress)
changed: [172.31.47.215] => (item=/tmp/wordpress.tar.gz)

TASK [post installtion -restart] *********************************************************************************************************************************
changed: [172.31.47.215] => (item=httpd)
changed: [172.31.47.215] => (item=mariadb)

PLAY RECAP *******************************************************************************************************************************************************
172.31.47.215              : ok=20   changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=1

```
