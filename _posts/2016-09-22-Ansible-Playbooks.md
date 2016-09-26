---
layout: post
title:  "[Ansible] Playbooks 요약 정리"
date:   2016-09-22 15:51:11 +0900
categories: Ansible
comments: true
---

{% raw  %}
# 공식 Playbooks tutorial 요약

## [Intro to Playbooks]
<http://docs.ansible.com/playbooks_intro.html>

### 0. About Playbooks
- Ansible의 configuration, deployment, orchestration 언어
- [ansible-exmamples repository](https://github.com/ansible/ansible-examples) 참고
- 아래와 같이 실행

```
ansible-playbook playbook.yml -f 10 [--verbose]

# host별 예상 실행 결과를 실행 전에 확인
ansible-playbook playbook.yml --list-hosts
```


### 1. Playbook Language Example
- one or more “plays” in a list
- play: hosts to roles(tasks) map
- task: a call to ansible module
- example within a play

```
---
# remote hosts/groups patterns
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  # user account
  remote_user: yourname
  become: yes
  # task list
  tasks:
    - name: disable selinux
      command: /sbin/setenforce 0
    - name: ensure apache is at the latest version
      yum: pkg=httpd state=latest
      become: yes
      become_user: root
    - name: write the apache config file
      template: src=/srv/httpd.j2 dest=/etc/httpd.conf
  # handlers
  notify: restart apache
    - name: ensure apache is running
      service: name=httpd state=started
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

### 2. Basics

#### Hosts and Users
- 위 예제 참고
- sudo password 는 ansible-playbook에 `--ask-become-pass` 옵션으로 지정(not in playbook)

#### Tasks list
- 각 play에는 tasks list 포함
- task는 in order, one at a time, against all machines로 실행됨
    - 하나의 task가 끝나야 다음 task로 진행됨(top to bottom)
    - 2.0부터는 [`strategies`](http://docs.ansible.com/ansible/playbooks_strategies.html)를 지정가능함
	    - `linear`, `free`
- module은 idempotent하므로 재실행하면 동일한 상태가 됨
    - `command`나 `shell`은 경우에 따라 아닐 수 있음
- task: name + module
- command, shell module은 exit value 를 상관하기 때문에 exit code가 zero가 아닌 경우는 작업을 해줘야 함

```
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand || /bin/true

# 또는
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand
    ignore_errors: True

# variable은 vars 섹션에서 선언하면 사용 가능함
tasks:
  - name: create a virtual host file for {{ vhost }}
    template: src=somefile.j2 dest=/etc/httpd/conf.d/{{ vhost }}
```

### 3. Handlers: Running Operations On Change
- task의 맨 뒤에 `notify` action 설정(handler의 이름으로 호출)
- 여러 번 조건을 만족해도 단 한번만 실행됨
- handler는 특별한 task의 list이며 이름으로 참조됨
- 대개 service restart 등에 쓰임

```
tasks:
  - name: template configuration file
    template: src=template.j2 dest=/etc/foo.conf
    notify:
      - restart memcached
      - restart apache
.....
handlers:
  - name: restart memcached
    service:  name=memcached state=restarted

  - name: restart apache
    service: name=apache state=restarted
```

- Ansible 2.2부터는 `listen` 추가되어 task -> multiple handler trigger를 보다 편하게

```
handlers:
  - name: restart memcached
    service: name=memcached state=restarted
    listen: "restart web services"

  - name: restart apache
    service: name=apache state=restarted
    listen: "restart web services"

tasks:
  - name: restart everything
    command: echo "this task will restart the web services"
    notify: "restart web services"
```

### 4. Ansible-Pull
ansible의 architecture와 반대로 node들이 conf 등을 pull 하고 싶은 경우 이용 가능

-----

## [Playbook Roles and Include Statements]
<http://docs.ansible.com/playbooks_roles.html>

### 1. Introduction
- task, handler, play(playbook) 등을 별도 파일로부터 include할 수 있음
- Role을 통한 재사용을 통해 big picture에 집중할 수 있게 해 줌

### 2. Task Include Files And Encouraging Reuse
- 기본적인 사용법

```
tasks:
  - include: tasks/foo.yml

# parameterized include
# include된 파일에서 {{ wp_user }}로 참조할 수 있다.
tasks:
  - include: wordpress.yml wp_user=timmy
  - include: wordpress.yml wp_user=alice
  - include: wordpress.yml wp_user=bob

# 1.4 or later
tasks:
  - { include: wordpress.yml, wp_user: timmy, ssh_keys: [ 'keys/one.txt', 'keys/two.txt' ] }

# another alternative syntax (1.0 or later)
tasks:
  - include: wordpress.yml
    vars:
      wp_user: timmy
      some_list_variable:
        - alpha
        - beta
        - gamma

# handler에서도 include를 사용가능하다.
handlers:
  - include: handlers/handlers.yml
```

- include된 것(task, handler)와 regular non-included랑 섞어 쓸 수 있다.
- playbook을 다른 playbook에서도 include할 수 있다. 단, variable 치환 불가

```
- name: this is a play at the top level of a file
  hosts: all
  remote_user: root

  tasks:
    - name: say hi
      tags: foo
      shell: echo "hi..."

# including playbooks
- include: load_balancers.yml
- include: webservers.yml
- include: dbservers.yml
```

### 3. Dynamic versus Static Includes

<http://docs.ansible.com/ansible/playbooks_roles.html#dynamic-versus-static-includes>

- Ansible 2.0부터 `include`를 동적으로 실행하도록 변경하여 아래와 같이 include loop나 include file name parametrize가 가능해짐.
	- 다만, 일부 제약이 생기기 때문에 `static` 옵션이 Ansible 2.1부터 추가됨
	- 특정 조건을 만족하면 Ansible 2.1이 자동으로 include의 static 여부를 판단함
		- loop가 없을 것
		- include 파일명에 변수가 없을 것
		- 설정(`ansible.cfg`, `static:no`)에 명시적으로  `static`을 disable로 하지 않을 것

```
# include loop
- include: foo.yml param={{ item }}
  with_items:
    - 1
    - 2
    - 3

# include 파일명 parametrize
- include: "{{ inventory_hostname }}.yml"

# explicit static include
- include: foo.yml
  static: <yes|no|true|false>
```

### 4. Roles
- version 1.2부터 지원
- Role이란 특정 알려진 파일 구조에 기초하여 vars_files, tasks, handlers를 자동으로 load하는 방법
- role_path는 1.4 이후부터 별도 지정 가능
- task 보다는 role 추천
    - 둘이 섞여 있는 경우는 task는 role 이후에 실행됨

```
# example project 구조
site.yml
webservers.yml
fooservers.yml
roles/
   common/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
   webservers/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/

...

# playbook sample
---
- hosts: webservers
  roles:
    - common
    - webservers
```

- `roles/*rollname*/tasks/main.yml`이 있으면 play에 task로 추가
- `roles/*rollname*/handlers/main.yml`이 있으면 play에 handler로 추가
- `roles/*rollname*/vars/main.yml`이 있으면 play에 variable로 추가
- `roles/*rollname*/meta/main.yml`이 있으면 play에 role dependencies 추가
- 모든 copy, script tasks는 `roles/*rollname*/files`의 파일을 (path를 다 쓰지 않아도) 바로 참조 가능
- 모든 template tasks는 `roles/*rollname*/templates`의 파일을 (path를 다 쓰지 않아도) 바로 참조 가능
- 모든 include tasks는 `roles/*rollname*/tasks`의 파일을 (path를 다 쓰지 않아도) 바로 참조 가능

#### parameterized roles, conditionally applied roles, tagging

```
---
- hosts: webservers
  roles:
    - common
    # parameterized
    - { role: foo_app_instance, dir: '/opt/a',  port: 5000 }
    - { role: foo_app_instance, dir: '/opt/b',  port: 5001 }
    # conditionally apply
    - { role: some_role, when: "ansible_os_family == 'RedHat'" }
    # tagging
    - { role: foo, tags: ["bar", "baz"] }
```

### 5. Role Default Variables
- role 디렉토리에 `defaults/main.yml` 파일을 추가하면 role과 dependent role에 default variable을 추가 가능함

### 6. Role Dependencies
- role 디렉토리에 `meta/main.yml` 파일을 추가하여 dependency 저장

```
---
dependencies:
  - { role: common, some_parameter: 3 }
  - { role: apache, port: 80 }
  - { role: postgres, dbname: blarg, other_parameter: 12 }
```

- full path를 다 쓸 수도 있음.

```
---
dependencies:
  - { role: '/path/to/common/roles/foo', x: 1 }
```

- source control repo나 tar 등에서 읽을 수도 있음(comma-separated format)

```
---
dependencies:
# path, version, friendly role name
  - { role: 'git+http://git.example.com/repos/role-foo,v1.1,foo' }
  - { role: '/path/to/tar/file.tgz,,friendly-name' }
```

- role은 dependency에 한 번만 추가될 수 있으나, `allow_duplicates`로 override 가능

`car` 라는 role이 아래와 같은 dependency를 가지고,

```
---
dependencies:
  - { role: wheel, n: 1 }
  - { role: wheel, n: 2 }
  - { role: wheel, n: 3 }
  - { role: wheel, n: 4 }
```

`wheel`의 `meta/main.yml`이 아래와 같은 경우

```
---
allow_duplicates: yes
dependencies:
  - { role: tire }
  - { role: brake }
```

실행 순서는 아래와 같다.

```
tire(n=1)
brake(n=1)
wheel(n=1)
tire(n=2)
brake(n=2)
wheel(n=2)
...
car
```

### 7. Embeddig Modules In Roles
<http://docs.ansible.com/playbooks_roles.html#embedding-modules-in-roles>


advanced topic. skip for now.


### 8. Ansible Galaxy
- [Ansible Galaxy](http://galaxy.ansible.com/)는 community 개발 Ansible role을 다운로드받을 수 있는 무료 사이트임.
- download용 client ‘ansible-galaxy’은 Ansible 1.4.2 or later에 포함되어 있음

{% endraw %}
