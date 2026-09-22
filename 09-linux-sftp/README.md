# Linux SFTP 서버 구성 및 보안 설정 실습

## 실습 개요

Rocky Linux에서 OpenSSH의 SFTP 기능을 이용하여 파일 업로드 및 다운로드를 실습하였다.

SFTP 전용 사용자를 생성한 후 `ForceCommand internal-sftp`를 이용하여 일반 SSH Shell 접속을 제한하고, `ChrootDirectory`를 설정하여 사용자가 지정된 디렉터리 밖으로 접근하지 못하도록 구성하였다.

또한 SFTP 접속 및 인증 로그를 확인하고 설정 파일의 백업과 복구 과정을 실습하였다.

---

## 실습 과정

### 1. OpenSSH Server 설치 및 서비스 상태 확인

OpenSSH Server 패키지가 설치되어 있는지 확인하였다.

```bash
[root@Server-A ~]# rpm -qa | grep openssh-server
openssh-server-9.9p1-7.el9_8.rocky.0.1.x86_64
```

`sshd` 서비스 상태 확인

```bash
[root@Server-A ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running)
```

OpenSSH Server가 설치되어 있으며 `sshd` 서비스가 정상적으로 실행 중인 것을 확인하였다.

---

### 2. SFTP 사용자 생성

SFTP 실습에 사용할 `sftpuser` 사용자 생성

```bash
[root@Server-A ~]# useradd sftpuser
```

비밀번호 설정

```bash
[root@Server-A ~]# passwd sftpuser
sftpuser 사용자의 비밀 번호 변경 중
새 암호:
새 암호 재입력:
passwd: 모든 인증 토큰이 성공적으로 업데이트 되었습니다.
```

사용자 정보 확인

```bash
[root@Server-A ~]# id sftpuser
uid=1012(sftpuser) gid=1013(sftpuser) groups=1013(sftpuser)
```

홈 디렉터리 확인

```bash
[root@Server-A ~]# ls -ld /home/sftpuser
drwx------. 3 sftpuser sftpuser 78  9월 22 09:35 /home/sftpuser
```

`sftpuser` 계정과 `/home/sftpuser` 홈 디렉터리가 정상적으로 생성된 것을 확인하였다.

---

### 3. SFTP 접속 테스트

localhost를 이용하여 `sftpuser` 계정으로 SFTP 접속

```bash
[root@Server-A ~]# sftp sftpuser@localhost
The authenticity of host 'localhost (::1)' can't be established.
ED25519 key fingerprint is SHA256:KmymTWFVG9z+5T7PkQP52JDcLq41JiLLUH97dU90v7E.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'localhost' (ED25519) to the list of known hosts.
sftpuser@localhost's password:
Connected to localhost.
```

Remote 작업 디렉터리 확인

```bash
sftp> pwd
Remote working directory: /home/sftpuser
```

파일 목록 확인

```bash
sftp> ls -la
drwx------    ? sftpuser sftpuser       78 Sep 22 09:35 .
drwxr-xr-x    ? root     root           48 Sep 22 09:35 ..
-rw-r--r--    ? sftpuser sftpuser       18 Apr 30  2024 .bash_logout
-rw-r--r--    ? sftpuser sftpuser      141 Apr 30  2024 .bash_profile
-rw-r--r--    ? sftpuser sftpuser      492 Apr 30  2024 .bashrc
```

SFTP 접속 종료

```bash
sftp> exit
```

`sftpuser` 계정으로 SFTP 접속이 정상적으로 가능한 것을 확인하였다.

---

### 4. SFTP 파일 업로드

root 홈 디렉터리에 테스트 파일 생성

```bash
[root@Server-A ~]# touch sftp-test.txt
[root@Server-A ~]# echo "SFTP TEST" > sftp-test.txt
```

SFTP 접속

```bash
[root@Server-A ~]# sftp sftpuser@localhost
sftpuser@localhost's password:
Connected to localhost.
```

`put` 명령을 이용하여 파일 업로드

