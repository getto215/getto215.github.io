---
title: "Ansible을 이용한 서버 초기 세팅 (Ubuntu)"
tags:
  - ansible
  - devops
---
서버를 신규로 할당 받으면 ntp, ssh, 호스트 파일 배포, 계정 생성 등 다양한 초기 세팅이 필요합니다. 
기존에 서버 초기 세팅을 bash로 작성된 스크립트를 이용했었는데, 유지보수도 쉽지 않고, OS 버전과 멱등 처리 불편한 점이 많아 이제야(?) ansible을 활용해보기로 하였습니다. <u>매우 단순하게 구성되어 있으므로, 테스트용으로만 참고하시기 바랍니다.</u>

## 구성 환경
* ssh id, password로 접근이 가능한 Ubuntu(14.04) 서버를 초기화
* ansible version : 2.5.5 (Mac OS)

## root 디렉토리 구성
ansible을 설치하면 기본으로 구성되는 폴더가 아닌 다른 곳에 루트 디렉토리를 구성하였습니다.

~~~bash
$ tree
.
├── ansible.cfg
├── group_vars
│   └── all.yml
├── init.yml
├── inventory
└── roles
~~~

#### ansible.cfg (~/ansible.cfg)

~~~ini
[defaults]
host_key_checking=False
~~~

#### inventory (~/inventory)
인벤토리 파일에는 호스트 그룹, ssh 등의 정보를 담고 있습니다.

~~~ini
[all]
test-01

[all:vars]
ansible_connection=ssh
ansible_ssh_user=ubuntu
ansible_ssh_pass=ubuntu
ansible_sudo_pass=ubuntu
ansible_become=true
~~~

#### Test
inventory 파일이 생성되었으니, 간단히 테스트를 할 수 있습니다.

~~~bash
$ ansible -i inventory all -m ping
test-01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
~~~

#### init.yml (~/init.yml)
초기화하는 플레이북입니다. 

~~~yml
---
- name: initializing hosts
  hosts: all
  roles:
    - init
~~~

## init Role 구성 (~/roles/init)
roles 폴더 안에 init이란 role을 만들어서 초기화하는 플레이북을 만듭니다. 
`ansible-galaxy init [role name]`를 이용하면 쉽게 레이아웃을 구성할 수 있습니다.

#### roles/init 레이아웃

~~~bash
$ cd roles
$ tree
.
└── init
    ├── files
    │   ├── hosts
    │   ├── java
    │   │   ├── jdk1.6.0_41.tar.gz
    │   │   ├── jdk1.7.0_71.tar.gz
    │   │   └── jdk1.8.0_171.tar.gz
    │   └── ntp.conf
    ├── handlers
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    └── vars
        └── main.yml
~~~

#### files 디렉토리
* hosts : 호스트 파일을 jinja를 이용하여 다이나믹하게 구성할 수도 있지만, 정적으로 배포할 수 있도록 구성하였습니다. 해당 파일은 /etc/hosts 파일을 대체.
* java : 다양한 java 버전을 사용하고 있어 패키지로 설치하지 않고 별도로 바이너리를 /usr/lib/jvm 내에 구성합니다.
* ntp.conf : 별도의 ntp가 필요할 경우, /etc/ntp.conf를 대체.

#### handlers (~/roles/init/handlers/main.yml)

~~~bash
---
# handlers file for init
- name: restart ntpd
  service:
    name: ntp
    state: restarted

- name: update apt
  apt:
    update_cache: yes
~~~

#### tasks (~/roles/init/tasks/main.yml)
작업 모듈별로 use 플래그 변수를 두고 사용 여부를 설정할 수 있습니다.

~~~bash
---
# tasks file for init
# 사용자 그룹을 추가
- name: Add user group
  group:
    name: "{{ item.username }}"
    gid: "{{ item.id }}"
    state: present
  with_items: "{{ users }}"
  when: use_user | default(false) == true

