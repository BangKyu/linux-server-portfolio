# Rocky Linux 파일 권한 관리 실습

## 실습 내용

Rocky Linux에서 파일과 디렉터리의 권한 및 소유권을 확인하고,
사용자별 접근 권한을 설정하는 방법을 실습하였다.

## 학습 내용

- 파일 및 디렉터리의 권한 구조
- Owner, Group, Other의 의미
- `r`, `w`, `x` 권한의 의미
- `chmod`를 이용한 권한 변경
- `chown`, `chgrp`를 이용한 소유권 변경
- `umask`를 이용한 기본 권한 설정
- Linux 특수 권한
  - Set-UID
  - Set-GID
  - Sticky Bit

## 학습 명령어

| 명령어 | 용도 | 사용 예시 | 설명 |
|---|---|---|---|
| `chmod` | 권한 변경 | `chmod 755 test.sh` | 파일 또는 디렉터리의 `r`, `w`, `x` 권한을 변경 |
| `chown` | 소유권 변경 | `chown user1:testgroup test.txt` | 파일 또는 디렉터리의 소유자와 소유 그룹을 변경 |
| `chgrp` | 소유 그룹 변경 | `chgrp testgroup test.txt` | 파일 또는 디렉터리의 소유 그룹을 변경 |
| `umask` | 기본 권한 설정 | `umask 0022` | 새로 생성되는 파일과 디렉터리의 기본 권한을 제한하는 마스크를 설정 |
| `ls -l` | 권한 및 소유권 확인 | `ls -l test.txt` | 파일의 권한, 소유자, 소유 그룹 등의 정보를 확인 |
| `ls -ld` | 디렉터리 자체 정보 확인 | `ls -ld testdir` | 디렉터리 내부 목록이 아닌 디렉터리 자체의 권한과 소유권을 확인 |
| `chmod 1777` | Sticky Bit 설정 | `chmod 1777 /test/share` | 공유 디렉터리에서 다른 사용자의 파일 삭제를 제한 |
| `chmod 2770` | Set-GID 설정 | `chmod 2770 /test/project` | 새 파일과 디렉터리가 부모 디렉터리의 그룹을 상속하도록 설정 |
---

## 실습

### 1. 기본 파일 및 디렉터리 권한 확인

```bash
[root@localhost ~]# mkdir -p /test/permission
[root@localhost ~]# cd /test/permission/
[root@localhost permission]# touch test.txt
[root@localhost permission]# mkdir testdir
[root@localhost permission]# ls -ld test.txt testdir
-rw-r--r--. 1 root root 0  9월 12 23:28 test.txt
drwxr-xr-x. 2 root root 6  9월 12 23:29 testdir
```

`ls -ld`를 이용하여 파일과 디렉터리 기본 권한 확인
파일은 644, 디렉터리는 755 권한으로 생성

---

### 2. chmod를 이용한 파일 권한 변경

```bash
[root@localhost permission]# chmod 600 test.txt
[root@localhost permission]# ls -l test.txt
-rw-------. 1 root root 0  9월 12 23:28 test.txt
[root@localhost permission]# chmod 640 test.txt
[root@localhost permission]# ls -l test.txt
-rw-r-----. 1 root root 0  9월 12 23:28 test.txt
[root@localhost permission]# chmod 754 test.txt
[root@localhost permission]# ls -l test.txt
-rwxr-xr--. 1 root root 0  9월 12 23:28 test.txt
```

`chmod`를 이용하여 파일 권한 변경
`600` : 소유자만 읽기, 쓰기 가능
`640` : 소유자 읽기/쓰기, 그룹 읽기 가능
`754` : 소유자 읽기/쓰기/실행, 그룹 읽기/실행, 기타 사용자 읽기 가능

---

### 3. `chmod`를 이용한 기호 방식 권한 변경

