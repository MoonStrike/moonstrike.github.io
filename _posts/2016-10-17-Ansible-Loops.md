---
layout: post
title:  "[Ansible] Loops 요약"
date:   2016-10-17 10:51:04 +0900
categories: Ansible
comments: true
---

{% raw  %}
# Loops

<http://docs.ansible.com/ansible/playbooks_loops.html>

## 1. Standard Loops - `with_items`

`with_items`를 사용하여 기본 loop 지정할 수 있음.
각 iteration에 해당하는 값은 `{{ item }}`으로 접근 가능함.

```
- name: add several users
  user: name={{ item }} state=present groups=wheel
  with_items:
     - testuser1
     - testuser2
```

변수를 YAML list로 저장한 경우 아래와 같이도 사용가능함.

```
with_items: "{{ somelist }}"
```

```
somelist:
  - testuser1
  - testuser2
```

변수를 hash의 list로 저장한 경우, subkey를 사용할 수 있음. `{{ item.name }}`, `{{ item.groups }}`

```
- name: add several users
  user: name={{ item.name }} state=present groups={{ item.groups }}
  with_items:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }
```

## 2. Nested Loops - `with_nested`

`with_nested`를 사용하여 중첩 loop 사용할 수 있음.

```
- name: give users access to multiple databases
  mysql_user: name={{ item[0] }} priv={{ item[1] }}.*:ALL append_privs=yes password=foo
  with_nested:
    - [ 'alice', 'bob' ]
    - [ 'clientdb', 'employeedb', 'providerdb' ]
```

아래와 같이 다른 list variable에서 읽어올 수 있음.

```
- name: here, 'users' contains the above list of employees
  mysql_user: name={{ item[0] }} priv={{ item[1] }}.*:ALL append_privs=yes password=foo
  with_nested:
    - "{{ users }}"
    - [ 'clientdb', 'employeedb', 'providerdb' ]
```

```
users:
  - alice
  - bob
```

## 3. Looping over Hashes - `with_dict`

`with_dict` 를 사용하여 아래와 같이 `users` hash list에 대해서도 loop 를 돌릴 수 있음.

```
---
users:
  alice:
    name: Alice Appleworth
    telephone: 123-456-7890
  bob:
    name: Bob Bananarama
    telephone: 987-654-3210
```

```
tasks:
  - name: Print phone records
    debug: msg="User {{ item.key }} is {{ item.value.name }} ({{ item.value.telephone }})"
    with_dict: "{{ users }}"
```

## 4. Looping over Files - `with_file`

`with_file`를 사용하여 목록에 있는 파일의 내용에 대해 loop를 돌릴 수 있음.
`{{ item }}`은 각 iteration file의 contents를 가리킴.

```
---
- hosts: all

  tasks:

    # emit a debug message containing the content of each file.
    - debug:
        msg: "{{ item }}"
      with_file:
        - first_example_file
        - second_example_file
```

## 5. Looping over Fileglobs - `with_fileglob`

`with_fileglob`를 사용하여 하나의 디렉토리 하에 있는 패턴과 일치하는 모든 파일 path에 대해 loop를 돌릴 수 있음.

상대 경로를 사용할 때에는 주의가 필요함. (role 내부에서 사용하는 경우 절대 경로가 role 기준으로 바뀜)

```
---
- hosts: all

  tasks:

    # first ensure our target directory exists
    - file: dest=/etc/fooapp state=directory

    # copy each file over that matches the given pattern
    - copy: src={{ item }} dest=/etc/fooapp/ owner=root mode=600
      with_fileglob:
        - /playbooks/files/fooapp/*
```

## 6. Looping over Parallel Sets of Data - `with_parallel`

두 list를 짝지어서 loop를 돌리고 싶은 경우 `with_parallel`을 사용함.

```
---
alpha: [ 'a', 'b', 'c', 'd' ]
numbers:  [ 1, 2, 3, 4 ]
```

이 경우, 아래 실행 결과는 `(a,1), (b,2), ...`

```
tasks:
    - debug: msg="{{ item.0 }} and {{ item.1 }}"
      with_together:
        - "{{ alpha }}"
        - "{{ numbers }}"
```

## 7. Looping over Subelements - `with_subelements`
list의 element 하위 요소에 대해 중첩 loop를 사용하고 싶을 때 `with_subelements`을 이용함.

```
---
users:
  - name: alice
    authorized:
      - /tmp/alice/onekey.pub
      - /tmp/alice/twokey.pub
  - name: bob
    authorized:
      - /tmp/bob/id_rsa.pub
```