```bash
sftp> put sftp-test.txt /home/sftpuser
Uploading sftp-test.txt to /home/sftpuser/sftp-test.txt
sftp-test.txt                                              100%   10    19.5KB/s   00:00
```

업로드된 파일 확인

```bash
sftp> ls -l
-rw-r--r--    ? sftpuser sftpuser       10 Sep 22 09:44 sftp-test.txt
```

접속 종료 후 실제 파일 확인

```bash
[root@Server-A ~]# ls -l /home/sftpuser
합계 4
-rw-r--r--. 1 sftpuser sftpuser 10  9월 22 09:44 sftp-test.txt
```

`put` 명령을 이용하여 파일을 SFTP 서버에 업로드할 수 있는 것을 확인하였다.

---

### 5. SFTP 파일 다운로드

기존 파일 이름 변경

```bash
[root@Server-A ~]# mv sftp-test.txt sftp-mv.txt
```

SFTP 접속 후 파일 확인

```bash
[root@Server-A ~]# sftp sftpuser@localhost
sftpuser@localhost's password:
Connected to localhost.

sftp> ls -l
-rw-r--r--    ? sftpuser sftpuser       10 Sep 22 09:44 sftp-test.txt
```

`get` 명령을 이용하여 파일 다운로드

```bash
sftp> get sftp-test.txt
Fetching /home/sftpuser/sftp-test.txt to sftp-test.txt
sftp-test.txt                                              100%   10     8.9KB/s   00:00
```

접속 종료

```bash
sftp> exit
```

파일 확인

```bash
[root@Server-A ~]# ls -l
-rw-r--r--. 1 root root 10  9월 22 09:43 sftp-mv.txt
-rw-r--r--. 1 root root 10  9월 22 09:47 sftp-test.txt
```

파일 내용 확인

```bash
[root@Server-A ~]# cat sftp-test.txt
SFTP TEST
```

`get` 명령을 이용하여 SFTP 서버의 파일을 다운로드할 수 있는 것을 확인하였다.

---

### 6. sshd_config 백업

SFTP 전용 설정을 적용하기 전 `sshd_config` 파일을 백업하였다.

```bash
[root@Server-A ~]# mkdir sshd_backup
[root@Server-A ~]# cp -r /etc/ssh/sshd_config sshd_backup/
```

확인

```bash
[root@Server-A ~]# ls -l sshd_backup/
합계 4
-rw-------. 1 root root 3674  9월 22 09:52 sshd_config
```

SSH 설정을 변경하기 전에 기존 설정 파일을 백업하였다.

---

### 7. SFTP 전용 사용자 설정

`/etc/ssh/sshd_config` 수정

```bash
[root@Server-A ~]# vi /etc/ssh/sshd_config
```

다음 내용을 추가하였다.

```text
Match User sftpuser
    ForceCommand internal-sftp
```

SSH 설정 문법 검사

```bash
[root@Server-A ~]# sshd -t
```

아무런 출력이 없으므로 설정 문법이 정상임을 확인하였다.

`sshd` 서비스 재시작

```bash
[root@Server-A ~]# systemctl restart sshd
```

서비스 상태 확인

```bash
[root@Server-A ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
     Active: active (running)
```

---

### 8. 일반 SSH Shell 접속 제한 확인

`sftpuser` 계정으로 일반 SSH 접속 시도

```bash
[root@Server-A ~]# ssh sftpuser@localhost
sftpuser@localhost's password:
This service allows sftp connections only.
Connection to localhost closed.
```

SFTP 접속 확인

```bash
[root@Server-A ~]# sftp sftpuser@localhost
sftpuser@localhost's password:
Connected to localhost.
```

`ForceCommand internal-sftp` 설정을 이용하여 일반 SSH Shell 접속은 제한하고 SFTP 접속만 허용되는 것을 확인하였다.

---

### 9. SFTP Chroot 디렉터리 구성

SFTP 사용자가 서버의 다른 디렉터리에 접근하지 못하도록 Chroot 환경을 구성하였다.

디렉터리 생성

```bash
[root@Server-A ~]# mkdir -p /sftp/sftpuser/upload
```

