# 4. 작업 제어

</br>

`echo $?` : 명령어는 가장 최근에 실행된 명령어의 종료 상태(exit status)를 출력

결과값=> 0일 때 : 정상 종료 되었음을 의미

            => 0이 아닌 수 : 명령어 실행이 실패했음을 의미

</br>

*멱등성이 적용되지 않는 모듈

`shell` `command` `raw` 

</br>

---

1. 반복문

2. 조건문, register 변수

3. 변경된 작업의 후속 작업 - handler

4. 작업 오류 제어

---

# 1. 반복문

```
# loop2.yaml
---
- hosts: all
  vars:
    mail_services:
      - postfix
      - dovecot
  tasks:
    - name: Postfix and Dovecot are running
      service:
        name: "{{ item }}"
        state: started
      loop: "{{ mail_services }}"
```

**`tasks`**:

    `service`:

        **`name: "{{ item }}"`**: `item`은 `loop`에서 반복할 때의 현재 항목을 나타냄. 

                                            여기서는 각 서비스의 이름

        **`loop`**: `mail_services` 목록에 포함된 각 서비스에 대해 작업을 반복하여 실행

</br>

사전 목록에 대한 반복문

```
# loop3.yaml
---
- hosts: all
  tasks:
    - name: Users exist and are in the correct groups
      user:
        name: "{{ item['name'] }}"
        state: present
        groups: "{{ item['groups'] }}"
      loop:
        - name: jane
          groups: wheel
        - name: joe
          groups: root
```

**`tasks`**:

    `user` :

        **`name: "{{ item['name'] }}"`**: `item`은 `loop`에서 현재 반복되는 항목을 나타냄.

                                                                 여기서는 사용자의 이름을 의미.

        **`state: present`**: 사용자가 존재해야 함을 의미. 사용자가 없으면 생성됨.

        **`groups: "{{ item['groups'] }}"`**: 현재 사용자를 지정된 그룹에 추가.

    **`loop`**: 두 명의 사용자를 정의. 

                 첫 번째: `jane`이고 `wheel` 그룹에 속함.

                 두 번째: `joe`이고 `root` 그룹에 속함.

</br>

---

# 2. 조건문

| 연산자            | 의미        | 예시                                                 |
| -------------- | --------- | -------------------------------------------------- |
| is defined     | 변수가 있음    | min_memory is defined                              |
| in not defined | 변수가 없음    | min_memory is not defined                          |
| not            | 거짓일 때 실행  | not memory_available                               |
| in             | 목록의 값에 포함 | ansible_facts['distribution'] in supported_distros |

```
- yum:
    name: mariadb-server
    state: latest
  loop: "{{ ansible_facts['mounts'] }}"
  when: item['mount'] == "/" and item['size_available'] > 300000000
```

**`yum`**:

        **`name: mariadb-server`**: 설치할 패키지 이름을 지정합니다.

        **`state: latest`**: 패키지가 최신 버전으로 설치되도록 합니다.

**`loop`**: `ansible_facts['mounts']` 리스트를 반복하여 각 마운트된 파일 시스템 정보를             `item`으로 참조

**`when`**:

        **`item['mount'] == "/"`**: 루트 파일 시스템(`/`)인지 확인

        **`item['size_available'] > 300000000`**: 

            사용 가능한 공간이 300MB 이상인 경우에만 이 태스크를 실행

        위 조건이 모두 충족되는 경우에만 `mariadb-server` 패키지를 설치

</br>

#### Register 변수

어떤 모듈이 실행되며 발생되는 이벤트를 캡쳐한 특별한 변수

debug 같은 모듈로 호출해서 내용을 살펴보거나 혹은 변수의 값을 통해 판단하는 when 조건을 설정할 수 있다.

```
# new-register.yaml
---
- name: Loop Register Test
  gather_facts: no
  hosts: seoul
  tasks:
    - name: Looping Echo Task
      ansible.builtin.shell: "echo This is my item: {{ item }}"
      loop:
        - one
        - two
      register: echo_results

    - name: Show stdout from the previous task.
      ansible.builtin.debug:
        msg: "STDOUT from previous task: {{ item['stdout'] }}"
      loop: "{{ echo_results['results'] }}"
```

