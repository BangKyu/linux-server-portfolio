# Linux 파일 권한 및 소유권 관리 실습

## 실습 개요

Rocky Linux에서 파일과 디렉터리의 권한 및 소유권을 관리하는 방법을 실습하였다.

`chmod`, `chown`을 이용하여 권한과 소유권을 변경하고 파일과 디렉터리에서 `r`, `w`, `x` 권한의 차이를 실습하였다.

`umask`를 이용한 기본 권한 설정과 SetUID, SetGID, Sticky Bit 특수 권한의 동작을 실습하였다.

---

## 실습 과정

### 1. 실습 파일 및 디렉터리 생성

```bash
[root@Server-A ~]# mkdir /test/linux-permission/
[root@Server-A ~]# cd /test/linux-permission/
[root@Server-A linux-permission]# pwd
/test/linux-permission

[root@Server-A linux-permission]# touch file1.txt
[root@Server-A linux-permission]# touch file2.txt
[root@Server-A linux-permission]# mkdir dir1

[root@Server-A linux-permission]# ls -l
합계 0
drwxr-xr-x. 2 root root 6  9월 16 14:43 dir1
-rw-r--r--. 1 root root 0  9월 16 14:42 file1.txt
-rw-r--r--. 1 root root 0  9월 16 14:42 file2.txt
```

`touch`를 사용하여 `file1.txt`, `file2.txt` 파일 생성

`mkdir`를 사용하여 `dir1` 디렉터리 생성

`ls -l`을 사용하여 파일과 디렉터리의 Permission과 Owner, Group 확인

---

### 2. 숫자 방식 Permission 변경

```bash
[root@Server-A linux-permission]# chmod 640 file1.txt
[root@Server-A linux-permission]# chmod 750 file2.txt
[root@Server-A linux-permission]# chmod 770 dir1

[root@Server-A linux-permission]# ls -l
합계 0
drwxrwx---. 2 root root 6  9월 16 14:43 dir1
-rw-r-----. 1 root root 0  9월 16 14:42 file1.txt
-rwxr-x---. 1 root root 0  9월 16 14:42 file2.txt
```

`chmod`의 숫자 방식을 사용하여 Permission 변경

- `640` → `rw-r-----`
- `750` → `rwxr-x---`
- `770` → `rwxrwx---`

`r=4`, `w=2`, `x=1` 값을 조합하여 Owner, Group, Other의 권한을 설정

---

### 3. 문자 방식 Permission 변경

```bash
[root@Server-A linux-permission]# chmod g-w,o+rx dir1
[root@Server-A linux-permission]# chmod u+x,g+x file1.txt
[root@Server-A linux-permission]# chmod u-x,g=rw,o+r file2.txt

[root@Server-A linux-permission]# ls -l
합계 0
drwxr-xr-x. 2 root root 6  9월 16 14:43 dir1
-rwxr-x---. 1 root root 0  9월 16 14:42 file1.txt
-rw-rw-r--. 1 root root 0  9월 16 14:42 file2.txt
```

문자 방식의 `chmod`를 사용하여 특정 대상의 권한을 추가하거나 제거

- `u` → Owner
- `g` → Group
- `o` → Other
- `+` → 권한 추가
- `-` → 권한 제거
- `=` → 지정한 권한으로 설정

`file1.txt`는 `rwxr-x---`, `file2.txt`는 `rw-rw-r--`, `dir1`은 `rwxr-xr-x`로 변경된 것을 확인

---

### 4. 파일 및 디렉터리 소유권 변경

```bash
[root@Server-A linux-permission]# chown guest:guest file1.txt
[root@Server-A linux-permission]# chown :guest file2.txt
[root@Server-A linux-permission]# chown guest:root dir1

[root@Server-A linux-permission]# ls -l
합계 0
drwxr-xr-x. 2 guest root  6  9월 16 14:43 dir1
-rwxr-x---. 1 guest guest 0  9월 16 14:42 file1.txt
-rw-rw-r--. 1 root  guest 0  9월 16 14:42 file2.txt
```

