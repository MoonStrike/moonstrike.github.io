---
layout: post
title:  "[Ansible] 유용한 개발/디버깅용 cli 옵션 소개"
date:   2016-09-20 01:45:11 +0900
categories: Ansible
---

Ansible 에는 실행 및 Playbook 개발 도중에 테스트 용도로 유용하게 쓸 수 있는 옵션들이 있다.

이 중 몇 가지를 소개한다.

### `-C`, `--check`
실제 실행은 하지 않고, 변경사항을 미리 확인할 수 있다.

### `-D`, `--diff`
텍스트, template 결과물 등에서 변경사항을 표시한다. 실제로 실행된다.
`-C` 옵션과 궁합 좋음.

### `--list-host`
실제 실행은 하지 않고, 영향받는 host 목록을 표시한다.

### `--syntax-check`
실제 실행은 하지 않고 playbook 문법에 오류가 없는지 확인한다.

### 참고 링크
- [Ansible Check Mode](http://docs.ansible.com/ansible/playbooks_checkmode.html)
- [Ansible Tips and Tricks](http://docs.ansible.com/ansible/playbooks_intro.html#tips-and-tricks)
- [Useful command line options for ansible-playbook](https://liquidat.wordpress.com/2016/02/29/useful-options-ansible-cli/)
