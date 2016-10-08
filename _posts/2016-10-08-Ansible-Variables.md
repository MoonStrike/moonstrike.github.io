---
layout: post
title:  "[Ansible] Variables 요약"
date:   2016-10-08 16:42:17 +0900
categories: Ansible
comments: true
---

{% raw  %}

# Variables
<http://docs.ansible.com/playbooks_variables.html>

Ansible에서는 변수를 사용하여 공통된 내용 속에 시스템 간의 차이를 표현할 수 있음.

예>

- `template`에서 value 채우기
- `when`을 사용하여 조건절에서 분기에 이용
- `group_by`을 사용하여 변수 값에 따라 묶음
- 암호화 변수 사용 가능(`vault`)
- ...

## 1. Naming Rule
letters, numbers, underscores로 구성해야 하며, letter로 시작해야 함

```
# OK
foo_port
foo5

# NOT OK
foo_port
foo port
foo.port
12
```

아래와 같이 YAML dictionary 지원됨

```
foo:
  field1: one
  field2: two
```

참조할 때는 아래와 같이 하면 됨(둘다 가능)

```
foo['field1']
foo.field1
```

하지만, **되도록이면 전자를 택하는 것이 좋음**.

`__`로 시작하고 끝나는 key나 아래 목록에 해당하는 key 이름인 경우에는 python 내부의 attribute와 충돌이 날 수 있음

```
add, append, as_integer_ratio, bit_length, capitalize, center, clear, conjugate, copy, count, decode, denominator, difference, difference_update, discard, encode, endswith, expandtabs, extend, find, format, fromhex, fromkeys, get, has_key, hex, imag, index, insert, intersection, intersection_update, isalnum, isalpha, isdecimal, isdigit, isdisjoint, is_integer, islower, isnumeric, isspace, issubset, issuperset, istitle, isupper, items, iteritems, iterkeys, itervalues, join, keys, ljust, lower, lstrip, numerator, partition, pop, popitem, real, remove, replace, reverse, rfind, rindex, rjust, rpartition, rsplit, rstrip, setdefault, sort, split, splitlines, startswith, strip, swapcase, symmetric_difference, symmetric_difference_update, title, translate, union, update, upper, values, viewitems, viewkeys, viewvalues, zfill
```

## 2. 변수 선언 가능 위치

### Inventory

- host
- group
- ...

