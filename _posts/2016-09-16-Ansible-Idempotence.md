---
layout: post
title:  "[Ansible] Idempotence"
date:   2016-09-16 15:53:26 +0900
categories: Ansible
comments: true
---
### 기본 개념

<http://docs.ansible.com/ansible/glossary.html> 에서 발췌

> Idempotency:
> The concept that change commands should only be applied when they need to be applied, and that it is better to describe the desired state of a system than the process of how to get to that state.


* 몇 번을 실행해도 동일한 결과가 되는 성질
* 멱등성이 보장되지 않는 연산의 경우 시스템의 상태를 추적해야 하므로, 알고리즘이나 처리 방법이 복잡해짐.
* 기본적으로 네트워크 어플리케이션의 경우 재시도를 염두에 둬야 함
* Chef, Puppet 등 많은 프로비저닝 툴들은 멱등성을 보장함.

### Ansible에서 멱등성이 보장되지 않는 경우

기본적으로 내장된 Ansible module은 멱등성이 보장되나, `shell`이나 `command` 등은 멱등성이 보장되지 않는다. 이 경우, 별도의 조치를 취해 멱등성을 보장하는 것이 좋다.

#### 예 1. 파일에 특정 문자열을 추가하는 경우

멱등성 보장 안됨.

```
---
- hosts: dev-servers
  tasks:
    - shell: echo test >> /tmp/forbar
```

멱등성 보장되도록 수정.

```
---
- hosts: dev-servers
  tasks:
    - shell: cat /tmp/forbar
      register: result

    - shell: echo test >> /tmp/foobar
      when: result.stdout.find('test') == -1
```

#### 예 2. `creates` 또는 `removes`을 사용해서 멱등성 보장
`creates`를 사용하면, 해당 파일(또는 glob pattern)이 존재하는 경우에는 task가 실행되지 않는다. 이 점을 이용하면 멱등성을 보장할 수 있는 경우가 있다. `removes`는 반대로 특정 파일이 존재하지 않으면 task가 실행되지 않는다.

```
- name: install pyton-apt
  shell: apt-get install -y python-apt >> /home/x/output.log creates=/home/x/output.log
```

### 추가 참고링크
* <http://docs.ansible.com/ansible/command_module.html>
* <http://knight76.tistory.com/entry/ansible-멱등성idempotent-용어-이해하기>
* <https://en.wikipedia.org/wiki/Idempotence>
* <http://stackoverflow.com/questions/24576315/how-do-i-make-an-idempontent-shell-in-ansible?answertab=votes#tab-top>
