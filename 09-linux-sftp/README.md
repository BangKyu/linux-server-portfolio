# SFTP Server

Rocky Linux의 OpenSSH를 이용하여 SFTP 서버를 구성하고, Windows 클라이언트에서 파일 업로드/다운로드를 테스트했습니다.

추가로 특정 사용자를 SFTP 전용 계정으로 제한하고 `ChrootDirectory`를 적용하여 지정된 경로 밖으로 접근하지 못하도록 구성했습니다.

---

## 1. 실습 환경

| 구분 | 내용 |
|---|---|
| Server | Rocky Linux |
| Server IP | `192.168.111.100` |
| Client | Windows |
| Protocol | SFTP |
| Service | OpenSSH |
| Port | TCP 22 |
| SFTP User | `sftpuser` |

OpenSSH 설치 및 SSH 서비스 상태를 확인했습니다.

```bash
rpm -q openssh-server
rpm -q openssh-clients
systemctl status sshd --no-pager
ss -lntp | grep ':22'
```

확인 결과 SSH 서버가 정상 실행 중이며 TCP 22번 포트에서 대기하고 있었습니다.

```text
LISTEN 0 128 0.0.0.0:22 0.0.0.0:* users:(("sshd",pid=1181,fd=7))
LISTEN 0 128 [::]:22    [::]:*    users:(("sshd",pid=1181,fd=8))
```

SFTP Subsystem 설정도 확인했습니다.

```bash
grep -R "^[[:space:]]*Subsystem[[:space:]]\+sftp" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null
```

```text
/etc/ssh/sshd_config:Subsystem  sftp    /usr/libexec/openssh/sftp-server
```

방화벽에서도 SSH 서비스가 허용된 상태였습니다.

```bash
firewall-cmd --list-services
```

```text
cockpit dhcpv6-client ssh
```

---

## 2. SFTP 사용자 확인

SFTP 테스트용 사용자 `sftpuser`를 사용했습니다.

```bash
id sftpuser
getent passwd sftpuser
ls -ld /home/sftpuser
```

```text
uid=1009(sftpuser) gid=1009(sftpuser) groups=1009(sftpuser)

sftpuser:x:1009:1009::/home/sftpuser:/bin/bash

drwx------. 3 sftpuser sftpuser 78  9월 14 09:45 /home/sftpuser
```

서버에 다운로드 테스트용 파일을 준비했습니다.

```text
/home/sftpuser/server-test.txt
```

```text
Rocky Linux SFTP Server Test
```

---

## 3. Windows에서 SFTP 접속

Windows에서 Rocky Linux 서버로 접속했습니다.

```powershell
sftp sftpuser@192.168.111.100
```

최초 접속 시 서버의 SSH Host Key를 확인한 후 접속에 성공했습니다.

```text
Connected to 192.168.111.100.
sftp>
```

Remote와 Local의 현재 경로를 각각 확인했습니다.

```text
sftp> pwd
Remote working directory: /home/sftpuser

sftp> lpwd
Local working directory: c:\users\이병규
```

- `pwd` : Remote 서버 현재 경로 확인
- `lpwd` : Local Windows 현재 경로 확인
- `cd` : Remote 경로 이동
- `lcd` : Local 경로 이동

---

## 4. 파일 다운로드

서버의 `server-test.txt` 파일을 Windows로 다운로드했습니다.

```text
sftp> get server-test.txt
```

Windows에서 다운로드된 파일을 확인했습니다.

```text
sftp> lls server-test.txt
 Directory of C:\Users\이병규

2026-09-14  오전 09:50                29 server-test.txt
```

Local 경로를 `Downloads`로 변경한 뒤 다시 다운로드하여 저장 위치가 변경되는 것도 확인했습니다.

```text
sftp> lcd Downloads

sftp> lpwd
Local working directory: c:\users\이병규\downloads

sftp> get server-test.txt
Fetching /home/sftpuser/server-test.txt to server-test.txt
server-test.txt 100% 29 9.4KB/s 00:00
```

---

## 5. 파일 업로드

