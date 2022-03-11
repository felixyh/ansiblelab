[TOC]



# Infra preparation

## Servers List

| Server Name | IP address    | Role           | OS      |
| ----------- | ------------- | -------------- | ------- |
| Ansible01   | 192.168.22.67 | Web Server - 1 | CentOS8 |
| Ansible02   | 192.168.22.68 | Web Server - 2 | CentOS8 |
| Ansible03   | 192.168.22.69 | HAProxy        | CentOS8 |

## Topology

Overall topology is like below:

![image-20220311092828023](/Users/felix/Library/Application Support/typora-user-images/image-20220311092828023.png)

## Provision Infra with Terraform and vSphere

Leverage terraform to provision the servers, create below main.tf file:

```yaml
# Configure the VMware vSphere Provider
provider "vsphere" {
  user           = "administrator@vsphere.local"
  password       = "xxx"
  vsphere_server = "192.168.20.251"

  # if you have a self-signed cert
  allow_unverified_ssl = true
}

# Deploy 3 linux VMs
module "centos-server-linuxvm" {
  source    = "Terraform-VMWare-Modules/vm/vsphere"
  #source    = ".terraform/modules/centos-server-linuxvm"
  version   = "3.4.1"
  vmtemp    = "CentOS8"
  instances = 3
  vmname    = "Ansible"
  vmrp      = "Felix-Cluster/Resources/ansiblelab"
  network = {
    "VM Network" = ["192.168.22.67", "192.168.22.68", "192.168.22.69"] # To use DHCP create Empty list ["",""]; You can also use a CIDR annotation;
  }
  vmgateway = "192.168.0.250"
  dc        = "Felix-DC"
  datastore = "Data"
  ipv4submask  = ["16", "16", "16"]
  network_type = ["vmxnet3", "vmxnet3", "vmxnet3"]
  dns_server_list  = ["192.168.22.1", "192.168.22.1", "192.168.22.1"]
  is_windows_image = false
}

output "vmnames" {
  value = module.centos-server-linuxvm.VM
}

output "vmnameswip" {
  value = module.centos-server-linuxvm.ip
}
```



```shell
# terrform init
# terrform plan
# terrform apply
```



# [Step by step to set up webservers and haproxy with ansible](https://github.com/felixyh/ansiblelab)

## Lab-01

### Initialize lab environment

Create hosts file

```
[web]
ansible01 ansible_host=192.168.22.67
ansible02 ansible_host=192.168.22.68

[haproxy]
ansible03 ansible_host=192.168.22.69

[web:vars]
ansible_ssh_user=root

[haproxy:vars]
ansible_ssh_user=root
```

Ignore SSH authenticity checking, to avoid any human intervention in the middle of the script execution for below prompts:

```
# GATHERING FACTS ***************************************************************
# The authenticity of host 'xxx.xxx.xxx.xxx (xxx.xxx.xxx.xxx)' can't be established.
# RSA key fingerprint is xx:yy:zz:....
# Are you sure you want to continue connecting (yes/no)?
```

Edit the *ansible.cfg*, set `host_key_checking=false`

```
felix@Felixs-MacBook-Pro 01-Environment % cat /etc/ansible/ansible.cfg | grep host_key
host_key_checking=false
```

Upload the key to servers in batch with Ansible module: `authorized_key`

```
ansible -i 01-Environment/hosts all  -m authorized_key -a "user=root key='{{ lookup('file','~/.ssh/id_rsa.pub')}}' path='/root/.ssh/authorized_keys' manage_dir=no" --ask-pass -c paramiko
#因为密码都一样，所以只需要输入一次密码即可，如果密码不同  需要自定义
 
#说明：
user=root    #将秘钥推送到远程主机的哪个用户下
key='{{ lookup('file','/root/.ssh/id_rsa.pub')}}'    #指定要推送的秘钥文件所在的路径
path='/root/.ssh/authorized_keys'    #将秘钥推送到远程主机的哪个目录下并重命名
manage_dir=no       #指定模块是否应该管理authorized_keys文件所在的目录，如果设置为yes,模块会创建目录，以及设置一个已存在目录的拥有者和权限。如果通过 path 选项，重新指定了一个 authorized key 文件所在目录，那么应该将该选项设置为 no
exclusive [default: no]： #是否移除 authorized_keys 文件中其它非指定 key
state (Choices: present, absent) [Default: present]： #present 添加指定 key 到 authorized_keys 文件中；absent 从 authorized_keys 文件中移除指定 key

```