```bash
[root@localhost permission]# chmod u-x test.txt
[root@localhost permission]# ls -l test.txt
-rw-r-xr--. 1 root root 0  9월 12 23:28 test.txt
[root@localhost permission]# chmod g+w test.txt
[root@localhost permission]# ls -l test.txt
-rw-rwxr--. 1 root root 0  9월 12 23:28 test.txt
[root@localhost permission]# chmod o-r test.txt
[root@localhost permission]# ls -l test.txt
-rw-rwx---. 1 root root 0  9월 12 23:28 test.txt
```

`chmod`의 기호 방식을 이용하여 사용자별 권한 추가/제거
`u-x` : 소유자의 실행 권한 제거
`g+w` : 그룹의 쓰기 권한 추가
`o-r` : 기타 사용자의 읽기 권한 제거

---

### 4. 일반 사용자의 파일 접근 권한 확인

테스트를 위한 일반 사용자 `user2` 생성
```bash
[root@localhost permission]# useradd user2
[root@localhost permission]# id user2
uid=1002(user2) gid=1002(user2) groups=1002(user2)
```

테스트 파일 내용 작성 후 권한 확인
```bash
[root@localhost permission]# echo "Permission Test" > test.txt
[root@localhost permission]# ls -l test.txt
-rw-rwx---. 1 root root 16  9월 12 23:52 test.txt
[root@localhost permission]# cat test.txt
Permission Test
```

`user2` 전환 후 파일 읽기
```bash
[root@localhost permission]# su - user2
[user2@localhost ~]$ cat /test/permission/test.txt
cat: /test/permission/test.txt: 허가 거부
```

기타 사용자 읽기 권한 추가 후 다시 확인
```bash
[root@localhost permission]# chmod o+r /test/permission/test.txt
[root@localhost permission]# ls -l /test/permission/test.txt
-rw-rwxr--. 1 root root 16  9월 12 23:52 /test/permission/test.txt
[root@localhost permission]# su - user2
[user2@localhost ~]$ cat /test/permission/test.txt
Permission Test
```

`test.txt`의 기타 사용자 권한이 `---`이면 `user2` 파일 읽기 불가능
`chmod o+r`를 사용하여 기타 사용자에게 읽기 권한 추가 후 `user2`의 파일 읽기가 가능한 것을 확인

---

### 5. `chown`, `chgrp`을 이용한 파일 소유권 변경

테스트를 위한 파일 상태 확인
```bash
[root@localhost permission]# ls -l test.txt
-rw-rwxr--. 1 root root 16  9월 12 23:52 test.txt
[root@localhost permission]# id user2
uid=1002(user2) gid=1002(user2) groups=1002(user2)
```

`chown`을 사용하여 파일 소유자를 `root`에서 `user2`로 변경
```bash
[root@localhost permission]# chown user2 test.txt
[root@localhost permission]# ls -l test.txt
-rw-rwxr--. 1 user2 root 16  9월 12 23:52 test.txt
```

`chgrp`을 사용하여 파일 소유 그룹을 `root`에서 `user2`로 변경
```bash
[root@localhost permission]# chgrp user2 test.txt
[root@localhost permission]# ls -l test.txt
-rw-rwxr--. 1 user2 user2 16  9월 12 23:52 test.txt
```

`chown`을 사용하여 소유자와 그룹을 한 번에 변경
```bash
[root@localhost permission]# chown root:root test.txt
[root@localhost permission]# ls -l test.txt
-rw-rwxr--. 1 root root 16  9월 12 23:52 test.txt
[root@localhost permission]# chown user2:user2 test.txt
[root@localhost permission]# ls -l test.txt
-rw-rwxr--. 1 user2 user2 16  9월 12 23:52 test.txt
```

`chown`은 파일의 소유자를 변경하며 `소유자:그룹` 형식을 사용하면 소유자와 소유 그룹을 동시에 변경 가능
`chgrp`은 파일의 소유 그룹만 변경 가능