디렉터리 상태 확인

```bash
[root@Server-A ~]# ls -ld /sftp /sftp/sftpuser /sftp/sftpuser/upload
drwxr-xr-x. 3 root root 22  9월 22 10:01 /sftp
drwxr-xr-x. 3 root root 20  9월 22 10:01 /sftp/sftpuser
drwxr-xr-x. 2 root root  6  9월 22 10:01 /sftp/sftpuser/upload
```

업로드 디렉터리 소유권 변경

```bash
[root@Server-A ~]# chown sftpuser:sftpuser /sftp/sftpuser/upload
```

확인

```bash
[root@Server-A ~]# ls -ld /sftp /sftp/sftpuser /sftp/sftpuser/upload
drwxr-xr-x. 3 root     root     22  9월 22 10:01 /sftp
drwxr-xr-x. 3 root     root     20  9월 22 10:01 /sftp/sftpuser
drwxr-xr-x. 2 sftpuser sftpuser  6  9월 22 10:01 /sftp/sftpuser/upload
```

Chroot 최상위 디렉터리는 `root`가 소유하고 실제 파일 업로드가 필요한 `upload` 디렉터리만 `sftpuser`에게 권한을 부여하였다.

---

### 10. sshd_config Chroot 설정

`/etc/ssh/sshd_config` 수정

```bash
[root@Server-A ~]# vi /etc/ssh/sshd_config
```

`sftpuser` 설정을 다음과 같이 구성하였다.