Lastly, add key footprint into known host in localhost and disable firewall on all servers

Create the setup.yaml file as below:

```
---
- hosts: all
  become: true
  become_user: root
  remote_user: root
  gather_facts: false

  tasks:
    - name: wait for ssh to be up
      become: false
      wait_for:
        port: 22
        delay: 5
        connect_timeout: 5
        timeout: 360
        host: "{{ ansible_host }}"
      delegate_to: localhost

    - name: add footprints
      shell: "ssh-keyscan -H {{ ansible_host }} >> ~/.ssh/known_hosts"

    - name: diable firewall
      service:
        name: firewalld
        state: stopped
        enabled: false
```



Run `setup.yaml ` 

```
# To add key footprint into known host in localhost
ansible-playbook -i 01-Environment/hosts 01-Environment/setup.yaml
```



## Lab-02

### Install necesssary packages and set global proxy

Create httpd.yaml file and create tasks for package installation

```yaml
- hosts: web
  gather_facts: false
  environment:
    http_proxy: 10.64.1.81:8080
    https_proxy: 10.64.1.81:8080
  tasks:
    - name: Installs necessary packages
      yum:
        name:
          - httpd
          - php*
          - git
        state: latest
        update_cache: true  
```

### Setup the virtual hosts

Create the virtual host configuration file: awesome-app