```
- authorized_key: "user={{ item.0.name }} key='{{ lookup('file', item.1) }}'"
  with_subelements:
     - "{{ users }}"
     - authorized
     - flags:
        skip_missing: true
```

`skip_missing` flag를 `true`로 하는 경우, 해당 subelement가 없으면 오류가 발생하지 않고 skip됨.

## 8. Looping over Integer Sequences - `with_sequence`

숫자들의 list를 생성(ascending order)해서 loop를 돌릴 때 `with_sequence` 사용.
아래와 같은 옵션 제공

- `start`
- `end`
- `stride`: 증가 폭을 지정하는 step 옵션
- `format`: printf style string

```
---
- hosts: all

  tasks:
    # create some test users
    - user: name={{ item }} state=present groups=evens
      with_sequence: start=0 end=32 format=testuser%02x

    # create a series of directories with even numbers for some reason
    - file: dest=/var/stuff/{{ item }} state=directory
      with_sequence: start=4 end=16 stride=2

    # a simpler way to use the sequence plugin
    # create 4 groups
    - group: name=group{{ item }} state=present
      with_sequence: count=4
```

## 9. Random Choices - `with_random_choice`

```
- debug: msg={{ item }}
  with_random_choice:
     - "go through the door"
     - "drink from the goblet"
     - "press the red button"
     - "do nothing"
```

## 10. Do-Until Loops - `until`

- `until`: exit 조건
- `retries`: 반복 회수
- `delay`: 매 실행 시마다 delay

```
- action: shell /usr/bin/foo
  register: result
  until: result.stdout.find("all systems go") != -1
  retries: 5
  delay: 10
```

do-until loop의 실행 결과는 마지막 실행 결과가 task의 실행결과로 리턴됨.

개별 retry의 결과를 보고 싶으면 `-vv` 옵션 사용하면 되며 시도 횟수는 `result.attempts`에 남음.

## 11. Finding First Matched Files - `with_first_found`

정확히는 loop는 아니나, 목록 중 처음 발견되는 파일을 지정할 수 있는 기능이 있음.

```
- name: INTERFACES | Create Ansible header for /etc/network/interfaces
  template: src={{ item }} dest=/etc/foo.conf
  with_first_found:
    - "{{ ansible_virtualization_type }}_foo.conf"
    - "default_foo.conf"
```

아래와 같이 여러 옵션을 지정할 수 있는 long form version이 있음.

- `files`: 파일 path 지정
- `paths`: 찾을 path 지정
- `skip`: true이면, 파일이 없는 경우 skip

```
- name: some configuration template
  template: src={{ item }} dest=/etc/file.cfg mode=0444 owner=root group=root
  with_first_found:
    - files:
       - "{{ inventory_hostname }}/etc/file.cfg"
      paths:
       - ../../../templates.overwrites
       - ../../../templates
    - files:
        - etc/file.cfg
      paths:
        - templates
      skip: true
```

## 12. Iterating Over The Results of a Program Execution - `with_lines`

특정 프로그램의 실행 결과를 line by line으로 loop를 돌릴 때 사용.

```
- name: Example of looping over a command result
  shell: /usr/bin/frobnicate {{ item }}
  with_lines: /usr/bin/frobnications_per_host --param {{ inventory_hostname }}
```

주의할 점은 **control machine에서만 실행된다는 점 !!**
원격 장비에서 실행해야 하는 경우 위에서 살펴본 것처럼 아래와 같이 실행.

```
...
with_items: "{{ command_result.stdout_lines }}"
```

## 13. Looping Over A List With An Index - `with_indexed_items`

list index를 같이 반환하면서 loop를 돌 때 사용.

```
- name: indexed loop demo
  debug: msg="at array position {{ item.0 }} there is a value {{ item.1 }}"
  with_indexed_items: "{{ some_list }}"
```

## 14. Using ini file with a loop - `with_ini`

2.0 이후 기능으로, ini 파일의 내용을 regexp로 추출하고 그 결과 set에 대해 loop를 돌릴 수 있음.

```
# lookup.ini file
[section1]
value1=section1/value1
value2=section1/value2

[section2]
value1=section2/value1
value2=section2/value2
```

```
- debug: msg="{{ item }}"
  with_ini: value[1-2] section=section1 file=lookup.ini re=true
```

실행 결과는 아래와 같음.

