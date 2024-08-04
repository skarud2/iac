# 3. 변수 관리

```
hosts:
...
vars:
vars_files:
roles:
...
tasks:
```

- 변수는 여러 위치에 정의할 수 있음

- `--extra-vars` 또는`-e`옵션을 사용해서 ansible-playbook 명령어의 인자로 전달할수 있음

- 동일한 이름의 서로 다른 위치에서 사용되는 변수는 우선순위에 따라 사용 값이 결정됨

```
변수 우선순위 확인 문서
https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable
```

</br>

---

1. 플레이북 변수

2. 인벤토리 변수

3. 변수 파일 암호화

4. 팩트 변수
   
   </br>

---

## 1. 플레이북 변수

`vars` : 플레이에서 변수를 선언할 때 사용

```
- hosts: all
  vars: 
    user: user01
    home: /home/user01
```

`user`라는 변수에 `user01`이라는 값이 할당

</br>

`vars_files` : 외부의 변수 파일을 호출해 변수를 읽음

```
# user.yaml
user: user01
home: /home/user01
```

```
- hosts: all
  vars_files:
    - user.yaml
```

</br>

이중 중괄호를 사용해 변수명을 호출 가능

***시작을 변수 값으로 입력할 경우 => 따옴표를 반드시 붙여 줘야함 (문자열로 처리 위해)**

```
vars:
  user: user01

tasks:
  - name: create user {{ user }}
    user:
      name: "{{ user }}"
```

</br>

</br>

---

## 2. 인벤토리 변수

- 호스트 변수 : 특정 호스트에 적용

- 그룹 호스트 : 중첩 그룹 그리고 모든 호스트에 적용되는 그룹 변수도 포함

- 우선순위 : 플레이북에서 정의한 변수 > 호스트 변수 > 그룹 변수

</br>

```
[servers1]
test1.example.com
test2.example.com

[servers2]
test3.example.com
test4.example.com

[servers:children]
servers1
servers2

[server1:vars]
user=user01

[servers:vars]
user=user02
```

- `servers1`이라는 그룹을 정의
  
  - 이 그룹에는 `test1.example.com`과 `test2.example.com`이라는 두 개의 호스트가 속함

- `servers2`라는 그룹을 정의
  
  - 이 그룹에는 `test3.example.com`과 `test4.example.com`이라는 두 개의 호스트가 속함

- `servers`라는 그룹을 정의
  
  - `servers1`과 `servers2`라는 두 하위 그룹을 포함

- `servers1` 그룹에 속한 모든 호스트에 대해
  
  - `user` 변수를 `user01`로 설정

- `servers` 그룹에 속한 모든 호스트에 대해
  
  - `user` 변수를 `user02`로 설정
  
  - `servers1` 그룹의 호스트(`test1.example.com`과 `test2.example.com`)는 `servers` 그룹에도 포함되지만, `servers1:vars`에서 설정한 변수가 더 우선하기 때문에 여전히 `user01`로 설정된다.

- `children`은 그룹 내에 하위 그룹(서브그룹)을 정의

- `vars`는 특정 그룹 또는 호스트에 대한 변수를 정의

</br>

변수만 따로 분리하여 외부에 위치 시킬수 있다

작업 디렉토리 내부에 group_vars 및 host_vars 라는 두 디렉토리를 생성하여 정의

```
ansible_project/
├── inventories/
│   ├── hosts
│   ├── group_vars/
│   │   └── all.yml
│   └── host_vars/
│       └── webserver1.yml
├── playbooks/
│   └── site.yml
└── ansible.cfg
```

```
# inventories/hosts
# 서버의 호스트와 그룹을 정의

[webservers]
webserver1

[dbservers]
dbserver1
```

```
# inventories/group_vars/all.yml
# 모든 그룹에 적용할 변수를 정의

common_packages:
  - vim
  - curl
  - git
```

```
# inventories/host_vars/webserver1.yml
# 특정 호스트에 적용할 변수를 정의

nginx_version: "1.18.0"
```

</br>

---

## 3. 변수 파일 암호화 (Ansible Vault)

- 암호파일 생성
  
  ```
  ansible-vault create secret.yaml
  ```

- 파일 확인
  
  ```
  ansible-vault view secret.yaml
  ```

- 파일 편집
  
  ```
  ansible-vault edit secret.yaml
  ```

- 기존 파일을 암호화
  
  ```
  ansible-vault encrypt webserver.yaml 
  ```

- 암호 변경
  
  ```
  ansible-vault rekey secret.yaml 
  ```

- 암호화 파일을 복호화
  
  ```
  ansible-vault decrypt webserver.yaml 
  ```

</br>

###### 플레이북 실행에 암호화된 파일 사용

- 암호 저장 파일 생성
  
  ```
  echo 'Test123!' > vault-pass
  ```

- `group_vars/seoul` 파일을 암호화하는 과정에서, 암호화에 사용할 암호를 `vault-pass` 파일에서 가져오도록 설정하는 명령어
  
  ```
  ansible-vault encrypt --vault-password-file=vault-pass group_vars/seoul
  ```

- 암호화된 변수 파일을 포함하는 Ansible 플레이북을 실행할 때 사용하는 명령어
  
  `vault-pass` 파일에 저장된 암호를 사용하여 플레이북 실행 시 암호화된 파일을 자동으로 복호화
  
  ```
  ansible-playbook --vault-password-file=vault-pass var.yaml
  ```

- 암호화된 파일을 복호화(암호 해제)하는 명령어
  
  암호화된 파일을 원래의 평문 텍스트 파일로 되돌릴 수 있다.
  
  ```
  ansible-vault decrypt --vault-password-file=vault-pass group_vars/seoul
  ```

</br>

---

## 4. 팩트 변수 (Ansible Fact)

관리 호스트에서 자동으로 검색한 호스트별 정보가 포함되어 있는 변수

- 호스트 이름
- 커널 버전
- 네트워크 인터페이스 이름
- 네트워크 인터페이스 IP 주소
- 운영 체제 버전
- CPU 개수
- 사용 가능한 메모리
- 스토리지 장치의 크기 및 여유 공간

</br>

##### 팩트 변수 비활성화

```
# disable_fact.yaml
---
- name: This play gathers no facts automatically
    hosts: webservers
    gather_facts: no    # 팩트 수집 중지
    tasks:
  - name: Manually gather facts
        setup:             # 수동으로 수집
```