`chown guest:guest`를 사용하여 Owner와 Group을 동시에 변경

`chown :guest`를 사용하여 Owner는 유지하고 Group만 변경

`chown guest:root`를 사용하여 Owner와 Group을 각각 지정

최종 소유권:

```text
             Owner    Group
dir1         guest    root
file1.txt    guest    guest
file2.txt    root     guest
```

---

### 5. 디렉터리 Permission 동작 확인

테스트용 디렉터리와 파일 생성

```bash
[root@Server-A linux-permission]# mkdir dir2
[root@Server-A linux-permission]# touch dir2/inside.txt
[root@Server-A linux-permission]# echo "Permission Test" > dir2/inside.txt
[root@Server-A linux-permission]# chown guest:root dir2
[root@Server-A linux-permission]# chown guest:root dir2/inside.txt

[root@Server-A linux-permission]# ls -ld dir2
drwxr-xr-x. 2 guest root 24  9월 16 14:54 dir2

[root@Server-A linux-permission]# ls -l dir2
합계 4
-rw-r--r--. 1 guest root 16  9월 16 14:54 inside.txt
```

#### `r` 권한만 부여

```bash
[root@Server-A linux-permission]# chmod 400 dir2
[root@Server-A linux-permission]# ls -ld dir2
dr--------. 2 guest root 24  9월 16 14:54 dir2

[root@Server-A linux-permission]# su - guest
[guest@Server-A ~]$ ls /test/linux-permission/dir2
ls: cannot access '/test/linux-permission/dir2/inside.txt': 허가 거부
inside.txt

[guest@Server-A ~]$ cd /test/linux-permission/dir2
-bash: cd: /test/linux-permission/dir2: 허가 거부

[guest@Server-A ~]$ cat /test/linux-permission/dir2/inside.txt
cat: /test/linux-permission/dir2/inside.txt: 허가 거부
```

디렉터리에 `r` 권한만 존재하면 내부 파일 이름은 확인할 수 있지만 `x` 권한이 없어 디렉터리 진입과 내부 파일 접근은 불가능한 것을 확인

#### `x` 권한만 부여

```bash
[root@Server-A linux-permission]# chmod 100 dir2
[root@Server-A linux-permission]# ls -ld dir2
d--x------. 2 guest root 24  9월 16 14:54 dir2

[root@Server-A linux-permission]# su - guest
[guest@Server-A ~]$ ls /test/linux-permission/dir2
ls: cannot open directory '/test/linux-permission/dir2': 허가 거부

[guest@Server-A ~]$ cd /test/linux-permission/dir2
[guest@Server-A dir2]$ pwd
/test/linux-permission/dir2

[guest@Server-A dir2]$ cat /test/linux-permission/dir2/inside.txt
Permission Test
```

`x` 권한이 있으면 디렉터리 내부 목록은 확인할 수 없지만 디렉터리 진입은 가능

파일 이름을 알고 있고 해당 파일의 Permission이 허용되면 내부 파일에 접근할 수 있는 것을 확인

#### `w+x` 권한 부여

```bash
[root@Server-A linux-permission]# chmod 300 dir2

[root@Server-A linux-permission]# ls -l
합계 0
drwxr-xr-x. 2 guest root   6  9월 16 14:43 dir1
d-wx------. 2 guest root  24  9월 16 14:54 dir2
-rwxr-x---. 1 guest guest  0  9월 16 14:42 file1.txt
-rw-rw-r--. 1 root  guest  0  9월 16 14:42 file2.txt

[root@Server-A linux-permission]# su - guest

[guest@Server-A ~]$ ls /test/linux-permission/dir2
ls: cannot open directory '/test/linux-permission/dir2': 허가 거부

[guest@Server-A ~]$ touch /test/linux-permission/dir2/new.txt

[guest@Server-A ~]$ rm /test/linux-permission/dir2/inside.txt

[guest@Server-A ~]$ ls -l /test/linux-permission/dir2/new.txt
-rw-r--r--. 1 guest guest 0  9월 16 15:01 /test/linux-permission/dir2/new.txt
```