**`gather_facts: no`** : 기본적으로 Ansible은 실행 전 호스트의 정보를 수집함

                                      이 옵션을 `no`로 설정하면 해당 작업을 건너뛴다.

**`register: echo_results`**: 이 작업의 결과를 `echo_results`라는 변수에 저장함

                         여기에는 각 반복 작업의 표준 출력(stdout) 및 기타 메타데이터를 포함한다.

**`ansible.builtin.debug`**:  앞서 등록된 `echo_results` 변수의 내용을 출력함

        **`msg`**: 이전 작업에서 생성된 표준 출력(stdout)을 `msg`를 통해 출력

</br>

```
# when2.yaml
---
- name: Test Variable is Defined Demo
  hosts: all
  vars:
    my_service: httpd
  tasks:
    - name: "{{ my_service }} package is installed"
      yum:
        name: "{{ my_service }}"
      when: my_service is defined
```

**`vars`**: 플레이북 내에서 사용할 변수를 정의

             여기서는 `my_service`라는 변수에 `httpd`라는 값을 할당

**`yum`** : 패키지를 관리하는 데 사용

            이 모듈을 사용하여 지정된 패키지를 설치, 제거 또는 업데이트할 수 있다.

    **`name: "{{ my_service }}"`**: `my_service` 변수의 값 (`httpd`)이 패키지 이름으로 사용

                                                        이 경우, `httpd` 패키지를 설치한다.

**`when: my_service is defined`**: `my_service` 변수가 정의되어 있을 때만 태스크를 실행

</br>

---

# 3. 변경된 작업의 후속 작업 - Handler

- 핸들러(handler): 다른 작업에서 트리거한 알림에 반응하는 작업

- 관리 호스트에서 작업이 변경될 때만 핸들러에 알림을 통해서 통지

- 각 핸들러는 플레이의 작업 블록 다음에 해당 이름으로 트리거됨

- 작업에서 이름을 사용하여 핸들러에 알리지 않으면 핸들러가 실행되지 않음

- 하나 이상의 작업에서 핸들러에 알리면 플레이의 다른 모든 작업이 완료된 후 핸들러가 한 번 실행

- 핸들러는 작업이므로 관리자가 다른 작업에 사용하려는 핸들러에 같은 모듈을 사용할 수 있음

- 일반적으로 핸들러는 호스트를 재부팅하고 서비스를 다시 시작하는 데 사용

- notify 문을 사용하여 명시적으로 호출된 경우에만 트리거되는 비활성 작업

</br>

```
- tasks:
    - name: notify tasks
      template:
        src: /var/lib/templates/demo.conf
        dest: /etc/httpd/conf.d/demo-httpd.conf
      notify: 
        - restart mysql
        - restart apache
handlers:
  - name: restart mysql
    service:
      name: mariadb
      state: restarted
  - name: restart apache
    service:
      name: httpd
      state: restarted
```

**`template`**:

        **`src: /var/lib/templates/demo.conf`**: 템플릿 파일의 원본 경로

        **`dest: /etc/httpd/conf.d/demo-httpd.conf`**: 템플릿 파일이 복사될 대상 경로

**`notify`**: 이 작업이 변경되면(즉, 템플릿 파일이 새로 복사되거나 업데이트되면) 

                정의된 핸들러(`restart mysql` 및 `restart apache`)를 호출 

**`handlers`**: 특정 작업이 변경될 때 실행되는 작업을 정의

        **`name: restart mysql`**:  `mariadb` 서비스를 재시작

        **`name: restart apache`**: `httpd` 서비스를 재시작

  </br>

- 핸들러는 항상 플레이의 handlers 섹션에서 지정한 순서대로 실행된다.
  
   작업의 notify 문에 의해 나열된 순서나 작업이 핸들러에 알리는 순서대로 실행 X

