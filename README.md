# AnisbleでDocker Server 構築 (CentOS7)
[![MIT License](http://img.shields.io/badge/license-MIT-blue.svg?style=flat)](LICENSE) ![GitHub release (latest by date including pre-releases)](https://img.shields.io/github/v/release/webdimension/ansible-softether-for-conoha?include_prereleases)
## 概要
- VPSなどのサーバー立ち上げ
- Ansibleで初期設定 (Root)
  - Rootでログイン 
  - 一般ユーザー作成 ( ssh鍵登録 ）
  - sshd Root login を無効
  - パスワード認証無効  (鍵認証のみ)
  - ssh port変更
  - ssh 許可IP設定
  - Firewalld 設定 
  - hostname 設定 
- AnsibleでDocker,Docker-compose,etc(tools)... をInstall
  - 指定したUserへ Group:docker 追加 
  - docker auto start
  - docker 許可IP指定 (port:2376)

## 環境
***Local***
- Ubuntu 18.04 On Vagrant + VMware fusion
- Ansible 2.9.7 

***Server***
- CentOS 7.8 

## 設定
### hosts/product
```yml
[docker-server]
# Server IPaddress
Product-Docker-Init ansible_host=xxx.xxx.xxx.xxx
Product-Docker

[Product-Docker-Init]
Product-Docker-Init
[Product-Docker-Init:vars]
ansible_ssh_user=root
ansible_port=22
ansible_ssh_pass={{ secret.root_user_password }}

[product]
Product-Docker-Init
Product-Docker
```
### group_vars/all/secret
```yml
secret:
  ansible_users:
    - name: 'ansible'
      groups: 'wheel'
      append: 'yes'
      state: 'present'
	  remove: 'no'
	  #Ansible user password
      password: "ansible_user_password"
      public_key: "~/.ssh/ansible_rsa.pub"
      login_shell: '/bin/bash'
      create_home: 'yes'
      sudo: 'present'
      comment: 'ansible user'
      expires: '-1'
  # groups
  user_groups:
    wheel:
      name: 'wheel'
      state: 'present'
    manager:
      name: 'manager'
      state: 'present'
```
### group_vars/all/sshs.yml
```yml
# ssh port
sshd_port: 50022 #Default 22
```

### host_vars/*/hostname.yml
```yml
# hostname
hostname: 'srv.docker'
```

### group_vars/*/secret
```yml
secret:
  # Server Root Password
  root_user_password: 'Server_Root_Password'

  firewalld_service:
    - service: 'ssh'
      zone: 'public'
      permanent: 'yes'
      state: 'enabled'
      immediate: true

  firewalld_rich_rules:
    - port: "{{ sshd_port }}"
      protocol: tcp
      control: accept
      ips:  # ssh Allow IPs
        - 111.222.333.444
        - 111.222.333.445
    - port: "{{ dockerd_port }}"
      protocol: tcp
      control: accept
      ips:  # Docker allow IPs
        - 111.222.333.444
        - 111.222.333.445
        - 111.222.333.446
  # users
  # If empty
  # users: []
  users:
    develop_001: # User prefix
      name: 'develop_001'  # User name
      groups: 'wheel,manager'
      append: 'yes'
      state: 'present'
      remove: 'no'
      password: "password"  # User password
      public_key: "~/.ssh/ansible_rsa.pub" # 公開鍵
      login_shell: '/bin/bash'
      create_home: 'yes'
      sudo: 'absent'
      comment: 'comment'
      expires: '-1'
      docker_group: true  # Docker user
    develop_002: # User prefix
      name: 'develop_002'  # User name
      groups: 'wheel,manager'
      append: 'yes'
      state: 'present'
      remove: 'no'
      password: "password"  # User password
      public_key: "~/.ssh/ansible_rsa.pub" # 公開鍵
      login_shell: '/bin/bash'
      create_home: 'yes'
      sudo: 'present'
      comment: 'comment'
      expires: '-1'
      docker_group: false # Docke user
```
## Example command
### For vagrant
#### Install all
Install all without init
```bash
ansible-playbook -i hosts/vagrant site.yml -l Vagrant-Docker --skip-tags init
```
#### Install Docker
Install Docker and Docker-compose
```bash
ansible-playbook -i hosts/vagrant site.yml -l Vagrant-Docker -t docker
```
#### Install tools
Install tools. glances, nvim, lazydocker, etc...
```bash
ansible-playbook -i hosts/vagrant site.yml -l Vagrant-Docker -t tools
```


### For Product
#### Add Ansible user and sshd setting, firewalld 
Add ansible user From root login and sshd, firewalld
```bash
ansible-playbook -i hosts/product site.yml -l Product-Docker-Init -t init -t firewalld
```
#### Install all
Install all without init
```bash
ansible-playbook -i hosts/product site.yml -l Product-Docker --skip-tags init
```
#### Install Docker
Install Docker and Docker-compose
```bash
ansible-playbook -i hosts/product site.yml -l Product-Docker -t docker
```
#### Install tools
Install tools. glances,lazydocker, etc...
```bash
ansible-playbook -i hosts/product site.yml -l Product-Docker -t tools
```

## 機密情報の暗号化
### vault pass
```bash
echo 'vault_password_file = ./vaultpass' > ansible.cfg
cp valtpass.sample valtpass
echo 'your_vault_password' > vaultpass
```
#### 暗号化
```bash
ansible-vault encrypt group_vars/*/secret.yml 
```
#### 復号化
```bash
ansible-vault decrypt group_vars/*/secret.yml 
```

## License
MIT