Windows의 `upload-test.txt` 파일을 서버로 업로드했습니다.

```text
sftp> put upload-test.txt
```

서버에서 실제 파일을 확인했습니다.

```bash
ls -l /home/sftpuser
```

```text
-rw-r--r--. 1 sftpuser sftpuser 29  9월 14 09:47 server-test.txt
-rw-r--r--. 1 sftpuser sftpuser 60  9월 14 09:52 upload-test.txt
```

업로드된 파일의 소유자는 `sftpuser:sftpuser`이며 권한은 `644`로 생성되었습니다.

---

## 6. SFTP 디렉터리 조작

Remote 서버에 디렉터리를 생성하고 이동했습니다.

```text
sftp> mkdir sftp-test

sftp> ls
server-test.txt   sftp-test   upload-test.txt

sftp> cd sftp-test

sftp> pwd
Remote working directory: /home/sftpuser/sftp-test
```

특정 Remote 디렉터리를 지정하여 업로드하는 것도 확인했습니다.

```text
sftp> put upload-test.txt sftp-test/

sftp> ls sftp-test
sftp-test/upload-test.txt
```

---

## 7. 사용자 접근 권한 확인

`/home/sftpuser`의 권한은 `700`으로 설정되어 있어 다른 일반 사용자가 접근할 수 없습니다.

```bash
runuser -u guest -- ls -l /home/sftpuser
runuser -u guest -- cat /home/sftpuser/server-test.txt
```

```text
ls: cannot open directory '/home/sftpuser': 허가 거부

cat: /home/sftpuser/server-test.txt: 허가 거부
```

이를 통해 Linux 파일 권한이 SFTP 사용자 데이터 접근에도 적용되는 것을 확인했습니다.

---

## 8. SSH 인증 로그 확인

`journalctl`을 통해 SFTP 접속에 사용된 SSH 인증 로그를 확인했습니다.

```text
Accepted password for sftpuser from 192.168.111.1 port 13035 ssh2

Failed password for sftpuser from 192.168.111.1 port 4045 ssh2

Accepted password for sftpuser from 192.168.111.1 port 4045 ssh2
```

정상 인증뿐만 아니라 잘못된 비밀번호 입력에 의한 인증 실패도 SSH 로그에 기록되는 것을 확인했습니다.

---

# SFTP 전용 사용자 구성

기본 설정에서는 `sftpuser`가 SFTP뿐만 아니라 일반 SSH Shell에도 로그인할 수 있었습니다.

```powershell
ssh sftpuser@192.168.111.100
```

```text
[sftpuser@localhost ~]$ whoami
sftpuser

[sftpuser@localhost ~]$ pwd
/home/sftpuser
```

보안을 강화하기 위해 `sftpuser`를 SFTP 전용 사용자로 제한하고 Chroot를 적용했습니다.

---

## 9. Chroot 디렉터리 구성

다음 구조로 SFTP 전용 디렉터리를 구성했습니다.

```text
/sftp/
└── sftpuser/
    └── upload/
```

Chroot 최상위 경로는 `root`가 소유하고, 실제 파일 업로드 디렉터리만 `sftpuser`가 소유하도록 설정했습니다.

```bash
mkdir -p /sftp/sftpuser/upload

chown root:root /sftp
chown root:root /sftp/sftpuser

chmod 755 /sftp
chmod 755 /sftp/sftpuser

chown sftpuser:sftpuser /sftp/sftpuser/upload
chmod 755 /sftp/sftpuser/upload
```

확인 결과:

```text
drwxr-xr-x. 3 root     root     20  9월 14 10:05 /sftp/sftpuser

drwxr-xr-x. 2 sftpuser sftpuser 29  9월 14 10:11 /sftp/sftpuser/upload
```

---

## 10. sshd_config 설정

설정 변경 전 원본 파일을 백업했습니다.

```bash
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.sftp-backup
```

`/etc/ssh/sshd_config` 마지막에 다음 설정을 추가했습니다.

