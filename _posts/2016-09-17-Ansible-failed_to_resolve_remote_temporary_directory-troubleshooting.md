---
layout: post
title:  "[Ansible] failed to resolve remote temporary directory from 오류 발생 시 대처법"
date:   2016-09-17 17:14:01 +0900
categories: Ansible
comments: true
---

Ansible을 실행할 때 실행 환경에 따라, 아래와 같은 오류가 불규칙적으로 발생하는 경우가 있다.

`failed to resolve remote temporary directory from ...`

내 경우에는 CentOS 6.x, Ansible 2.0 ~ Ansible 2.1 에서 겪었던 것으로 기억이 되는데,
이 경우 아래와 같은 workarround 를 적용하면, 문제를 해결할 수 있다. (Ansible 2.2 에서는 해결될 듯)

### 해결 방법 - `ansible.cfg`에 아래와 같은 항목 추가

```
[ssh_connection]
ssh_args = -o ControlMaster=no
```

참조링크: [https://github.com/ansible/ansible/issues/13876#issuecomment-184295571](https://github.com/ansible/ansible/issues/13876#issuecomment-184295571)