디렉터리에 `w+x` 권한이 있으면 내부 목록을 확인할 수 없어도 파일 생성 및 삭제가 가능한 것을 확인

디렉터리에서 파일을 생성하거나 삭제할 때 부모 디렉터리의 `w`와 `x` 권한이 중요하다는 것을 확인

---

### 6. umask를 이용한 기본 Permission 확인

기본 `umask` 확인

```bash
[root@Server-A linux-permission]# umask
0022
```

`umask 0022` 환경에서 생성된 파일과 디렉터리

```bash
[root@Server-A linux-permission]# touch umask-file.txt
[root@Server-A linux-permission]# mkdir umask-dir

[root@Server-A linux-permission]# ls -l
합계 0
drwxr-xr-x. 2 root root 6  9월 16 15:07 umask-dir
-rw-r--r--. 1 root root 0  9월 16 15:07 umask-file.txt
```

일반 파일은 `644`, 디렉터리는 `755`로 생성되는 것을 확인

`umask`를 `0027`로 변경

```bash
[root@Server-A linux-permission]# umask 027
[root@Server-A linux-permission]# umask
0027

[root@Server-A linux-permission]# touch umask-file2.txt
[root@Server-A linux-permission]# mkdir umask-dir2

[root@Server-A linux-permission]# ls -l
합계 0
drwxr-xr-x. 2 root root 6  9월 16 15:07 umask-dir
drwxr-x---. 2 root root 6  9월 16 15:07 umask-dir2
-rw-r--r--. 1 root root 0  9월 16 15:07 umask-file.txt
-rw-r-----. 1 root root 0  9월 16 15:07 umask-file2.txt
```

`umask 0027` 적용 결과:

```text
파일      → 640 → rw-r-----
디렉터리  → 750 → rwxr-x---
```

실습 후 기본값으로 복구

```bash
[root@Server-A linux-permission]# umask 022
[root@Server-A linux-permission]# umask
0022
```

---

### 7. Sticky Bit 설정 및 동작 확인

Linux의 대표적인 Sticky Bit 적용 디렉터리인 `/tmp` 확인

```bash
[root@Server-A linux-permission]# ls -ld /tmp
drwxrwxrwt. 21 root root 4096  9월 16 15:01 /tmp
```

테스트용 공유 디렉터리 생성

```bash
[root@Server-A linux-permission]# mkdir shared
[root@Server-A linux-permission]# chmod 1777 shared/

[root@Server-A linux-permission]# ls -ld shared/
drwxrwxrwt. 2 root root 6  9월 16 15:10 shared/
```

`1777`을 설정하여 Other의 실행 권한 위치에 Sticky Bit을 나타내는 `t`가 표시되는 것을 확인

`guest` 사용자가 파일 생성

```bash
[guest@Server-A ~]$ touch /test/linux-permission/shared/guest.txt

[guest@Server-A ~]$ ls -l /test/linux-permission/shared/guest.txt
-rw-r--r--. 1 guest guest 0  9월 16 15:12 /test/linux-permission/shared/guest.txt
```

`user1`에서 `guest`가 생성한 파일 삭제 시도

```bash
[user1@Server-A ~]$ rm /test/linux-permission/shared/guest.txt
rm: remove write-protected 일반 빈 파일 '/test/linux-permission/shared/guest.txt'? yes
rm: cannot remove '/test/linux-permission/shared/guest.txt': 명령을 허용하지 않음
```

`user1` 자신의 파일은 생성 및 삭제 가능

```bash
[user1@Server-A ~]$ touch /test/linux-permission/shared/user1.txt
[user1@Server-A ~]$ rm /test/linux-permission/shared/user1.txt

[user1@Server-A ~]$ ls -l /test/linux-permission/shared/
합계 0
-rw-r--r--. 1 guest guest 0  9월 16 15:12 guest.txt
```

