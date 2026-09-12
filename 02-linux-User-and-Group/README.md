# Rocky Linux 사용자 및 그룹 관리 실습

## 실습 내용

Rocky Linux에서 사용자와 그룹을 생성하고
사용자 정보 및 그룹 소속을 관리하는 방법을 실습

## 학습 명령어

| 명령어 | 용도 | 사용 예시 | 설명 |
|---|---|---|---|
| `useradd` | 사용자 생성 | `useradd user1` | 새로운 사용자 생성 |
| `usermod` | 사용자 수정 | `usermod -aG testgroup user1` | 사용자를 보조 그룹에 추가 |
| `userdel` | 사용자 삭제 | `userdel -r user1` | 사용자와 홈 디렉터리 삭제 |
| `passwd` | 비밀번호 관리 | `passwd user1` | 사용자 비밀번호 설정 |
| `groupadd` | 그룹 생성 | `groupadd testgroup` | 새로운 그룹 생성 |
| `groupdel` | 그룹 삭제 | `groupdel testgroup` | 그룹 삭제 |
| `id` | 사용자 정보 확인 | `id user1` | UID, GID 및 그룹 확인 |
| `getent` | 시스템DB 정보 조회 | `getent group testgroup` | 그룹의 정보 확인|
| `su -` | 사용자 전환 | `su - user1` | 사용자의 로그인 환경으로 전환 |

## 주요 시스템 파일

| 파일 | 역할 |
|---|---|
| `/etc/passwd` | 사용자 기본 정보 |
| `/etc/shadow` | 비밀번호 해시 및 만료 정보 |
| `/etc/group` | 그룹 정보 |
| `/etc/skel` | 신규 사용자에게 제공되는 기본 파일 

## 실습

### 1. 사용자 생성
```bash
[root@localhost ~]# useradd user1
[root@localhost ~]# id user1
uid=1001(user1) gid=1002(user1) groups=1002(user1)
```

`user1` 사용자를 생성하고 `id`를 이용하여 UID, GID 및 그룹 정보를 확인

---

### 2. 그룹 생성

```bash
[root@localhost ~]# groupadd testgroup
[root@localhost ~]# getent group testgroup
testgroup:x:1001:

```

`testgroup` 그룹을 생성하고 그룹 정보를 확인

---

### 3. 비밀번호 설정

```bash
[root@localhost ~]# passwd user1
user1 사용자의 비밀 번호 변경 중
새 암호:
새 암호 재입력:
passwd: 모든 인증 토큰이 성공적으로 업데이트 되었습니다.

```

`user1` 사용자의 비밀 번호 설정

---

### 4. 사용자를 그룹에 추가

```bash

[root@localhost ~]# usermod -aG testgroup user1
[root@localhost ~]# id user1
uid=1001(user1) gid=1002(user1) groups=1002(user1),1001(testgroup)

```

`user1`을 `testgroup` 그룹에 추가

---

### 5. 사용자 전환

```bash
[root@localhost ~]# su - user1
[user1@localhost ~]$ whoami
user1
[user1@localhost ~]$ pwd
/home/user1
```

`su - user1`을 이용하여 `user1`의 로그인 환경으로 전환

---

### 6. 계정 및 그룹 정보 확인

```bash
[root@localhost ~]# grep 'user1:' /etc/passwd
user1:x:1001:1002::/home/user1:/bin/bash
[root@localhost ~]# grep 'testgroup:' /etc/group
testgroup:x:1001:user1
```

`/etc/passwd`에서 계정명, UID, GID, 홈 디렉터리, 로그인 Shell 정보 확인
`/etc/group`에서 그룹 이름, GID, 등록된 사용자 정보 확인

---

### 7. 계정 및 그룹 삭제

```bash
[root@localhost ~]# userdel user1
[root@localhost ~]# groupdel testgroup
[root@localhost ~]# id user1
id: `user1': no such user
[root@localhost ~]# getent group testgroup
```

`userdel`과 `groupdel`를 이용하여 `user1`과 `testgroup` 삭제

## 실습 결과
 
 - `useradd`를 이용하여 새로운 사용자 계정 생성
 - `groupadd`를 이용하여 새로운 그룹 생성
 - `passwd`를 이용하여 사용자의 비밀번호 설정
 - `usermod -aG`를 이용하여 기존 그룹을 유지하면서 보조 그룹에 사용자 추가
 - `id`를 이용하여 사용자의 UID, GID, 그룹 소속 정보 확인
 - `su -`를 이용하여 다른 사용자의 로그인 환경 전환
 - `grep`을 이용하여`/etc/passwd`와 `/etc/group` 안에 있는 정보 확인
 - `userdel`과 `groupdel`를 이용하여 사용자와 그룹 삭제