```text
Match User sftpuser
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

주요 설정:

| 설정 | 역할 |
|---|---|
| `Match User sftpuser` | `sftpuser`에게만 설정 적용 |
| `ChrootDirectory /sftp/%u` | 사용자의 접근 범위를 지정된 경로로 제한 |
| `ForceCommand internal-sftp` | 일반 Shell 대신 SFTP만 실행 |
| `AllowTcpForwarding no` | SSH TCP Forwarding 차단 |
| `X11Forwarding no` | X11 Forwarding 차단 |

설정 문법을 검사했습니다.

```bash
sshd -t
```

오류 출력이 없음을 확인한 후 실제 적용 값을 확인했습니다.

```text
x11forwarding no
allowtcpforwarding no
forcecommand internal-sftp
chrootdirectory /sftp/%u
```

설정을 적용했습니다.

```bash
systemctl reload sshd
```

---

## 11. 일반 SSH Shell 차단 확인

설정 적용 후 Windows에서 일반 SSH 접속을 시도했습니다.

```powershell
ssh sftpuser@192.168.111.100
```

```text
This service allows sftp connections only.
Connection to 192.168.111.100 closed.
```

`ForceCommand internal-sftp` 설정으로 인해 일반 SSH Shell 사용이 차단된 것을 확인했습니다.

---

## 12. Chroot SFTP 접속 확인

일반 SSH Shell은 차단되었지만 SFTP 접속은 정상적으로 동작했습니다.

```powershell
sftp sftpuser@192.168.111.100
```

```text
Connected to 192.168.111.100.

sftp> pwd
Remote working directory: /

sftp> ls
upload
```

사용자에게 `/`로 보이는 경로는 실제 Rocky Linux 서버의 다음 경로입니다.

```text
SFTP 사용자에게 보이는 경로    실제 서버 경로

/                         →    /sftp/sftpuser
/upload                   →    /sftp/sftpuser/upload
```

따라서 `sftpuser`는 시스템 전체 파일 시스템이 아닌 지정된 Chroot 영역 내부에서만 작업할 수 있습니다.

---

## 13. Chroot 내부 파일 업로드

Chroot 내부의 `/upload` 디렉터리로 이동했습니다.

```text
sftp> cd upload

sftp> pwd
Remote working directory: /upload
```

Windows 파일을 업로드했습니다.

```text
sftp> put upload-test.txt
Uploading upload-test.txt to /upload/upload-test.txt
upload-test.txt 100% 60 29.3KB/s 00:00
```

서버에서 실제 저장 위치를 확인했습니다.

```bash
ls -l /sftp/sftpuser/upload
```

```text
-rw-r--r--. 1 sftpuser sftpuser 60  9월 14 10:11 upload-test.txt
```

`stat`으로 실제 권한과 소유권도 확인했습니다.

```text
File: /sftp/sftpuser/upload/upload-test.txt
Size: 60
Access: (0644/-rw-r--r--)
Uid: (1009/sftpuser)
Gid: (1009/sftpuser)
```

즉 Chroot 최상위 디렉터리는 `root`가 관리하고, 사용자는 허용된 `/upload` 디렉터리에서만 파일을 업로드할 수 있도록 구성했습니다.

---

## 14. 최종 결과

| 테스트 | 결과 |
|---|---|
| OpenSSH Server 실행 | 성공 |
| TCP 22 포트 확인 | 성공 |
| Windows → Rocky SFTP 접속 | 성공 |
| 파일 다운로드 `get` | 성공 |
| 파일 업로드 `put` | 성공 |
| Remote / Local 경로 조작 | 성공 |
| 다른 사용자 접근 차단 | 성공 |
| SSH 인증 로그 확인 | 성공 |
| 일반 SSH Shell 접속 | 차단 |
| SFTP 접속 | 성공 |
| Chroot 경로 제한 | 성공 |
| `/upload` 파일 업로드 | 성공 |

SFTP가 SSH를 기반으로 동작한다는 것을 확인하고, 파일 전송뿐만 아니라 Linux 권한과 SSH 설정을 이용하여 **SFTP 전용 사용자 및 Chroot 기반 접근 제한 환경**까지 구성했습니다.