```
{
      "changed": false,
      "msg": "All items completed",
      "results": [
          {
              "invocation": {
                  "module_args": "msg=\"section1/value1\"",
                  "module_name": "debug"
              },
              "item": "section1/value1",
              "msg": "section1/value1",
              "verbose_always": true
          },
          {
              "invocation": {
                  "module_args": "msg=\"section1/value2\"",
                  "module_name": "debug"
              },
              "item": "section1/value2",
              "msg": "section1/value2",
              "verbose_always": true
          }
      ]
  }
```

## 15. Flattening A List - `with_flattened`

list of list를 flattening하기 위해서 사용함.

```
----
# file: roles/foo/vars/main.yml
packages_base:
  - [ 'foo-package', 'bar-package' ]
packages_apps:
  - [ ['one-package', 'two-package' ]]
  - [ ['red-package'], ['blue-package']]
```

```
- name: flattened loop demo
  yum: name={{ item }} state=installed
  with_flattened:
     - "{{ packages_base }}"
     - "{{ packages_apps }}"
```


## 16. Using register with a loop

<http://docs.ansible.com/ansible/playbooks_loops.html#using-register-with-a-loop>

`register`를 loop와 같이 사용하면 결과에 개별 iteration 실행 결과에 대한 `results` list가 생김.

`results`를 loop로 하여 다시 후처리를 할 수 있음.

```
- shell: echo "{{ item }}"
  with_items:
    - one
    - two
  register: echo

- name: Fail if return code is not 0
  fail:
    msg: "The command ({{ item.cmd }}) did not have a 0 return code"
  when: item.rc != 0
  with_items: "{{ echo.results }}"
```

## 17. Looping over the inventory

inventory에 대해서 loop를 도는 방법은 대략 아래와 같은 것들이 있음.

### `play_hosts`/`groups` + `with_items`

```
# show all the hosts in the inventory
- debug: msg={{ item }}
  with_items: "{{ groups['all'] }}"

# show all the hosts in the current play
- debug: msg={{ item }}
  with_items: "{{ play_hosts }}"
```

### `with_inventory_hostnames`

```
# show all the hosts in the inventory
- debug: msg={{ item }}
  with_inventory_hostnames: all

# show all the hosts matching the pattern, ie all but the group www
- debug: msg={{ item }}
  with_inventory_hostnames: all:!www
```

## 18. Loop Control - `loop_control`

- 2.1 이후부터 지원하는 기능

### `loop_var`

- 2.0 이후부터는 task include + `with_` loop를 같이 쓸 수 있게 되었음.
- 이 경우 include 된 task에 자체 loop가 있던 경우 `{{ item }}`이 가리키는 값이 덮어씌워지는 문제가 발생함.
- 이 문제의 해결을 위해 loop iteration에서 사용하는 변수명을 `{{ item }}`이 아닌 다른 이름으로 변경할 수 있게 해 주는 기능 추가됨.

```
# main.yml
- include: inner.yml
  with_items:
    - 1
    - 2
    - 3
  loop_control:
    loop_var: outer_item

# inner.yml
- debug: msg="outer item={{ outer_item }} inner item={{ item }}"
  with_items:
    - a
    - b
    - c
```

### `label`

- 복잡한 자료구조를 처리하다보면 사용하지 않는데도 불필요한 출력이 너무 많이 되는 경우 사용함.
- `label`로 지정한 값만 화면에 출력됨.


```
- name: create servers
  digital_ocean: name={{item.name}} state=present ....
  with_items:
    - name: server1
      disks: 3gb
      ram: 15Gb
      netowrk:
        nic01: 100Gb
        nic02: 10Gb
        ...
  loop_control:
    label: "{{item.name}}"
```

### `pause`

loop의 매 iteration마다 잠시 대기 시간을 주고 싶을 때 사용.

```
# main.yml
- name: create servers, pause 3s before creating next
  digital_ocean: name={{item}} state=present ....
  with_items:
    - server1
    - server2
  loop_control:
    pause: 3
```

## 19. Loops and Includes in 2.0

`loop_control`이 2.0에서는 지원되지 않으므로 `loop_var` 대신 `set_fact`를 사용하는 workaround를 써야 함.

```
# main.yml
- include: inner.yml
  with_items:
    - 1
    - 2
    - 3

# inner.yml
- set_fact:
    outer_item: "{{ item }}"

- debug:
    msg: "outer item={{ outer_item }} inner item={{ item }}"
  with_items:
    - a
    - b
    - c
```

## 20. Writing Your Own Iterators

<http://docs.ansible.com/ansible/developing_plugins.html> 참고

{% endraw %}