[Inventory](http://docs.ansible.com/ansible/intro_inventory.html) 문서 참고

### playbook
- `vars`
- `var_files`

```
- hosts: webservers
  vars:
    http_port: 80
```

### include된 file이나 role
<http://docs.ansible.com/ansible/playbooks_roles.html> 참고

## 3. 변수 참조 방법(`Jinja2`)
Ansible은 playbook 내에서 변수를 참조할 때, [`Jinja2`](http://jinja.pocoo.org) templating system을 사용함.

`Jinja2`에 아주 많은 기능들이 있지만 일단은 아래만 알아도 기본은 충분함.

```
My amp goes to {{ max_amp_value }}

...

template: src=foo.cfg.j2 dest={{ remote_install_path }}/foo.cfg
```

- 기본적으로 해당 host의 scope내의 변수에 모두 접근 가능
- 하지만, 다른 호스트의 변수에도 접근할 수 있는 방법이 있음(`groups`, `hostvars` 등을 이용)

## 4. `Jinja2` Filter
Ansible은 Jinja2의 필터를 사용하여 변수나 데이터를 다른 형태로 변환하는 것이 가능함.  아래 문서 참고


- Jinja2 [builtin filters](http://jinja.pocoo.org/docs/templates/#builtin-filters)
- <http://docs.ansible.com/ansible/playbooks_filters.html>


## 5. YAML 파싱 오류 주의
YAML 문법은 `{{ foo }}`으로 시작하는 값을 만나면 전체 라인을 quote하도록 요구함.

사용자가 YAML dictionary를 사용하는 경우와 혼동하지 않도록 하기 위함.

- <http://docs.ansible.com/ansible/YAMLSyntax.html> 참고

```
- hosts: app_servers
  vars:
      # NOT OK
      # app_path: {{ base_path }}/22

      # OK
      app_path: "{{ base_path }}/22"
```

## 6. Facts - 각종 시스템 관련 변수를 비롯한 동적 생성 정보
`ansible hostname -m setup`로 확인 가능. 굉장히 많은 것이 나옴.
보통 Playbook 실행 시 최초에 저절로 gather됨.

예>

- `{{ ansible_hostname }}`
- `{{ ansible_nodename }}`
- `{{ ansible_eth0 }}`
- `{{ ansible_env.HOME }}`
- `{{ ansible_version }}`
- ...

조건문이나 template 에서 다양하게 사용가능하며, `group_by` module을 사용해서 동적인 inventory 생성에도 사용가능함.

### A. Facts 끄기
필요없다고 확신이 있는 경우에 Facts를 끄면, 대규모 장비를 다룰 때 성능 향상이 있음

```
- hosts: whatever
  gather_facts: no
```

### B. Local Facts(Facts.d)

`setup` module로 얻을 수 있는 fact에 user-defined value(custom fact)를 추가할 수 있도록 제공하는 mechanism.
아래와 같이 사용 가능

1. `/etc/ansible/facts.d` 디렉토리에 `.fact`로 끝나는 파일이 있고,
2. 위 파일이 JSON, INI 형식이거나 혹은 JSON을 실행결과로 반환하는 실행파일인 경우

예> `etc/ansible/facts.d/preferences.fact`이 아래 내용으로 존재하는 경우

```
[general]
asdf=1
bar=2
XYZ=3
```

`ansible <hostname> -m setup -a "filter=ansible_local"` 명령을 실행하면,

아래와 같이 fact에 추가되며,`{{ ansible_local.preferences.general.asdf }}`와 같이 사용 가능함

```
"ansible_local": {
        # .fact 파일의 이름
        "preferences": {
            "general": {
                "asdf" : "1",
                "bar"  : "2",
                # key 이름이 소문자로 변경됨
                "xyz"  : "3"
            }
        }
 }
```

playbook을 통해 custom fact를 등록하도록 하면, 명시적으로 `setup` module을 실행해야 바로 효과가 나타난다.

```
- hosts: webservers
  tasks:
    - name: create directory for ansible custom facts
      file: state=directory recurse=yes path=/etc/ansible/facts.d
    - name: install custom impi fact
      copy: src=ipmi.fact dest=/etc/ansible/facts.d
    - name: re-read facts after adding custom fact
      setup: filter=ansible_local
```

### C. Fact Caching
Ansible 실행 시에 host들의 fact를 미리 영속적인 장소에 저장해 놓고 나중에 쓸 수 있게 하는 매커니즘. 아래와 같은 상황에 유리할 수 있음.

#### 예. 다른 서버에 해당하는 변수를 참고하는 경우

- `{{ hostvars['asdf.com']['ansible_os_family'] }}`와 같이 타 서버 변수를 참조할 수 있음
  - 단, 이 경우 실제 Playbook 실행 중에 해당 호스트에 한 번이라도 사전 방문(실행)이 되어야 함.
  - (default fact caching off)
- 하지만 fact caching을 enable시키고 사전에 미리 서버에서 fact를 얻어서 저장해 놓으면...
  - 이후에는 해당 호스트 방문 없이도 그 호스트에 대한 fact(변수)를 사용가능하게 됨.
- 대량의 서버 리스트가 있고 그 중 소수 서버에 대한 작업을 하는데 그 외 서버의 정보가 필요한 경우 등에 유용함.
  - 예. 밤중에 batch로 대량 서버 정보 수집하여 fact caching 후 실행 시 사용

inventory 파일의 내용이 아래와 같다고 가정함.

```
[group1]
asdf.com

[group2]
qwer.com
```

실행하려는 Playbook이 아래와 같다고 가정함.

```
# qwer.com
- hosts: group2
  tasks:
    - name: test
      # fact caching off(default)인 경우 NOT OK!
      debug: msg="{{ hostvars['asdf.com']['ansible_os_family'] }}"
```

이 경우 `asdf.com`에 대해서는 사전에 ansible `setup`이 실행되지 않았으므로 `hostvars['asdf.com']` 값을 가져오지 못함.(변수는 존재하나 value가 없음)

하지만, fact caching을 on 한 후, 사전에 먼저 아래 Playbook을 실행하고 위 Playbook을 다시 실행하면 잘됨.

```
# asdf.com
- hosts: group1
  tasks:
    - name: test
      ...
```

#### Fact caching enable 방법

`ansible.cfg`에 아래 내용 추가

```
# Redis 사용 예
[defaults]
# 'smart' or 'explicit'
gathering = smart
fact_caching = redis
# seconds
fact_caching_timeout = 86400

...
# JSON 형식 파일 사용 예
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /path/to/cachedir
# seconds
fact_caching_timeout = 86400
```

## 7. task 실행 결과를 변수에 저장
task 실행 결과를 저장하고자 할 때에는 `register` 지시어를 사용하면 됨.

- module 마다 결과 포맷이 다 다르기 때문에 `-v` 옵션으로 확인 가능
  - 또는 명시적으로 `debug` module 사용(`debug: var=foo_result`)
- `with_items` 등으로 loop를 도는 경우에는 `register`로 저장한 변수 하위에 `results` 배열로 저장됨
- task가 skipped 되거나 failed 되어도 `register` 변수는 생성됨
  - 생성을 피하고 싶은 경우는 `tags`를 사용하는 방법 뿐..


```
- hosts: web_servers
  tasks:
     - shell: /usr/bin/foo
       register: foo_result
       ignore_errors: True

     - shell: /usr/bin/bar
       when: foo_result.rc == 5
```

## 8. 복잡한 다단계 데이터에 접근하기

```
{{ ansible_eth0["ipv4"]["address"] }}
또는

{{ ansible_eth0.ipv4.address }}

배열은 아래와 같이 element에 접근 가능
{{ foo[0] }}
```

## 9. 기본 Magic Variables 및 타 host에 대한 변수
default로 ansible이 제공하고 있는 변수들이 있는데 대부분 유용하게 사용가능함. 당연히 reserved name이므로 재사용 불가

### A. `hostvars`
다른 호스트들의 변수 또는 fact를 얻을 때 유용함.

```
{{ hostvars['test.example.com']['ansible_distribution'] }}
```

### B. `groups`
inventory 내 전체 group list.
아래 패턴을 자주 사용하게 됨. fact caching 에 유의가 필요할 수 있음.

```
{% for host in groups['app_servers'] %}
   {{ hostvars[host]['ansible_eth0']['ipv4']['address'] }}
{% endfor %}
```

### C. `group_names`
현재 host가 속한 group 이름 list

```
{% if 'webserver' in group_names %}
   # some part of a configuration file that only applies to webservers
{% endif %}
```

### D. `inventory_hostname`, `inventory_hostname_short`
inventory에 실제  기록되어 있는 host 이름. `fact`에 있는 `ansible_hostname`도 비교하여 참고할 것.

`inventory_hostname_short`는 FQDN 형식의 이름인 경우 첫번째 `.`까지의 이름임.

### E. 각종 디렉토리/파일 관련 변수
- `inventory_dir`:  현재 실행 중인 inventory 파일이 있는 디렉토리
- `inventory_file`: 현재 실행 중인 inventory 파일의 pathname + filename
- `playbook_dir`: 현재 실행 중인 playbook이 있는 기본 디렉토리
- `role_path`: 현재 role의 pathname

### F. `play_hosts`(2.2 이전), `ansible_play_batch`(2.2 이후)
- `ansible_play_hosts`:현재 play에서 아직 active인 모든 host list
- `ansible_play_batch`: 위와 동일하나 `serial`로 정의되는 batch에만 해당하는 host list
  - 2.2 이전에는 `play_hosts`

### G. 기타
- `ansible_check_mode`: 2.1 이후 추가. `--check` 옵션으로 실행한 경우 `True`로 설정됨

## 10. Variable File Separation
정리 상의 편의 또는 보안 등의 이유로 변수를 담고 있는 별도 파일로 분리할 수 있음.

```
---

- hosts: all
  remote_user: root
  vars:
    favcolor: blue
  vars_files:
    - /vars/external_vars.yml

  tasks:
  ...
```

```
---
# in the above example, this would be vars/external_vars.yml
somevar: somevalue
password: magic
```

## 11. Command Line로 변수 전달
`--extra-vars` 옵션을 이용하여 실행시간에 CLI로 변수 값 전달 가능함.

```
---

- hosts: '{{ hosts }}'
  remote_user: '{{ user }}'

  tasks:
     - ...

ansible-playbook release.yml --extra-vars "hosts=vipers user=starbuck"

# 또는 JSON 형식으로
ansible-playbook release.yml --extra-vars \
  '{"hosts": "vipers", "user=starbuck", "ghosts":["inky","pinky","clyde","sue"], release: 1}'
```

단, 변수의 type을 살리고 싶으면 반드시 JSON 형식으로 전달할 것.
기본 `key=value` 형식은 모든 value가 string으로 인식됨.

`@`를 사용하여 JSON 파일로부터 바로 읽을 수도 있음(1.3 이후)

```
ansible-playbook release.yml --extra-vars "@some_file.json"
```

## 12. 변수 우선 순위

[자세한 내용은 생략한다.](http://docs.ansible.com/ansible/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)

2.0 기준으로 아래 순서에 따름(뒤로 갈 수록 우선 순위가 높음)

1. role defaults
2. inventory vars
3. inventory group_vars
4. inventory host_vars
5. playbook group_vars
6. playbook host_vars
7. host facts
8. play vars
9. play vars_prompt
10. play vars_files
11. registered vars
12. set_facts
13. role and include vars
14. block vars (only for tasks in block)
15. task vars (only for the task)
16. extra vars (always win precedence)

## 13. 변수 유효 범위(Scope)
- Global: CLI로 지정하거나 환경변수 값
- Play: 개별 Play에 등록한 변수, role defaults/vars
- Host: Host 별로 설정된 변수(inventory 등)

## 14. 변수 종류와 overriding

### Group/host varaible
- 모든 group에 해당하는 변수는 `group_vars/all`에 저장. 최상위 변수
- 특정 group에 해당하는 변수는 `group_vars/region`과 같이 저장 가능.
  - `all`을 비롯한 부모 group의 변수들을 overriding
- 특정 host에 해당하는 변수는 `host_vars/xxx.yyy.zzz`과 같이 저장 가능.
  - 위 둘을 overriding

### Role variable
Role은 재배포 가능한 단위로서의 Task 모음

- role 내에서 deafult 값이 필요한 변수는 `roles/x/defaults/main.yml`
  - Ansible에서 가장 default이므로 그 외 모든 것에 의해 overriding 가능함
- role 내에 사용할 변수 값을 고정하고 싶은 경우 `roles/x/vars/main.yml` 사용
  - 외부 값에 의해 overriding 안됨! 단, `-e` 옵션은 예외
  - overriding을 허용할 변수는 여기 두지 맙시다.. 아래 paramererized role을 참조

```
roles:
   - { role: app_user, name: Ian    }
   - { role: app_user, name: Terry  }
   - { role: app_user, name: Graham }
   - { role: app_user, name: John   }
```

## 15. Advanced Syntax
<http://docs.ansible.com/ansible/playbooks_advanced_syntax.html>

Jinja2 에서 templating에 사용하지 않아야 하는 문자 등이 있는 경우 아래 방법 중 하나 택.
하지만 `!unsafe`가 가장 편함.
{% endraw %}

- 개별 escaping
- `{{ "{% raw " }}%} ... {{ "{% endraw " }}%}`: Jinja2 자체 template disable 기능
- `!unsafe` 태그

{% raw %}
```
---
my_unsafe_variable: !unsafe 'this variable has {{ characters that should not be treated as a jinja2 template'

...

---
my_unsafe_array:
    - !unsafe 'unsafe element'
    - 'safe element'

my_unsafe_hash:
    unsafe_key: !unsafe 'unsafe value'
```
{% endraw %}