- name: Add user
  user:
    name: "{{ item.username }}"
    shell: /bin/bash
    password: "{{ item.password }}"
    group: "{{ item.username }}"
    uid: "{{ item.id }}"
    append: yes
  with_items: "{{ users }}"
  when: use_user | default(false) == true

# 패스워드 없이 ssh를 접근하기 위해 .ssh를 복제합니다. 테스트용이며, 실제 프러덕트 환경에서는 ssh-keyzen으로 좀 더 디테일하게 구성해야 합니다.
- name: Copy a .ssh
  become: true
  become_user: "{{ item.username }}"
  copy:
    src: .ssh
    dest: /home/{{ item.username }}/
    owner: "{{ item.username }}"
    group: "{{ item.username }}"
    mode: preserve
  with_items: "{{ users }}"
  when: use_user | default(false) == true

# 생성되는 계정에 기본 그룹이 필요할 경우
- name: Add defualt group
  group:
    name: "{{ default_group_name }}"
    gid: "{{ default_group_id }}"
    state: present
  when: use_default_group | default(false) == true

- name: Add user in default group
  user:
    name: "{{ item.username }}"
    groups: "{{ item.username }},{{ default_group_name }}"
    append: yes
  with_items: "{{ users }}"
  when: 
    - use_default_group | default(false) == true
    - use_user | default(false) == true

# Sudoers
- name: Add admin user in sudoers
  lineinfile:
    dest: /etc/sudoers
    state: present
    line: '{{ item }} ALL=(ALL) NOPASSWD:ALL' 
    validate: 'visudo -cf %s'  
  with_items: "{{ sudoers }}"
  when: use_sudoers | default(false) == true

# ------------------------------
# ntp
- name: Install ntpd
  apt:
    name: ntp
  when: use_ntp | default(false) == true

- name: Copy a ntp.conf
  copy:
    src: ntp.conf
    dest: /etc/
    backup: yes
    owner: root
    group: root
    mode: 0644
  when: use_ntp | default(false) == true
  notify: [ 'restart ntpd' ]


# ------------------------------
# sysctl
# /etc/sysctl.conf
- name: Modify vm swappiness value 
  sysctl:
    sysctl_file: /etc/sysctl.conf
    name: "{{ item.name }}"
    value: "{{ item.value }}"
  with_items: "{{ vm_value }}"
  when: use_vm_swappiness | default(false) == true

# file descriptor (/etc/security/limits.conf)
- pam_limits:
    domain: '*'
    limit_type: '-'
    limit_item: nofile
    value: "{{ max_file_descriptor }}"
  when: use_file_descriptor | default(false) == true

- pam_limits:
    domain: '*'
    limit_type: '-'
    limit_item: core
    value: unlimited
  when: use_file_descriptor | default(false) == true

- pam_limits:
    domain: root
    limit_type: '-'
    limit_item: nofile
    value: "{{ max_file_descriptor }}"
  when: use_file_descriptor | default(false) == true

- pam_limits:
    domain: root
    limit_type: '-'
    limit_item: core
    value: unlimited
  when: use_file_descriptor | default(false) == true


# ------------------------------
# apt update, install
- name: Update apt
  apt:
    update_cache: yes
  when: use_apt_update | default(false) == true

- name: Install packages (apt-get)
  apt:
    name: "{{ item }}"
  with_items: "{{ apt_install }}"
  when: use_apt_install | default(false) == true
  

# ------------------------------
# java
- name: Create java home dir {{ java_path }}
  file:
    path: "{{ java_path }}"
    owner: root
    group: root
    state: directory
  when: use_java | default(false) == true

- name: Copy and Extract jdk files into {{ java_path }}
  unarchive:
    src: "java/{{ item.jdk }}.tar.gz"
    dest: "{{ java_path }}"
    owner: root
    group: root
  with_items: "{{ java_files }}"
  when: use_java | default(false) == true