---

### 6. `umask`를 이용한 기본 권한 확인

현재 `umask` 값 확인
```bash
[root@localhost permission]# umask
0022
```

새 파일과 디렉터리 생성, 기본 권한 확인
```bash
[root@localhost permission]# touch umask-file
[root@localhost permission]# mkdir umask-dir
[root@localhost permission]# ls -ld umask-file umask-dir
drwxr-xr-x. 2 root root 6  9월 13 00:17 umask-dir
-rw-r--r--. 1 root root 0  9월 13 00:17 umask-file
```
`umask` = `0022`일 경우 : `새 파일` = `644`, `새 디렉터리` = `755` 


`umask`를  `0022`에서  `0002`로 변경
```bash
[root@localhost permission]# umask 0002
[root@localhost permission]# umask
0002
[root@localhost permission]# touch umask-file2
[root@localhost permission]# mkdir umask-dir2
[root@localhost permission]# ls -ld umask-file2 umask-dir2
drwxrwxr-x. 2 root root 6  9월 13 00:22 umask-dir2
-rw-rw-r--. 1 root root 0  9월 13 00:22 umask-file2
```
`umask` = `0002`일 경우 : `새 파일` = `664`, `새 디렉터리` = `775`

---


### 7. Sticky Bit을 이용한 공유 디렉터리 권한 설정

공유 디렉터리를 생성하고 모든 사용자에게 읽기, 쓰기, 실행 권한을 부여
```bash
[root@localhost permission]# mkdir /test/share
[root@localhost permission]# chmod 777 /test/share
[root@localhost permission]# ls -ld /test/share
drwxrwxrwx. 2 root root 6  9월 13 00:24 /test/share
```

`user2`가 공유 디렉터리에 파일 생성
```bash
[root@localhost permission]# su - user2
[user2@localhost ~]$ echo "user2 file" > /test/share/user2.txt
[user2@localhost ~]$ ls -l /test/share
합계 4
-rw-r--r--. 1 user2 user2 11  9월 13 00:25 user2.txt
```

`user3`가 `user2`의 파일 삭제 시도
```bash
[root@localhost permission]# su - user3
[user3@localhost ~]$ rm /test/share/user2.txt
rm: remove write-protected 일반 파일 '/test/share/user2.txt'? yes
[user3@localhost ~]$ ls -l /test/share
합계 0
```
공유 디렉터리가 `777`인 경우 다른 사용자가 생성한 파일도 삭제할 수 있음을 확인

`Sticky Bit` 적용
```bash
[root@localhost permission]# chmod 1777 /test/share
[root@localhost permission]# ls -ld /test/share
drwxrwxrwt. 2 root root 6  9월 13 00:25 /test/share
```

`user2`가 다시 파일 생성
```bash
[root@localhost permission]# su - user2
[user2@localhost ~]$ echo "user2 file" > /test/share/user2.txt
[user2@localhost ~]$ ls -l /test/share
합계 4
-rw-r--r--. 1 user2 user2 11  9월 13 00:26 user2.txt
```

`user3`가 삭제 시도
```bash
[root@localhost permission]# su - user3
[user3@localhost ~]$ rm /test/share/user2.txt
rm: remove write-protected 일반 파일 '/test/share/user2.txt'? yes
rm: cannot remove '/test/share/user2.txt': 명령을 허용하지 않음
```

공유 디렉터리가 `777`인 경우 다른 사용자의 파일도 삭제 가능
`Sticky Bit(1777)` 적용 후에는 파일 소유자, 디렉터리 소유자 또는 root가 아닌 사용자의 삭제가 제한되는 것을 확인

---

### 8. Set-GID를 이용한 공유 디렉터리 그룹 상속 설정