> [A good article for virtual hosts](https://www.cnblogs.com/swl0221/p/11752080.html) setup to understand the concept: 

```html
# Listen 8067
# <VirtualHost *:8067>
<VirtualHost *:80>
  DocumentRoot /var/www/awesome-app
  # ServerName www.awesomeapp.com
  ErrorLog /var/log/httpd/error_log
  CustomLog /var/log/httpd/access_log combined
  <Directory /var/www/awesome-app>
    Options -Indexes +FollowSymlinks
    Require all granted
    AllowOverride All
  </Directory>
</VirtualHost>
```

@httpd.yaml file, create task to copy awesome-app configuration file to webserver: /etc/httpd/conf.d/

```yaml
    - name: Push default virtual host configuration
      copy:
        src: files/awesome-app
        dest: /etc/httpd/conf.d/awesome-app.conf
        mode: 0640
```



### Deploy the web servers

@httpd.yaml file, create task to deploy awesome application

```yaml
    - name: Deploy our awesome application
      git:
        repo: https://github.com/leucos/ansible-tuto-demosite.git
        dest: /var/www/awesome-app
      tags: deploy
```



### Check configuration and roll back to default http server as needed

@httpd.yaml file, create tasks to check configuration and rollback as needed

```yaml
   - name: Check that our config is valid
      command: apachectl configtest
      register: result
      ignore_errors: true
      notify:
        - restart httpd

    - name: Rolling back - Removing the virtualhost
      file:
        path:
          - /etc/httpd/conf.d/awesome-app.conf
          - /var/www/awesome-app
        state: absent
      when: result is failed

    - name: Rolling back - Ending playbook
      fail:
        msg: "Configuration file is not valid. Please  check that before re-running the playbook"
      when: result is failed
```



### Run command to deploy

```
ansible-playbook -i hosts httpd.yaml
```

<img src="/Users/felix/Library/Application Support/typora-user-images/image-20220311102203398.png" alt="image-20220311102203398" style="zoom:50%;" />



## Lab-03

### Create haproxy template to be rendered

create `haproxy.cfg.j2` file under teamplate folder

```
global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

listen cluster
    bind {{ ansible_host }}:80
    mode http
    stats enable
    balance roundrobin
{% for backend in groups['web'] %}
    server {{ hostvars[backend]['ansible_hostname'] }} {{ hostvars[backend].ansible_host }} check port 80
{% endfor %}
    option httpchk HEAD /index.php HTTP/1.0
```

### Install haproxy for two web servers

Create `haproxy.yaml` file and create tasks to install haproxy and enable service on server boot

```yaml
- hosts: web
  gather_facts: true

- hosts: haproxy
  tasks:
    - name: Installs haproxy load balancer
      yum:
        name: haproxy
        state: present
        update_cache: yes
    - name: Enable and Allow haproxy to restart automatically at system boot-up
      service:
        name: haproxy
        enabled: true
      notify:
        - restart haproxy
```

Create tasks to push template to haproxy server

```yaml
    - name: Pushes configuration
      template:
        src: templates/haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
        mode: 0640
        owner: root
        group: root
      notify:
        - restart haproxy
```

### Run command to deploy haproxy

```shell
ansible-playbook -i hosts httpd.yaml haproxy.yaml
```

<img src="/Users/felix/Library/Application Support/typora-user-images/image-20220311102500323.png" alt="image-20220311102500323" style="zoom:50%;" />



Access http://haproxy-IP/haproxy?stats

<img src="/Users/felix/Library/Application Support/typora-user-images/image-20220311102526388.png" alt="image-20220311102526388" style="zoom:50%;" />

## Lab-04

### set variables for tempalte associated with hosts/groups

create two folders named as `group_vars` and `host_vars`

#### Group vars

The check interval will be set in a group_vars file for haproxy. This will ensure all haproxies will inherit from it.

We just need to create the file `group_vars/haproxy.yml` below the inventory directory. **The file has to be named after the group you want to define the variables for**. **If we wanted to define variables for the web group, the file would be named `group_vars/web.yml`.**

@haproxy.yaml

```yaml
---
haproxy_check_interval: 3000
```



Note that the `.yml` is optionalm: we could name haproxy group vars file `group_vars/haproxy` and Ansible would be ok with it. The extension just helps editors picking the right syntax highlighter.

```
haproxy_check_interval: 3000
haproxy_stats_socket: /tmp/sock
```

The name is arbitrary. Meaningful names are recommended of course, but there is no required syntax. You could even use complex variables (a.k.a. Python dict) like this:

```
haproxy:
    check_interval: 3000
    stats_socket: /tmp/sock
```

This is just a matter of taste. Complex vars can help group stuff logically. They can also, under some circumstances, merge subsequently defined keys (note however that this is not the default ansible behaviour). For now we'll just use simple variables.

#### Hosts vars

Hosts vars follow exactly the same rules, but live in files under `host_vars` directory.

Let's define weights for our backends in `host_vars/host1.example.com`:

```
haproxy_backend_weight: 100
```

and `host_vars/host2.example.com`:

```
haproxy_backend_weight: 150
```

If we'd define `haproxy_backend_weight` in `group_vars/web`, it would be used as a 'default': variables defined in `host_vars` files overrides variables defined in `group_vars`.

@ansible01.yaml

``` yaml
---
haproxy_backend_weight: 150
```

@ansible02.yaml

``` yaml
---
haproxy_backend_weight: 100
```

@ansible03.yaml

``` yaml
---
haproxy_backend_weight: 150
haproxy_stats_socket: /run/sock
```



### Update the template file with defined variables

``` yaml
global
    daemon
    maxconn 256
{% if haproxy_stats_socket %}
    stats socket {{ haproxy_stats_socket }} # use variables: haproxy_stats_socket defined in host var
{% endif %}

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

listen cluster
    bind {{ ansible_host }}:80
    mode http
    stats enable
    balance roundrobin
{% for backend in groups['web'] %}
    server {{ hostvars[backend]['ansible_hostname'] }} {{ hostvars[backend].ansible_host }} check port 80
{% endfor %}
    option httpchk HEAD /index.php HTTP/1.0
```



### Run command to update

ansible-playbook -i hosts haproxy.yaml

> we don't need to run httpd.yaml any more, as there's no change to web servers. However we added an empty play for web hosts at the top. It does nothing except `gather_facts: true`. But it's here because it will trigger facts gathering on hosts in group `web`. This is required because the haproxy playbook needs to pick facts from hosts in this group. If we don't do this, ansible will complain saying that `ansible_all_ipv4_addresses` key doesn't exist.



@haproxy.yaml 

``` yaml
- hosts: web
  gather_facts: true

- hosts: haproxy
  tasks:
    - name: Installs haproxy load balancer
      yum:
        name: haproxy
        state: present
        update_cache: yes

    - name: Pushes configuration
      template:
        src: templates/haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
        mode: 0640
        owner: root
        group: root
      notify:
        - restart haproxy


    - name: Enable and Allow haproxy to restart automatically at system boot-up
      service:
        name: haproxy
        enabled: true
      notify:
        - restart haproxy

  handlers:
    - name: restart haproxy
      service:
        name: haproxy
        state: restarted
```

