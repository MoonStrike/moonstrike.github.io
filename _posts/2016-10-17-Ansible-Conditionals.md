---
layout: post
title:  "[Ansible] Conditionals 요약"
date:   2016-10-17 10:48:04 +0900
categories: Ansible
comments: true
---

{% raw  %}
# Conditionals

<http://docs.ansible.com/playbooks_conditionals.html>

## 1. `when` statement

`when` 구문에는 `{{ }}` 없이 raw Jinja2 expression을 사용하여 조건을 지정할 수 있음.

단, ***최신 버전에서는 `{{ }}`를 명시하도록 경고 문구가 나옴. 향후 deprecate 예정***

`when` 내에서는 여러 형태의 variable과 ansible_fact 등을 사용할 수 있음.

```
tasks:
  - name: "shut down Debian flavored systems"
    command: /sbin/shutdown -t now
    when: "{{ ansible_os_family == 'Debian' }}"
    # note that Ansible facts and vars like ansible_os_family can be used
    # directly in conditionals without double curly braces

```

`()`을 사용하여 조건 여러 개를 자유롭게 grouping할 수 있음.

```
...
    when: "{{ (ansible_distribution == 'CentOS' and ansible_distribution_major_version == '6') or
          (ansible_distribution == 'Debian' and ansible_distribution_major_version == '7') }}"
```

and 조건은 아래처럼도 표현 가능함.

```
...
    when:
      - "{{ ansible_distribution == 'CentOS' }}"
      - "{{ ansible_distribution_major_version == '6' }}"
```

### Jinja2 filter 사용

Jinja2에서 제공하는 많은 filter/test 등을 사용할 수 있음.

Ansible에서 이 중 일부에 대해서는 Ansible에서만 유효한 방식으로 쉽게 쓸 수 있도록 자체 제공함.

- `failed`, `succeeded`, `skipped`, `changed`, ...

```
tasks:
  - command: /bin/false
    register: result
    # 아래 줄이 없으면 error-exit 되어버림.
    ignore_errors: True

  - command: /bin/something
    when: "{{ result|failed }}"

  # In older versions of ansible use |success, now both are valid but succeeded uses the correct tense.
  - command: /bin/something_else
    when: "{{ result|succeeded }}"

  - command: /bin/still/something_else
    when: "{{ result|skipped }}"
```

### `int` 필터를 통해 string -> int conversion 후 산술 연산 예

```
tasks:
  - shell: echo "only on Red Hat 6, derivatives, and later"
    # int 필터를 통해 string -> int 변환
    when: "{{ ansible_os_family == 'RedHat' and ansible_lsb.major_release|int >= 6 }}"
```

### boolean 변수 사용 예

```
vars:
  epic: true
  ...
tasks:
  - shell: echo "This certainly is epic!"
    when: "{{ epic }}"

  - shell: echo "This certainly isn't epic!"
    when: "{{ not epic }}"

```

### 사용할 변수가 정의되지 않은 경우 대처 예 - `defined`/`undefined` or `default()`

```
tasks:
    - shell: echo "I've got '{{ foo }}' and am not afraid to use it!"
      when: "{{ foo is defined }}"

    - fail: msg="Bailing out. this play requires 'bar'"
      when: "{{ bar is undefined }}"

    ...
    - shell: echo "I'm the '{{ foo|default('John Doe') }}'"

```

## 2. Loops and Conditionals

`when`과 `with_item` 같은 loop 가 같이 사용되는 경우에는 `when` 조건이 loop의 매 iteration마다 독립적으로 처리됨!

```
tasks:
    - command: echo {{ item }}
      with_items: [ 0, 2, 4, 6, 8, 10 ]
      # 0, 2, 4, 6, 8, 10 에 대해 각각 처리됨
      when: item > 5
```

loop 에서 사용되는 값이 정의되지 않는 경우 전체를 skip하기 위해서는 아래와 같이 `default()`를 이용함.

```
- command: echo {{ item }}
  # list
  with_items: "{{ mylist|default([]) }}"
  when: item > 5

- command: echo {{ item.key }}
  # dictionary
  with_dict: "{{ mydict|default({}) }}"
  when: item.value > 5

```

## 3. Loading in Custom Facts

custom fact도 사전에 gather하기만 하면, 사용이 간단함.

```
tasks:
    - name: gather site specific fact data
      action: site_facts
    - command: /usr/bin/thingy
      when: my_custom_fact_just_retrieved_from_the_remote_system == '1234'
```

## 4. Applying ‘when’ to roles and includes

role이나 task의 `include`에도 `when` 조건을 사용가능함.
2.0 이후부터는 play에도 가능함.

```
# task include
...
tasks:
  - name: some task
    shell: echo "test"

  ...

  - include: tasks/sometasks.yml
    when: "'reticulating splines' in output"
```

```
# role include
- hosts: webservers
  roles:
     - { role: debian_stock_config, when: ansible_os_family == 'Debian' }
```

## 5. Conditional Imports

조건에 따라 다른 variable이나 파일들을 import할 수 있음.

```
---
- hosts: all
  remote_user: root
  vars_files:
    - "vars/common.yml"
    # 앞의 것이 없으면 뒤의 것이 선택됨
    - [ "vars/{{ ansible_os_family }}.yml", "vars/os_defaults.yml" ]
  tasks:
  - name: make sure apache is running
    service: name={{ apache }} state=running
```

## 6. Selecting Files And Templates Based On Variables - `with_first_found`

`with_first_found` 구문을 이용하여 주어진 조건에 최초로 부합하는 것을 선택가능함.

```
- name: template a file
  template: src={{ item }} dest=/etc/myapp/foo.conf
  with_first_found:
    - files:
       - {{ ansible_distribution }}.conf
       - default.conf
      paths:
       - search_location_one/somedir/
       - /opt/other_location/somedir/
```

## 7. Register Variables - `register`

`register`로 저장된 실행 결과의 string contents는 `.stdout`에 저장됨.

```
- name: test play
  hosts: all

  tasks:

      - shell: cat /etc/motd
        register: motd_contents

      - shell: echo "motd contains the word hi"
        when: motd_contents.stdout.find('hi') != -1
```

`with_items` 등으로 loop를 돌린 경우에는 `.stdout_lines`에 저장됨.

`.stdout`을 `split()`로 분리해도 동일한 결과임.

```
- name: registered variable usage as a with_items list
  hosts: all

  tasks:

      - name: retrieve the list of home directories
        command: ls /home
        register: home_dirs

      - name: add home dirs to the backup spooler
        file: path=/mnt/bkspool/{{ item }} src=/home/{{ item }} state=link
        with_items: "{{ home_dirs.stdout_lines }}"
        # same as with_items: "{{ home_dirs.stdout.split() }}"
```

실행결과가 빈 경우에는 아래와 같이 existence check 가능함. (`.stdout == ""`)

```
- name: check registered variable for emptiness
  hosts: all

  tasks:

      - name: list contents of directory
        command: ls mydir
        register: contents

      - name: check contents for emptiness
        debug: msg="Directory is empty"
        when: contents.stdout == ""
```

{% endraw %}
