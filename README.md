# PHP Hit Counter

Simple hit counter for tracking website traffic, using PHP (PDO) and a MySQL database to store information.

## Features

- Track hits per page
- Track total hits
- Track total unique visitors
- Track IP, user agent, and timestamp information
- View consolidated information with tables and graphs

## Usage

Add this snippet to the page you want to track:

    require_once('conn.php');
    require_once('counter.php');
	
    updateCounter("page name"); // Updates page hits
    updateInfo(); // Updates hit info

## Installation notes

Tested with PHP 5.4.16 and MySQL 5.1.72

- Make sure you have PHP Data Objects (PDO) enabled.
- Open up `conn.php` and add your database information where required.
- Run `install.php`. A message will be displayed if the installation was successful.
- Delete `install.php`.
- Add relevant code to the page you wish to track.

## For Deployment Automation through Ansible please follwo the below steps:

The first step is to install Ansible. This is easily accomplished by installing the PPA (Personal Package Archive), and installing the Ansible package with apt.
First, add the PPA using the apt-add-repository command.
- sudo apt-add-repository ppa:ansible/ansible
Once that has finished, update the apt cache.
- sudo apt-get update
- sudo apt-get install ansible
Create a new directory
- mkdir ~/ansible-php
Move into the new directory.
- cd ~/ansible-php/
- vi ansible.cfg
Add in the hostfile configuration option with the value of hosts in the [defaults] group by copying the following into the ansible.cfg file.
[defaults]
hostfile = hosts
- nano hosts
Copy the below to add in a section for php, replacing your_server_ip with your server IP address and sammy with the sudo non-root user you created in the prerequisites on your PHP Droplet.
[php]
your_server_ip ansible_ssh_user=sammy
- ansible php -m ping
Output
111.111.111.111 | success >> {
    "changed": false,
    "ping": "pong"
}
- vi php.yml
---
- hosts: php
  sudo: yes

  tasks:

  - name: install packages
    apt: name={{ item }} update_cache=yes state=latest
    with_items:
      - git
      - mcrypt
      - nginx
      - php5-cli
      - php5-curl
      - php5-fpm
      - php5-intl
      - php5-json
      - php5-mcrypt
      - php5-sqlite
      - sqlite3
      - 
      
      -ansible-playbook php.yml --ask-sudo-pass
      -vi php.yml
      ---
- hosts: php
  sudo: yes

  tasks:

  - name: install packages
    apt: name={{ item }} update_cache=yes state=latest
    with_items:
      - git
      - mcrypt
      - nginx
      - php5-cli
      - php5-curl
      - php5-fpm
      - php5-intl
      - php5-json
      - php5-mcrypt
      - php5-sqlite
      - sqlite3

  - name: ensure php5-fpm cgi.fix_pathinfo=0
    lineinfile: dest=/etc/php5/fpm/php.ini regexp='^(.*)cgi.fix_pathinfo=' line=cgi.fix_pathinfo=0
    notify:
      - restart php5-fpm
      - restart nginx

  - name: enable php5 mcrypt module
    shell: php5enmod mcrypt
    args:
      creates: /etc/php5/cli/conf.d/20-mcrypt.ini

  handlers:
    - name: restart php5-fpm
      service: name=php5-fpm state=restarted

    - name: restart nginx
      service: name=nginx state=restarted
      
      Finally, run the playbook.

- ansible-playbook php.yml --ask-sudo-pass
cloning data from git repository.
- vi php.yml
...

  - name: enable php5 mcrypt module
    shell: php5enmod mcrypt
    args:
      creates: /etc/php5/cli/conf.d/20-mcrypt.ini

  - name: create /var/www/ directory
    file: dest=/var/www/ state=directory owner=www-data group=www-data mode=0700

  - name: Clone git repository
    git: >
      dest=/var/www/laravel
      repo=https://github.com/laravel/laravel.git
      update=no
    sudo: yes
    sudo_user: www-data

  handlers:
    - name: restart php5-fpm
      service: name=php5-fpm state=restarted

    - name: restart nginx
      service: name=nginx state=restarted
      
      Save and close the playbook, then run it.

- ansible-playbook php.yml --ask-sudo-pass

configure nginx:
- vi nginx.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    root /var/www/laravel/public;
    index index.php index.html index.htm;

    server_name {{ inventory_hostname }};

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /var/www/laravel/public;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
- vi php.yml
---
- hosts: php
  sudo: yes

  tasks:

  - name: install packages
    apt: name={{ item }} update_cache=yes state=latest
    with_items:
      - git
      - mcrypt
      - nginx
      - php5-cli
      - php5-curl
      - php5-fpm
      - php5-intl
      - php5-json
      - php5-mcrypt
      - php5-sqlite
      - sqlite3

  - name: ensure php5-fpm cgi.fix_pathinfo=0
    lineinfile: dest=/etc/php5/fpm/php.ini regexp='^(.*)cgi.fix_pathinfo=' line=cgi.fix_pathinfo=0
    notify:
      - restart php5-fpm
      - restart nginx

  - name: enable php5 mcrypt module
    shell: php5enmod mcrypt
    args:
      creates: /etc/php5/cli/conf.d/20-mcrypt.ini

  - name: create /var/www/ directory
    file: dest=/var/www/ state=directory owner=www-data group=www-data mode=0700

  - name: Clone git repository
    git: >
      dest=/var/www/laravel
      repo=https://github.com/laravel/laravel.git
      update=no
    sudo: yes
    sudo_user: www-data
    register: cloned

  - name: install composer
    shell: curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    args:
      creates: /usr/local/bin/composer

  - name: composer create-project
    composer: command=create-project working_dir=/var/www/laravel optimize_autoloader=no
    sudo: yes
    sudo_user: www-data
    when: cloned|changed

  - name: set APP_DEBUG=false
    lineinfile: dest=/var/www/laravel/.env regexp='^APP_DEBUG=' line=APP_DEBUG=false

  - name: set APP_ENV=production
    lineinfile: dest=/var/www/laravel/.env regexp='^APP_ENV=' line=APP_ENV=production

  - name: Configure nginx
    template: src=nginx.conf dest=/etc/nginx/sites-available/default
    notify:
      - restart php5-fpm
      - restart nginx

  handlers:
    - name: restart php5-fpm
      service: name=php5-fpm state=restarted

    - name: restart nginx
      service: name=nginx state=restarted
     
     Save and run the playbook again:

- ansible-playbook php.yml --ask-sudo-pass


AWS horizontal scaling using CloudFormation

Here you configure an auto scaling group with a minimum size of 2 and maximum size of 4, with a defined set of metrics and at what schedule (or granularity) the metrics will be collected.

my_group: 
 Type: “AWS::AutoScaling::AutoScalingGroup”
 Properties: 
   AvailabilityZones: 
     Fn::GetAZs: “”
   LaunchConfigurationName: 
     Ref: “LaunchConfig”
   MinSize: “2”
   MaxSize: “4”
   LoadBalancerNames: 
     – Ref: “ElasticLoadBalancer”
   MetricsCollection: 
     – 
       Granularity: “1Minute”
       Metrics: 
         – “GroupMinSize”
         – “GroupMaxSize”

Here is a simple construct for defining an instance.

SimpleConfig:
 Type: AWS::AutoScaling::LaunchConfiguration
 Properties:
   ImageId: my_ami
   SecurityGroups:
   – Ref: mySecurityGroup
   – myExistingEC2SecurityGroup
   InstanceType: m1.small

And here is a construct to define the metrics and thresholds for how the scaling group will grow.
