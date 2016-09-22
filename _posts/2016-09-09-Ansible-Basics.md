---
layout: post
title:  "[Ansible] Basics"
date:   2016-09-09 12:13:43 +0900
comments: true
categories: Ansible
---

# 기본 참고 자료
- [deview 2014 김용환님 발표 자료](http://www.slideshare.net/deview/1a7ansible)
- [공식 intro 자료](http://docs.ansible.com/intro.html)
- [공식 사이트 비디오](http://www.ansible.com/resources)
- [github 저장소](https://github.com/ansible/ansible)
- [ansible-examples](https://github.com/ansible/ansible-examples)

----------

# 공식 tutorial 요약

## Intro
- Configuration Management Tool, automates cloud provisioning, application deployment, …
- by default, over the **SSH** protocol
- **no agents, no daemons, no database** and no additional custom security infrastructure
- simple language: **Ansible Playbooks - YAML**
- module 을 remote에 던져서 실행하는 concept
- unix의 권한 시스템 반영 (user, sudo 등 지정 가능)
- idempotent 보장

----------

## Installation
<http://docs.ansible.com/intro_installation.html>

- SSH protocol
- Control Machine: (Python 2.6 or above)
- Managed Node: (Python 2.5 or above), 2.5 이하는 `python-simplejson` 설치 필요함
- 소스 버전으로 실행하는 것이 쉬워서 많은 사람들이 dev. version을 사용함
- **Tarballs**, CentOS yum, pip or from source, **brew**
- root 권한 필요없음
- 사전 설치가 필요한 S/W 없음

### 1. Tarball 위치
- <http://releases.ansible.com/ansible/>

### 2. 소스 설치

    $ git clone git://github.com/ansible/ansible.git --recursive
    $ cd ./ansible
    $ source ./hacking/env-setup

Python module 을 별도로 설치해야 함.

	# pip 설치가 필요한 경우에만
	$ sudo easy_install pip
    $ sudo pip install paramiko PyYAML Jinja2 httplib2

update할 경우는 submodules도 해야 함

    $ git pull --rebase
    $ git submodule update --init --recursive


### 3. CentOS Yum으로 설치

    # install the epel-release RPM if needed on CentOS, RHEL, or Scientific Linux
    $ sudo yum install ansible


### 4. Pip로 설치

    $ sudo easy_install pip
    $ sudo pip install ansible

----------

## Getting Started
<http://docs.ansible.com/intro_getting_started.html>

### 1. Remote Connection Information
- native OpenSSH (1.3 or later)
- ControlPersist 기능이 지원되지 않는 OS(RedHat, CentOS)에서는 paramiko 사용
    - [Accelerated Mode](http://docs.ansible.com/playbooks_acceleration.html) 참고
- 간혹 SFTP가 지원되지 않는 경우는 SCP Mode로 switch
- SSH trusted login이 선호되나 password auth, sudo password auth도 지원가능
    - *`--ask-pass`, `--ask-sudo-pass`*
- ping test


		# default inventory file path: /etc/ansible/hosts
	    # for overring it, see below
		$ echo "127.0.0.1" > ~/ansible_hosts
	    $ export ANSIBLE_HOSTS=~/ansible_hosts

	    # password
	    $ ansible all -m ping --ask-pass

	    # trusted login as memex
	    $ ansible all -m ping -u memex

	    # trusted login as memex, sudoing root
	    $ ansible all -m ping -u memex --sudo

	    # as memex, sudoing to batman
	    $ ansible all -m ping -u memex --sudo --sudo-user batman

- run a live command

        $ ansible all -a "/bin/echo hello"


- host에 최초 SSH접속하는 경우 yes/no를 응답해야 하는데 이 경우에 대한 대처
    - `/etc/ansible/ansible.cfg` 또는 `~/.ansible.cfg` 또는 `./ansible.cfg`에 아래 내용 추가

      > [defaults]
      > host_key_checking = False

    - 또는 환경변수 세팅

            $ export ANSIBLE_HOST_KEY_CHECKING=False

----------

## Inventory
<http://docs.ansible.com/intro_inventory.html>

- default inventory file: *`/etc/ansible/hosts`*, 환경 변수로 지정가능

        export ANSIBLE_HOSTS=~/ansible_hosts

- log_path 지정 가능
- host 및 group을 inventory 에 지정 가능

### 1. Hosts and Groups
- inventory file은 아래와 같은 형태를 가짐
- 한 호스트가 동시에 여러 곳에 속할 수 있음
- port 번호 지정 가능
- 특정 variable을 host, group 등에 할당 가능함
    - host var나 group var는 다른 파일로 찢어놓는 게 좋음
- Ansible 만의 host alias 지정 가능(위 variable 기능 이용)
- group of group도 가능함

#### - example

```
mail.example.com

[webservers]
foo.example.com
bar.example.com
www[01:50].example.com

[dbservers]
foo.example.com
one.example.com
two.example.com
three.example.com
badwolf.example.com:5309
jumper ansible_ssh_port=5555 ansible_ssh_host=192.168.1.50
db-[a:f].example.com
```

#### - connection type and user (per host basis)
- connection type: paramiko, ssh, local(API 호출 등 local connection)

```
[targets]

localhost              ansible_connection=local
other1.example.com     ansible_connection=ssh        ansible_ssh_user=mpdehaan
other2.example.com     ansible_connection=ssh        ansible_ssh_user=mdehaan
```

### 2. Host Variables
- 자세한 사항은 Playbook에서
- 파일 이름은 var. name

```
[atlanta]
host1 http_port=80 maxRequestsPerChild=808
host2 http_port=303 maxRequestsPerChild=909
```

### 3. Group Variables

```
[atlanta]
host1
host2

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```

### 4. Groups of Groups, and Group Variables
- group들의 group을 만들 수 있고 여기에 변수 할당도 가능함
- 단 ansible-playbook에서만 사용가능(ansible 불가)

```
[atlanta]
host1
host2

[raleigh]
host2
host3

[southeast:children]
atlanta
raleigh

[southeast:vars]
some_server=foo.southeast.example.com
halon_system_timeout=30
self_destruct_countdown=60
escape_pods=2

[usa:children]
southeast
northeast
southwest
northwest
```

### 5. Splitting Out Host and Group Specific Data

- 변수 값들은 보통 inventory에 넣지 않고 별도 파일에 저장하는 것이 일반적임
- 변수 파일의 형식은 YAML

```
ntp_server: acme.example.org
database_server: storage.example.org
```

- 변수 파일 위치
    - root는 Playbook 디렉토리 또는 inventory 디렉토리: 둘 다 있는 경우는 전자가 override

```
    # inventory 파일이 /etc/ansible/hosts에 있는 경우를 가정함

    # group variables
    /etc/ansible/group_vars/raleigh
    /etc/ansible/group_vars/webservers

    # host variables
    /etc/ansible/host_vars/foosball

    # 아래와 같이 디렉토리 이름을 변수명으로 두고 하위에 다른 임의의 파일로 쪼개도 됨
    /etc/ansible/group_vars/raleigh/db_settings
    /etc/ansible/group_vars/raleigh/cluster_settings
```

### 6. List of Behavioral Inventory Paramaters

기본 ansible 변수는 아래와 같음

`ansible_ssh_host`: 접속할 ssh host명 (alias 이름과 다른 경우)

`ansible_ssh_port`: ssh port (22가 아닌 경우)

`ansible_ssh_user`: 기본 ssh user name

`ansible_ssh_pass`: ssh password (insecure함!)

`ansible_sudo_pass`: sudo password (insecure함!)

`ansible_sudo_exe`: (new in version 1.8) sudo command path.

`ansible_connection`: Connection type. local, ssh 또는 paramiko. default 값은 paramiko(before Ansible 1.2)1.2 이후는 'smart'  - ControlPersist가 지원되는지 여부에 따라 자동으로 결정

`ansible_ssh_private_key_file`: ssh Private key 파일

`ansible_shell_type`:  target 시스템의 shell type. 기본값은 'sh'-style syntax.  'csh'나 'fish' 도 사용가능

`ansible_python_interpreter`: target host의 python path.

`ansible_*_interpreter`: 위 옵션과 유사. ruby나 perl 등에도 사용가능

### 7. dynamic inventory
<http://docs.ansible.com/intro_dynamic_inventory.html>

- inventory를 외부 S/W system에 저장(e.g. CMDB software, AWS EC2/Eucalyptus, Openstack)
- 자세한 내용은 링크 참조
- Ansible Tower로 관리, GUI제공 가능

----------

## Pattern
<http://docs.ansible.com/intro_patterns.html>

- ansible이 주어진 conf나 IT 프로세스를 **어느 대상(hosts or groups)에 적용할지** 선택하는 방법
- host나 group 등을 지정할 때 다양한 방법 사용 가능
    - IP나 host 명으로 직접 지정(wildcard나  range도 가능)
    - group으로 지정 가능(index, range, regex, AND/OR/EXCLUDE…)

### 1. 사용법

`ansible <pattern_goes_here> -m <module_name> -a <arguments>`

### 2. command 예제
    # example
    ansible webservers -m service -a "name=httpd state=restarted"

    # 대상 제한
    ansible-playbook site.yml --limit datacenter2

    # 대상 제한 - retry_hosts.txt 파일 안으로 제한
    ansible-playbook site.yml --limit @retry_hosts.txt

### 3. pattern 예제

```
# 모든 대상 지정
all
*

# 특정 host(s) 명 또는 주소로 지정
one.example.com
one.example.com:two.example.com
192.168.1.50
192.168.1.*

# 특정 group(s)명으로 지정, ':'는 OR를 의미함
webservers
webservers:dbservers

# 특정 group 배제(!)
webservers:!phoenix

# intersection(&)
webservers:&staging

# 조합
webservers:dbservers:&staging:!phoenix

# 변수 사용 가능(드문 사용예)
webservers:!{{excluded}}:&{{required}}

# wildcard 사용(Individual host names, IPs and groups)
*.example.com
*.com

# 동시에 wildcard pattern 과 groups 조합
one*.com:dbservers

# group에서 numbered server 지정 가능
webservers[0]
webservers[0:25]

# regular expresion(~)
~(web|db).*\.example\.com
```

----------

## Ad-Hoc command
<http://docs.ansible.com/intro_adhoc.html>

- 1회성으로 실행하는 명령을 뜻함
- sudo 권한을 특정 path만 주는 방식은 호환되지 않음. Ansible Tower로 대응가능
- 실행할 module을 선택가능(-m) : default는 '***command***'
    - 단, piping이나 shell var. 지원안함. 대신 ‘***shell***’을 사용할 것

            ansible raleigh -m shell -a 'echo $TERM’

### 1. Parallelism and Shell Commands
    # 10 parallel forks
    $ ansible atlanta -a "/sbin/reboot" -f 10

    # sudo
    $ ansible atlanta -a "/usr/bin/foo" -u username --sudo [--ask-sudo-pass]

    # sudo user 변경
    $ ansible atlanta -a "/usr/bin/foo" -u username -U otheruser [--ask-sudo-pass]

### 2. File Transfer

#### `copy` module
    $ ansible atlanta -m copy -a "src=/etc/hosts dest=/tmp/hosts"

playbook에서는 `template` module 참고


#### `file` module
파일의 ownership/permission 변경 가능(copy module에도 사용가능)
`state=directory`로 dir 생성가능, `state=absent`로 dir/file 삭제 가능


    $ ansible webservers -m file -a "dest=/path/to/c mode=755 owner=mdehaan group=mdehaan state=directory"
    $ ansible webservers -m file -a "dest=/path/to/c state=absent"


### 3. Managing Packages: `yum`/`apt` module

    $ ansible webservers -m yum -a "name=acme-1.5 state=present"
    $ ansible webservers -m yum -a "name=acme state=latest"
    $ ansible webservers -m yum -a "name=acme state=absent"


### 4. Users and Groups: `user` module
user 및 group 조작

    $ ansible all -m user -a "name=foo password=<crypted password here>"
    $ ansible all -m user -a "name=foo state=absent"

### 5. Deploying From Source Control: `git` module
소스 관리: 소스가 변경된 경우 바로 update하고 deploy 등이 가능함

    $ ansible webservers -m git -a "repo=git://foo.example.org/repo.git dest=/srv/myapp version=HEAD”


### 6. Managing Services: `service` module
service 관리: `state=started`, `stopped` or `restarted`

    $ ansible webservers -m service -a "name=httpd state=restarted”


### 7. Time Limited Background Operations: `asnyc_status` module

    # don't poll
    $ ansible all -B 3600 -a "/usr/bin/long_running_operation --do-stuff"

    # 추후 상태 확인
    $ ansible all -m async_status -a "jid=123456789"

    # poll(최대 30분동안 실행 & 60초마다 status 확인)
    $ ansible all -B 1800 -P 60 -a "/usr/bin/long_running_operation --do-stuff"


### 8. Gathering Facts: `setup` module
시스템 info. 확인

    $ ansible all -m setup