공유 그룹을 생성하고 `user2`, `user3`를 그룹에 추가
```bash
[root@localhost permission]# groupadd testgroup
[root@localhost permission]# usermod -aG testgroup user2
[root@localhost permission]# usermod -aG testgroup user3
[root@localhost permission]# id user2
uid=1002(user2) gid=1002(user2) groups=1002(user2),1004(testgroup)
[root@localhost permission]# id user3
uid=1003(user3) gid=1003(user3) groups=1003(user3),1004(testgroup)
```

공유 디렉터리 생성 및 그룹 권한 설정
```bash
[root@localhost permission]# mkdir /test/project
[root@localhost permission]# chown root:testgroup /test/project
[root@localhost permission]# chmod 770 /test/project
[root@localhost permission]# ls -ld /test/project
drwxrwx---. 2 root testgroup 6  9월 13 00:32 /test/project
```

Set-GID 적용 전 `user2`가 파일 생성
```bash
[root@localhost permission]# su - user2
[user2@localhost ~]$ touch /test/project/before-sgid.txt
[user2@localhost ~]$ ls -l /test/project
합계 0
-rw-r--r--. 1 user2 user2 0  9월 13 00:32 before-sgid.txt
```

Set-GID 적용
```bash
[root@localhost permission]# chmod 2770 /test/project
[root@localhost permission]# ls -ld /test/project
drwxrws---. 2 root testgroup 29  9월 13 00:32 /test/project
```

Set-GID 적용 후 user2가 파일 생성
```bash
[root@localhost permission]# su - user2
[user2@localhost ~]$ touch /test/project/after-sgid.txt
[user2@localhost ~]$ ls -l /test/project
합계 0
-rw-r--r--. 1 user2 testgroup 0  9월 13 00:33 after-sgid.txt
-rw-r--r--. 1 user2 user2     0  9월 13 00:32 before-sgid.txt
```

Set-GID 적용 전에는 새 파일의 소유 그룹이 파일을 생성한 사용자의 기본 그룹인 `user2`로 설정
`chmod 2770`으로 Set-GID 적용 후에는 새 파일의 소유 그룹이 부모 디렉터리의 소유 그룹인 `testgroup`으로 상속되는 것을 확인

---

### 9. Set-UID를 이용한 실행 권한 확인

Set-UID가 적용된 `passwd` 명령어의 권한 확인

```bash
[root@localhost permission]# ls -l /usr/bin/passwd
-rwsr-xr-x. 1 root root 32656  5월 15  2022 /usr/bin/passwd
```

`일반 사용자` `user2`로 `/etc/shadow` 파일 접근 확인
```bash
[root@localhost permission]# su - user2
[user2@localhost ~]$ ls -l /etc/shadow
----------. 1 root root 1183  9월 13 00:24 /etc/shadow
[user2@localhost ~]$ head -n 1 /etc/shadow
head: cannot open '/etc/shadow' for reading: 허가 거부
```

`user2`는 `/etc/shadow` 파일을 직접 읽을 수 없음을 확인

`/usr/bin/passwd`에는 Set-UID가 설정되어 있어 일반 사용자가 실행하더라도
프로그램이 파일 소유자인 `root`의 유효 사용자 권한(EUID)으로 필요한 작업을 수행 가능

Set-UID는 권한의 소유자 실행 위치에 `s`로 표시되며 숫자 권한에서는 앞자리 `4`를 사용

---

## 실습 결과

 - 파일 및 디렉터리 권한과 소유권 확인
 - `chmod`, `chown`, `chgrp`를 이용하여 권한 및 소유권 변경
 - 일반 사용자를 이용하여 파일 권한에 따른 접근 가능 여부 확인
 - `umask` 값에 따라 새로 생성되는 파일과 디렉터리의 기본 권한이 달라지는 것 확인
 - Sticky Bit, Set-GID, Set-UID 특수 권한의 동작을 실습하여 공유 디렉터리의 파일 보호, 그룹 상속, 실행 파일의 유효 사용자 권한 동작을 확인