```text
Match User sftpuser
    ChrootDirectory /sftp/sftpuser
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

설정 검사

```bash
[root@Server-A ~]# sshd -t
```

서비스 재시작

```bash
[root@Server-A ~]# systemctl restart sshd
```

확인

```bash
[root@Server-A ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
     Active: active (running)
```

---

### 11. Chroot 적용 확인

SFTP 접속

```bash
[root@Server-A ~]# sftp sftpuser@localhost
sftpuser@localhost's password:
Connected to localhost.
```

현재 디렉터리 확인

```bash
sftp> pwd
Remote working directory: /
```

파일 목록 확인

```bash
sftp> ls -l
drwxr-xr-x    ? 1012     1013            6 Sep 22 10:01 upload
```

`/upload`로 이동

```bash
sftp> cd upload
sftp> pwd
Remote working directory: /upload
```

실제 서버의 `/sftp/sftpuser` 디렉터리가 SFTP 사용자에게는 `/`로 보이는 것을 확인하였다.

---

### 12. Chroot 환경에서 파일 업로드

파일 확인

```bash
sftp> lls sftp-mv.txt
sftp-mv.txt
```

`/upload` 디렉터리에 업로드

```bash
sftp> put sftp-mv.txt
Uploading sftp-mv.txt to /upload/sftp-mv.txt
sftp-mv.txt                                                100%   10    10.3KB/s   00:00
```

파일 확인

```bash
sftp> ls -l
-rw-r--r--    ? 1012     1013           10 Sep 22 10:02 sftp-mv.txt
```

접속 종료 후 실제 서버 경로 확인

```bash
[root@Server-A ~]# ls -l /sftp/sftpuser/upload/
합계 4
-rw-r--r--. 1 sftpuser sftpuser 10  9월 22 10:02 sftp-mv.txt
```

SFTP에서 보이는 `/upload`는 실제 서버에서는 `/sftp/sftpuser/upload`인 것을 확인하였다.

---

### 13. Chroot 접근 제한 확인

상위 디렉터리 이동 시도

```bash
sftp> pwd
Remote working directory: /

sftp> cd ../
sftp> pwd
Remote working directory: /
```

상위 디렉터리로 이동을 시도해도 `/`보다 위로 이동할 수 없었다.

실제 서버의 `/etc` 접근 시도

```bash
sftp> cd /etc
stat remote: No such file or directory
```

Chroot 환경으로 인해 실제 서버의 `/etc` 디렉터리에 접근할 수 없는 것을 확인하였다.

---

### 14. Chroot 최상위 디렉터리 쓰기 제한 확인

Chroot의 `/` 위치에 파일 업로드 시도

```bash
sftp> put sftp-mv.txt /
Uploading sftp-mv.txt to /sftp-mv.txt
dest open "/sftp-mv.txt": Permission denied
```

`/upload`에는 정상적으로 업로드되었다.

```bash
sftp> put sftp-mv.txt /upload
Uploading sftp-mv.txt to /upload/sftp-mv.txt
sftp-mv.txt                                                100%   10    20.7KB/s   00:00
```

디렉터리 권한 구조

```text
/sftp/sftpuser
→ root:root
→ 직접 파일 생성 불가

/sftp/sftpuser/upload
→ sftpuser:sftpuser
→ 파일 업로드 가능
```

Chroot 최상위 디렉터리는 root가 관리하고 하위 업로드 디렉터리만 사용자에게 쓰기 권한을 부여하는 구조를 확인하였다.

---

### 15. SFTP 접속 로그 확인

`/var/log/secure`에서 `sftpuser` 관련 로그 확인

```bash
[root@Server-A ~]# grep sftpuser /var/log/secure
```

주요 로그인 성공 기록

```text
Accepted password for sftpuser from ::1 port 34130 ssh2
pam_unix(sshd:session): session opened for user sftpuser(uid=1012)
Disconnected from user sftpuser ::1 port 34130
pam_unix(sshd:session): session closed for user sftpuser
```

`journalctl`을 이용한 sshd 로그 확인

```bash
[root@Server-A ~]# journalctl -u sshd --no-pager | tail -n 20
```

로그의 의미

```text
Accepted password
→ 인증 성공

session opened
→ 세션 시작

Disconnected
→ 연결 종료

session closed
→ 세션 종료
```

---

### 16. SFTP 전용 계정의 SSH 차단 로그 확인

일반 SSH 접속을 시도했을 때 다음 로그가 기록되었다.

```text
error: Connection from user sftpuser ::1 port 47182: refusing non-sftp session
```

이는 `ForceCommand internal-sftp` 설정으로 인해 SFTP가 아닌 일반 SSH Shell 세션이 거부된 기록이다.

SFTP 전용 계정 설정이 정상적으로 동작하는 것을 로그에서도 확인하였다.

---

### 17. 인증 실패 로그 확인

일부러 잘못된 비밀번호를 이용하여 접속을 시도하였다.

```bash
[root@Server-A ~]# ssh sftpuser@localhost
sftpuser@localhost's password:
Permission denied, please try again.
```

로그 확인

```bash
[root@Server-A ~]# grep sftpuser /var/log/secure | tail -n 10
```

결과

```text
password check failed for user (sftpuser)
pam_unix(sshd:auth): authentication failure; user=sftpuser
Failed password for sftpuser from ::1 port 40560 ssh2
Connection closed by authenticating user sftpuser ::1 port 40560 [preauth]
```

성공과 실패 로그를 비교하였다.

```text
Accepted password
→ 인증 성공

Failed password
→ 인증 실패
```

로그를 이용하여 SSH/SFTP 인증 성공 및 실패 기록을 확인할 수 있는 것을 확인하였다.

---

### 18. sshd_config 원본과 수정본 비교

백업해둔 설정과 현재 설정을 `diff`로 비교하였다.

```bash
[root@Server-A ~]# diff /root/sshd_backup/sshd_config /etc/ssh/sshd_config
130a131,135
> Match User sftpuser
>     ChrootDirectory /sftp/sftpuser
>     ForceCommand internal-sftp
>     AllowTcpForwarding no
>     X11Forwarding no
```

기존 설정 파일과 현재 설정 파일의 차이가 SFTP 전용 및 Chroot 설정임을 확인하였다.

현재 적용된 SFTP 전용 설정 파일을 별도로 백업하였다.
```bash
[root@Server-A ~]# mkdir sshd_backup2
[root@Server-A ~]# cp /etc/ssh/sshd_config sshd_backup2/
```

백업 파일 확인
```bash
[root@Server-A ~]# ls -l /root/sshd_backup2/sshd_config
-rw-------. 1 root root 3807  9월 22 10:18 /root/sshd_backup2/sshd_config
```
sshd_backup2에는 ForceCommand internal-sftp와 ChrootDirectory 설정이 적용된 SFTP 전용 sshd_config를 보관하였다.
---

### 19. SFTP 설정 원복 확인

기존 설정으로 원복한 후 일반 SSH 접속 테스트

```bash
[root@Server-A ~]# ssh sftpuser@localhost
sftpuser@localhost's password:
```

정상적으로 Shell 접속

```bash
[sftpuser@Server-A ~]$ whoami
sftpuser

[sftpuser@Server-A ~]$ pwd
/home/sftpuser
```

접속 종료

```bash
[sftpuser@Server-A ~]$ exit
로그아웃
Connection to localhost closed.
```

SFTP 접속 확인

```bash
[root@Server-A ~]# sftp sftpuser@localhost
sftpuser@localhost's password:
Connected to localhost.

sftp> pwd
Remote working directory: /home/sftpuser
```

SFTP 전용 및 Chroot 설정을 제거하면 일반 SSH와 SFTP 모두 사용할 수 있으며 SFTP의 기본 위치가 다시 `/home/sftpuser`가 되는 것을 확인하였다.

---

### 20. SFTP 전용 Chroot 설정 재적용

일반 설정 파일을 추가로 백업하였다.

```bash
[root@Server-A ~]# mkdir sshd_backup3
[root@Server-A ~]# cp /etc/ssh/sshd_config /root/sshd_backup3/
```

SFTP 전용 설정이 저장된 백업 파일을 복원하였다.

```bash
[root@Server-A ~]# cp sshd_backup2/sshd_config /etc/ssh/
cp: overwrite '/etc/ssh/sshd_config'? y
```

설정 검사

```bash
[root@Server-A ~]# sshd -t
```

서비스 재시작

```bash
[root@Server-A ~]# systemctl restart sshd
```

상태 확인

```bash
[root@Server-A ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
     Active: active (running)
```

---

### 21. 최종 SFTP 전용 계정 동작 확인

일반 SSH 접속 테스트

```bash
[root@Server-A ~]# ssh sftpuser@localhost
sftpuser@localhost's password:
This service allows sftp connections only.
Connection to localhost closed.
```

일반 SSH Shell 접속이 제한되는 것을 확인하였다.

SFTP 접속

```bash
[root@Server-A ~]# sftp sftpuser@localhost
sftpuser@localhost's password:
Connected to localhost.
```

Chroot 적용 확인

```bash
sftp> pwd
Remote working directory: /
```

최종적으로 `sftpuser` 계정은 일반 SSH Shell 접속은 제한되고 SFTP 접속과 Chroot 환경만 사용할 수 있도록 구성하였다.

---

## 실습 결과

* `sftp` 명령을 이용한 SFTP 접속 확인
* `put` 명령을 이용한 파일 업로드
* `get` 명령을 이용한 파일 다운로드
* `ForceCommand internal-sftp`를 이용하여 일반 SSH Shell 접속 제한
* `ChrootDirectory`를 이용하여 SFTP 사용자의 접근 가능 경로 제한
* `/upload` 디렉터리에 `sftpuser` 쓰기 권한 부여
* Chroot 환경에서 상위 디렉터리 및 실제 서버 `/etc` 접근 제한 확인
* Chroot 최상위 `/`에는 파일 업로드가 불가능한 것을 확인
* `/upload`에는 정상적으로 파일 업로드가 가능한 것을 확인
* `/var/log/secure`에서 SFTP 로그인 성공 및 종료 로그 확인
* `ForceCommand internal-sftp`에 의한 일반 SSH 세션 차단 로그 확인
* 잘못된 비밀번호 입력 후 `Failed password` 인증 실패 로그 확인
* `diff`를 이용하여 기존 `sshd_config`와 수정된 설정 비교
* SFTP 설정 원복 후 일반 SSH 및 SFTP 접속 확인
* SFTP 전용 Chroot 설정을 다시 적용하여 최종 동작 확인
