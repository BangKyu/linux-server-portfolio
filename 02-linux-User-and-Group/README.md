# Linux 사용자 및 그룹 관리 실습

## 실습 개요

Rocky Linux에서 사용자와 그룹을 생성하고 관리하는 방법을 실습하였다.

사용자의 UID, GID, Primary Group, Supplementary Group을 확인하고 `/etc/passwd`, `/etc/group`, `/etc/shadow`를 통해 계정 정보를 확인하는 방법을 실습하였다.

또한 `su -`를 이용한 사용자 전환과 사용자 및 그룹 삭제하는 방법을 실습하였다.

---

## 실습 과정

### 1. 사용자 및 그룹 존재 여부 확인

실습 시작 전 사용할 사용자와 그룹의 존재 여부 확인

```bash
[root@Server-A ~]# grep '^root:' /etc/passwd
root:x:0:0:root:/root:/bin/bash

[root@Server-A ~]# grep '^root:' /etc/group
root:x:0:

[root@Server-A ~]# getent passwd user1

[root@Server-A ~]# getent passwd user2

[root@Server-A ~]# getent group developers
```

`grep '^root:'`를 사용하여 `/etc/passwd`와 `/etc/group`에서 `root` 계정 및 그룹 정보 확인

`getent passwd`와 `getent group`을 사용하여 `user1`, `user2`, `developers`의 존재 여부 확인

출력이 없으므로 해당 사용자와 그룹이 존재하지 않는 것을 확인

---

### 2. 그룹 및 사용자 생성

```bash
[root@Server-A ~]# groupadd developers
[root@Server-A ~]# useradd user1
[root@Server-A ~]# passwd user1
user1 사용자의 비밀 번호 변경 중
새 암호:
새 암호 재입력:
passwd: 모든 인증 토큰이 성공적으로 업데이트 되었습니다.

[root@Server-A ~]# usermod -aG developers user1

[root@Server-A ~]# id user1
uid=1001(user1) gid=1002(user1) groups=1002(user1),1001(developers)
```

`groupadd`를 사용하여 `developers` 그룹 생성

`useradd`를 사용하여 `user1` 사용자 생성

`passwd`를 사용하여 `user1`의 비밀번호 설정

`usermod -aG`를 사용하여 `user1`을 `developers` Supplementary Group에 추가

`id user1`을 사용하여 UID, Primary Group, Supplementary Group 확인

---

### 3. 사용자 및 그룹 정보 확인

```bash
[root@Server-A ~]# grep '^user1:' /etc/passwd
user1:x:1001:1002::/home/user1:/bin/bash

[root@Server-A ~]# grep '^user1:' /etc/group
user1:x:1002:

[root@Server-A ~]# getent group developers
developers:x:1001:user1
```

`/etc/passwd`를 통해 `user1`의 UID, Primary GID, Home Directory, Login Shell 확인

`/etc/group`과 `getent group`을 통해 `user1` 그룹과 `developers` 그룹 정보 확인

`developers` 그룹의 Supplementary Group 구성원으로 `user1`이 등록된 것을 확인

---

### 4. Password 정보 및 Home Directory 확인

```bash
[root@Server-A ~]# grep '^user1:' /etc/shadow
user1:<PASSWORD_HASH>:20712:0:99999:7:::

[root@Server-A ~]# ls -l /home
합계 4
drwx------. 14 guest guest 4096  9월 12 14:20 guest
drwx------.  3 user1 user1   78  9월 16 12:50 user1
```

`/etc/shadow`를 통해 `user1`의 Password Hash 및 Password 관련 정보가 저장되어 있는 것을 확인

`ls -l /home`을 사용하여 `/home/user1` Home Directory 생성 및 소유자 확인

> `/etc/shadow`의 실제 Password Hash는 보안상 README에 기록하지 않음

---

### 5. 사용자 전환