- name: Java symbolic link
  file:
    src: "{{ java_path }}/{{ item.jdk }}"
    dest: "{{ java_path }}/{{ item.link }}"
    owner: root
    state: link
  with_items: "{{ java_files }}"
  when: use_java | default(false) == true


# ------------------------------
- name: Deploy a hosts file
  copy:
    src: hosts
    dest: /etc/
    backup: yes
    owner: root
    group: root
    mode: 0644
  when: use_hosts | default(false) == true
~~~

#### vars (~/roles/init/handlers/main.yml)
사용할 변수들을 정의합니다. 여기에선 vars 디렉토리 내에 정의했지만 default 디렉토리 내에 main.yml에 정의할 수도 있습니다.

~~~bash
---
# init vars
# ---------------
# User
# ---------------
# Add default group
use_default_group: false
default_group_name: defaultgroup
default_group_id: 2000

# Add users
# 계정을 추가하며 패스워드는 python의 passlib을 이용하여 생성합니다. 이 password는 /etc/shadow에 기재됩니다. 다음과 같이 파이썬 모듈을 설치한 뒤 패스워드를 생성할 수 있습니다.
#   $ sudo pip install passlib
#   $ python -c "from passlib.hash import sha512_crypt; print(sha512_crypt.encrypt('password'))
use_user: false
users:
  - username: test01
    password: $6$rounds=656000$Rd3rShP6n4tfATeK$mRTYKhu12AnESSwCl/kTZBvpcDqrkhDzDDGj04Fx8R4njkRpxqugjq1ktqftdL7fRHRomZ4E2HhUgnziqguEt1
    id: 2001
  - username: test02
    password: $6$rounds=656000$Rd3rShP6n4tfATeK$mRTYKhu12AnESSwCl/kTZBvpcDqrkhDzDDGj04Fx8R4njkRpxqugjq1ktqftdL7fRHRomZ4E2HhUgnziqguEt1
    id: 2002

# sudoers
use_sudoers: false
sudoers:
  - "ubuntu"


# ---------------
# ntp
# ---------------
use_ntp: false


# ---------------
# sysctl
# ---------------
# /etc/sysctl.conf
use_vm_swappiness: false
vm_value:
  - name: vm.swappiness
    value: 1
  - name: vm.overcommit_memory
    value: 0

# file descriptor (/etc/security/limits.conf)
use_file_descriptor: false
max_file_descriptor: 65536


# ---------------
# Apt update, install
# ---------------
use_apt_update: false
use_apt_install: false
apt_install:
  - htop


# ---------------
# java
# ---------------
# jdk를 복제한 뒤 link로 연결합니다.
use_java: false
java_path: /usr/lib/jvm
java_files:
  - { jdk: jdk1.6.0_41, link: java-6-sun }
  - { jdk: jdk1.7.0_71, link: java-7-sun }
  - { jdk: jdk1.8.0_171, link: java-8-sun }


# ---------------
# hosts
# ---------------
use_hosts: false
~~~

## 실행
모든 파일이 구성되면 ansible 루트에서 다음 명령어를 실행하여 초기화를 진행할 수 있습니다.

~~~bash
$ ansible-playbook -i inventory -i init.yml
~~~

## 앞으로
추후 다양한 어플리케이션의 배포, 시작, 종료 등을 ansible로 구성할 예정입니다. 

## 참고
* best practice : [https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
* example : [https://github.com/ansible/ansible-examples](https://github.com/ansible/ansible-examples)
* ansible galaxy: [https://galaxy.ansible.com](https://galaxy.ansible.com)
* 설치 및 기본 기능 익히기 : [http://wiki.tunelinux.pe.kr/display/sysadmin/Ansible](http://wiki.tunelinux.pe.kr/display/sysadmin/Ansible)
* Ansible variables : [https://moonstrike.github.io/ansible/2016/10/08/Ansible-Variables.html](https://moonstrike.github.io/ansible/2016/10/08/Ansible-Variables.html)

