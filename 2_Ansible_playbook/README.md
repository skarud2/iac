# 2. 플레이북 시행

</br>

Ansible 파일 유형

1. 구성파일

2. 인벤토리 파일

3. 플레이북 파일

</br>

---

### 1. 구성파일

각 섹션에 키-값 

일반사용자로 접근 -> root로 권한 상승

(공통으로 사용할 계정 생성 -> 무조건 ssh 연결 가능)

(root로 직접 접근X, 한 번 거쳐서 접근하기)

password 물어보지 않게 -> 자동화 위해

</br>

`useradd ansible-user`

새로운 사용자 `ansible-user`를 추가

</br>

`ssh serverA 'useradd ansible-user'`

`ssh serverB 'useradd ansible-user'`

`serverA`에 접속하여 `ansible-user`라는 사용자를 추가하고 홈 디렉토리 및 셸을 설정

</br>

`vi /etc/sudoers`

![](C:\Users\KDP\AppData\Roaming\marktext\images\2024-07-30-15-44-47-image.png)

ansible-user 추가로 작성

루트 권한으로 실행

</br>

`su - ansible-user`

: 현재 사용자 세션에서 `ansible-user` 사용자로 전환하는 데 사용

![](C:\Users\KDP\AppData\Roaming\marktext\images\2024-07-30-16-01-50-image.png)

`-` : 로그인 셸을 시작하여 해당 사용자의 환경 설정 파일을 로드 

이는 새 사용자 세션을 시작하고, 그 사용자의 홈 디렉토리 및 환경 변수를 설정하는 데 유용

`exit` : 로그아웃

</br>

`passwd ansible-user`

`ansible-user` 사용자의 비밀번호를 설정하거나 변경

</br>

`cat /etc/hosts`

시스템의 호스트 이름과 IP 주소 매핑 정보를 담고 있는 `/etc/hosts` 파일의 **내용**을 출력

![](C:\Users\KDP\AppData\Roaming\marktext\images\2024-07-30-15-54-58-image.png)

</br>

</br>

---

### 2. 인벤토리 파일 작성

인벤토리 파일: 앤서블에서 관리할 호스트 목록 정의

1. 텍스트 파일 -> 정적 호스트 인벤토리 정의

2. 외부 정보 프로바이더

</br>

##### 정적 인벤토리

텍스트 파일

앤서블이 관리할 호스트를 목록으로 지정

호스트 명 혹은 IP주소를 한줄에 입력

```
web1.test.com
web2.test.com
db1.test.com
db2.test.com
192.168.0.100
```

호스트 그룹을 지정

대괄호[ ]를 사용하여 이름을 지정한 뒤 다음행부터 한줄씩 그룹의 구성원 호스트들을 나열

```
192.168.0.100

[web-servers]
web1.test.com
web2.test.com

[db-servers]
db1.test.com
db2.test.com
```

```
192.168.0.100 
//다음 대괄호가 나올 때까지 빈칸은 무시됨
//web-servers 4개가 됨

[web-servers]
web1.test.com
web2.test.com 


db1.test.com
db2.test.com 

[db-servers]
db1.test.com
db2.test.com
```

</br>

`all` : 모든 호스트 목록을 포함하는 그룹

`ungrouped` : 인벤토리에서 그룹에 속하지 않는 모든 호스트 목록

</br>

</br>

`vi inventory`

![](C:\Users\KDP\AppData\Roaming\marktext\images\2024-07-30-16-30-17-image.png)

`ansible -i inventory -m ping all`

- **`-i inventory`**: `inventory` 파일을 인벤토리로 지정
  
  인벤토리 파일은 Ansible이 관리할 호스트(서버)의 목록을 포함

- **`-m ping`**: `ping` 모듈을 사용하여 각 호스트에 대해 ping 명령어를 실행
  
   이 ping은 ICMP 네트워크 핑이 아니라, Ansible이 대상 호스트에 접근 가능하고 Python이 설치되어 있는지 확인하는 간단한 체크

- **`all`**: 인벤토리에 나열된 모든 호스트를 대상으로 명령어 실행

이 명령어를 실행하면 각 호스트에 대해 ping을 시도하고, 연결이 성공하면 "pong" 메시지를 반환. 이 과정을 통해 Ansible이 각 호스트에 접근할 수 있는지, Ansible의 환경이 올바르게 구성되었는지를 확인할 수 있다.

![](C:\Users\KDP\AppData\Roaming\marktext\images\2024-07-30-16-31-17-image.png)

`ansible -i inventory -m ping webservers`

![](C:\Users\KDP\AppData\Roaming\marktext\images\2024-07-30-16-32-06-image.png)

</br>

</br>

##### 범위 지정

`[처음값:마지막값]`

- 192.168.[4:7].[0:255] → 192.168.4.0 ~ 192.168.7.255 범위의 모든 IP 주소

- server[01:20].example.com → server01.example.com 부터 server20.example.com 까지 모든 호스트

- [a:e].dns.example.com → a.dns.example.com 부터 e.dns.example.com 까지의 호스트

</br>

`ansible -i ./inventory`

`-i` 인벤토리 파일의 기본위치가 아닌 사용자가 원하는 파일로 대체하는 옵션

---

### 3. 플레이북 파일

- YAML 포맷으로 작성된 텍스트 파일

- 특정한 작업을 수행하기 위해 플레이북 파일에 작업내용을 미리 정의

- 단일 혹은 여러개의 플레이가 작성 될 수 있음

- 단일 플레이에는 작업 대상이 되는 관리 호스트 정보와 작업 내용을 담고 있는 정보가 필요

- `hosts`: 작업대상

- `tasks`: 작업내용

- `---`: 시작 마커

- `...`: 종료 마커 (보통 생략)

</br>

`name: test play`

name 키는 플레이의 목적 및 목표를 나타내는 정보이며 임의의 문자열을 입력할 수 있음

(필수 항목은 아님)

`hosts: web1.test.com`

실행할 호스트를 선택

hosts에 입력된 값은 인벤토리에 포함되어 있는 목록을 선택할 수 있음

```
  tasks: 
    - name: create new user
      user:
        name: user01
        uid: 4000
        state: present
```

실제로 수행할 작업 목록을 구성

</br>

##### 플레이북 실행

`ansible-playbook 파일이름.확장자`: 제어노드에서 플레이북 파일의 내용을 읽어 플레이를 실행하는 명령어

* Ansible 플레이북의 작업은 멱등

* 플레이북을 여러 번 실행하는 것이 안전

![](C:\Users\KDP\AppData\Roaming\marktext\images\2024-08-04-14-35-10-스크린샷%202024-07-31%20105606.png)