```bash
[root@Server-A ~]# su - user1

[user1@Server-A ~]$ whoami
user1

[user1@Server-A ~]$ pwd
/home/user1

[user1@Server-A ~]$ id
uid=1001(user1) gid=1002(user1) groups=1002(user1),1001(developers) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

`su - user1`을 사용하여 `user1`의 로그인 환경으로 사용자 전환

`whoami`를 사용하여 현재 사용자가 `user1`인지 확인

`pwd`를 사용하여 현재 작업 디렉터리가 `/home/user1`인지 확인

`id`를 사용하여 UID, Primary Group, Supplementary Group 확인

---

### 6. Primary Group을 지정한 사용자 생성

```bash
[root@Server-A ~]# useradd -g developers user2

[root@Server-A ~]# passwd user2
새 암호:
새 암호 재입력:
passwd: 모든 인증 토큰이 성공적으로 업데이트 되었습니다.

[root@Server-A ~]# id user2
uid=1002(user2) gid=1001(developers) groups=1001(developers)

[root@Server-A ~]# grep '^user2:' /etc/passwd
user2:x:1002:1001::/home/user2:/bin/bash

[root@Server-A ~]# getent group developers
developers:x:1001:user1
```

`useradd -g developers`를 사용하여 `developers`를 Primary Group으로 지정한 `user2` 생성

`passwd`를 사용하여 `user2`의 비밀번호 설정

`id user2`를 사용하여 `developers`가 Primary Group으로 지정된 것을 확인

`/etc/passwd`에서 `user2`의 Primary GID가 `developers`의 GID인 `1001`로 설정된 것을 확인

`user2`는 `developers`를 Primary Group으로 사용하므로 `getent group developers`의 Supplementary Member 목록에는 표시되지 않는 것을 확인

---

### 7. user2 로그인 환경 확인

```bash
[root@Server-A ~]# su - user2

[user2@Server-A ~]$ whoami
user2

[user2@Server-A ~]$ pwd
/home/user2

[user2@Server-A ~]$ id
uid=1002(user2) gid=1001(developers) groups=1001(developers) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

`su - user2`를 사용하여 `user2`의 로그인 환경으로 사용자 전환

`whoami`를 사용하여 현재 사용자가 `user2`인지 확인

`pwd`를 사용하여 Home Directory가 `/home/user2`인지 확인

`id`를 사용하여 `developers`가 Primary Group으로 설정된 것을 확인

---

### 8. 사용자 및 그룹 삭제

```bash
[root@Server-A ~]# userdel -r user1
[root@Server-A ~]# userdel -r user2

[root@Server-A ~]# groupdel developers
```

`userdel -r`을 사용하여 `user1`, `user2` 계정과 Home Directory를 함께 삭제

`groupdel`을 사용하여 `developers` 그룹 삭제

```bash
[root@Server-A ~]# grep '^user1:' /etc/passwd

[root@Server-A ~]# grep '^user2:' /etc/passwd

[root@Server-A ~]# getent group developers

[root@Server-A ~]# ls -l /home
합계 4
drwx------. 14 guest guest 4096  9월 12 14:20 guest
```

사용자 및 그룹 조회 결과가 출력되지 않는 것을 통해 삭제 여부 확인

`/home` 디렉터리를 확인하여 `user1`, `user2`의 Home Directory가 삭제된 것을 확인

---

## 실습 결과

- `groupadd`, `useradd`를 사용하여 그룹과 사용자 생성
- `passwd`를 사용하여 사용자 비밀번호 설정
- `usermod -aG`를 사용하여 사용자를 Supplementary Group에 추가
- `useradd -g`를 사용하여 특정 그룹을 Primary Group으로 지정
- `id`를 사용하여 UID, GID 및 그룹 소속 정보 확인
- `/etc/passwd`, `/etc/group`, `/etc/shadow`를 통해 계정 정보 확인
- `su -`를 사용하여 사용자 로그인 환경으로 전환
- Primary Group과 Supplementary Group의 차이 확인
- `userdel -r`을 사용하여 사용자와 Home Directory 삭제
- `groupdel`을 사용하여 그룹 삭제