Sticky Bit이 설정된 공유 디렉터리에서는 여러 사용자가 파일을 생성할 수 있지만 다른 사용자가 소유한 파일의 삭제가 제한되는 것을 확인

---

### 8. SetGID를 이용한 Group 상속 확인

SetGID 동작 확인을 위해 `user1`을 `guest` 그룹에 추가

```bash
[root@Server-A linux-permission]# usermod -aG guest user1
[root@Server-A linux-permission]# id user1
uid=1001(user1) gid=1001(user1) groups=1001(user1),1000(guest)
```

`usermod -aG`를 사용하여 `user1`을 `guest` Supplementary Group에 추가

`id user1`을 사용하여 `user1`의 Primary Group은 `user1`, Supplementary Group은 `guest`인 것을 확인

테스트용 공유 디렉터리 생성 후 Group을 `guest`로 지정

```bash
[root@Server-A linux-permission]# mkdir teamdir
[root@Server-A linux-permission]# chown :guest teamdir
[root@Server-A linux-permission]# chmod 2775 teamdir

[root@Server-A linux-permission]# ls -ld teamdir/
drwxrwsr-x. 2 root guest 6  9월 16 15:18 teamdir/
```

`2775`를 설정하여 Group 실행 권한 위치에 SetGID를 나타내는 `s`가 표시되는 것을 확인

`user1`이 `teamdir` 내부에 파일 생성

```bash
[user1@Server-A ~]$ touch /test/linux-permission/teamdir/user1-file.txt

[user1@Server-A ~]$ ls -l /test/linux-permission/teamdir/user1-file.txt
-rw-r--r--. 1 user1 guest 0  9월 16 15:26 /test/linux-permission/teamdir/user1-file.txt
```

파일의 Owner는 파일을 생성한 `user1`으로 설정되고 Group은 부모 디렉터리의 Group인 `guest`를 상속한 것을 확인

SetGID가 설정된 디렉터리에서는 새로 생성되는 파일 및 디렉터리의 Group을 부모 디렉터리의 Group으로 통일할 수 있음을 확인

---

### 9. SetUID 확인

SetUID가 적용된 대표적인 실행 파일인 `/usr/bin/passwd` 확인

```bash
[root@Server-A linux-permission]# ls -l /usr/bin/passwd
-rwsr-xr-x. 1 root root 32656  5월 15  2022 /usr/bin/passwd

[root@Server-A linux-permission]# stat /usr/bin/passwd
Access: (4755/-rwsr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
```

`/usr/bin/passwd`의 Permission이 `4755`이며 Owner 실행 권한 위치에 SetUID를 나타내는 `s`가 표시되는 것을 확인

SetUID가 적용된 실행 파일은 실행 시 파일 Owner의 권한을 사용하여 필요한 작업을 수행할 수 있음을 확인

---

## 실습 결과

- 숫자 방식과 문자 방식의 `chmod`를 사용하여 파일 및 디렉터리 Permission 변경
- `r`, `w`, `x` 권한과 Owner, Group, Other의 Permission 구조 확인
- `chown`을 사용하여 파일과 디렉터리의 Owner 및 Group 변경
- 파일과 디렉터리에서 `r`, `w`, `x` 권한이 다르게 동작하는 것을 실습으로 확인
- 디렉터리의 `x` 권한이 디렉터리 진입 및 내부 경로 접근에 필요함을 확인
- 디렉터리의 `w+x` 권한을 통해 내부 파일 생성 및 삭제가 가능함을 확인
- `umask 0022`, `0027`에 따른 기본 파일 및 디렉터리 Permission 차이 확인
- Sticky Bit `1777`을 사용하여 공유 디렉터리에서 다른 사용자의 파일 삭제 제한 확인
- SetGID `2775`를 사용하여 새 파일이 부모 디렉터리의 Group을 상속하는 것을 확인
- `/usr/bin/passwd`를 통해 SetUID `4755`가 적용된 실행 파일 확인