- 핸들러는 일반적으로 플레이에서 다른 모든 작업이 완료된 후에 실행된다. 
  
  플레이북 중 tasks 부분의 작업에서 호출한 핸들러는 tasks 아래에 있는 모든 작업이 처리될 때까지 실행 X

- 핸들러 이름은 플레이별 네임스페이스에 있다. 
  
  두 핸들러에 같은 이름이 잘못 지정되면 하나의 핸들러만 실행됨

- 둘 이상의 작업에서 하나의 핸들러에 알리는 경우 해당 핸들러가 한 번 실행된다. 
  
  핸들러에 알리는 작업이 없는 경우 핸들러 실행 X

- notify 문이 포함된 작업에서 changed 결과를 보고하지 않으면 핸들러에 알리지 X

</br>

changed_when 속성으로 조건을 설정하여 특정 조건에서 알림이 발생하도록 구성

```
tasks:
  - shell:
      cmd: /usr/local/bin/upgrade-database
    register: command_result
    changed_when: "'Success' in command_result.stdout"
    notify: 
      - restart_database
```

</br>

강제로 false 값을 입력하여 항상 changed를 보고 X

```
- name: validate config
  command: httpd -t
  changed_when: false
```

</br>

---

# 4. 작업 오류 제어

각 작업의 반환 코드를 평가하여 작업의 성공 여부를 판단

일반적으로 작업이 실패하면 Ansible은 바로 이후의 모든 작업을 건너뜀

</br>

작업이 실패해도 플레이를 계속 실행하는 방법 => `ignore_errors`

```
- yum:
    name: http
    state: latest
  ignore_errors: yes
```

</br>

특정 작업에서 에러가 발생-> 플레이 중단-> 이전에 알림을 받았던 핸들러는 모두 동작 X

이때 플레이는 중단되더라도 알림을 받은 핸들러는 모두 동작하도록=> `force_handlers` 

```
- hosts: all
  force_handlers: yes
  tasks:
....(생략)....
```

</br>

원하는 결과가 나오지 않더라도 작업에서 정상적으로 넘어가는 것 처럼 보이는 경우

작업은 성공되더라도 에러가 발생되게 강제화=> `failed_when`,  `fail`

```
tasks:
  - name: run script
    shell: /usr/local/bin/create_users.sh
    register: command_result
    failed_when: "'password missing' in command_result.stdout"
```

```
tasks:
  - name: run script
    shell: /usr/local/bin/create_users.sh
    register: command_result
    ignore_errors: yes
  - name: fail message
    fail: 
      msg: "Passowrd missing"
    when: "'password missing' in command_result.stdout"
```

</br>

`block` : 작업을 논리적인 그룹으로 만들수 있음. 연관된 작업들을 묶어 관리

`when` : 여러 작업에 조건을 적용

```
tasks:
  - name: install and configuration versionlock
    block: 
    - name: package install 
      yum: 
        name: python3-dnf-plugin-versionlock
        state: present
    - name: config lock
        lineinfile:
          dest: /etc/yum/pluginconf.d/versionlock.list
          line: tzdata-2016j-1
          state: present
    when: ansible_distribution == "RedHat"
```

`lineinfile` : 특정 패키지 버전을 잠금 리스트에 추가하여 

                         시스템 업데이트 시 해당 패키지 버전이 고정되도록 설정

</br>

`rescue` : 오류에 대한 작업 내용을 정의. 블록 내의 작업이 실패했을 때 실행

`always` : 실패 유무와는 상관없이 항상 실행

```
tasks:
  - name: Upgrade DB
    block:
      - name: upgrade the database
        shell:
          cmd: /usr/local/lib/upgrade-database
    rescue:
      - name: revert the database upgrade
        shell:
          cmd: /usr/local/lib/revert-database
    always:
      - name: always restart the database
        service:
          name: mariadb
          state: restarted
```

**`shell` **: 쉘 명령을 실행하는 데 사용

**`cmd`**: 실행할 명령어를 지정

**`state: restarted`**: 서비스를 재시작
