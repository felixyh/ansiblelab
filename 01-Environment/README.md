[TOC]

# Prepare the environment

## Use terraform to provision 3 vm instances in vcenter

```buildoutcfg
# Configure the VMware vSphere Provider
provider "vsphere" {
  user           = "administrator@vsphere.local"
  
  # need to replace with the pass
  password       = "xxxx" 
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

## [Adding your SSH keys on the virtual machines](https://www.cnblogs.com/szk5043/articles/8962506.html)

### Create hosts file
```buildoutcfg
[k8s]
ansible01 ansible_host=192.168.22.67
ansible02 ansible_host=192.168.22.68
ansible03 ansible_host=192.168.22.69

[k8s:vars]
ansible_ssh_user=root
```

### [Ignore the SSH authenticity checking made by Ansible](https://stackoverflow.com/questions/32297456/how-to-ignore-ansible-ssh-authenticity-checking)

```buildoutcfg
# to avoid any human intervention in the middle of the script execution for  below prompts:
# GATHERING FACTS ***************************************************************
# The authenticity of host 'xxx.xxx.xxx.xxx (xxx.xxx.xxx.xxx)' can't be established.
# RSA key fingerprint is xx:yy:zz:....
# Are you sure you want to continue connecting (yes/no)?

felix@Felixs-MacBook-Pro 01-Environment % cat /etc/ansible/ansible.cfg | grep host_key
host_key_checking=false
```

### Upload the key to servers in batch with Ansible module:authorized_key
```
ansible -i 01-Environment/hosts k8s  -m authorized_key -a "user=root key='{{ lookup('file','~/.ssh/id_rsa.pub')}}' path='/root/.ssh/authorized_keys' manage_dir=no" --ask-pass -c paramiko
#因为密码都一样，所以只需要输入一次密码即可，如果密码不同  需要自定义
 
#说明：
user=root    #将秘钥推送到远程主机的哪个用户下
key='{{ lookup('file','/root/.ssh/id_rsa.pub')}}'    #指定要推送的秘钥文件所在的路径
path='/root/.ssh/authorized_keys'    #将秘钥推送到远程主机的哪个目录下并重命名
manage_dir=no       #指定模块是否应该管理authorized_keys文件所在的目录，如果设置为yes,模块会创建目录，以及设置一个已存在目录的拥有者和权限。如果通过 path 选项，重新指定了一个 authorized key 文件所在目录，那么应该将该选项设置为 no
exclusive [default: no]： #是否移除 authorized_keys 文件中其它非指定 key
state (Choices: present, absent) [Default: present]： #present 添加指定 key 到 authorized_keys 文件中；absent 从 authorized_keys 文件中移除指定 key


# To add key footprint into known host in localhost
ansible-playbook -i 01-Environment/hosts 01-Environment/setup.yaml
```
