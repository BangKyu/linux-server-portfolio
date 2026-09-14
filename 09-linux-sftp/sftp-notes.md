# SFTP 이론 정리

## 1. FTP란?

FTP(File Transfer Protocol)는 Client와 Server 사이에서 파일을 전송하기 위한 프로토콜이다.

기본적으로 다음 포트를 사용한다.

```text
TCP 21 : FTP 제어 연결
TCP 20 : FTP 데이터 전송
```

FTP는 인증 정보와 데이터를 암호화하지 않은 평문 형태로 전송할 수 있기 때문에 보안에 취약하다.

---

## 2. SSH란?

SSH(Secure Shell)는 네트워크를 통해 원격 시스템에 안전하게 접속하기 위한 프로토콜이다.

기본 포트:

```text
TCP 22
```

SSH를 이용하면 다음과 같은 작업이 가능하다.

- 원격 Shell 접속
- 사용자 인증
- 암호화된 통신
- 파일 전송
- Port Forwarding

일반적인 접속 형식:

```bash
ssh 사용자명@서버IP
```

예:

```bash
ssh sftpuser@192.168.111.100
```

---

## 3. SFTP란?

SFTP(SSH File Transfer Protocol)는 SSH 연결을 이용하여 파일을 안전하게 전송하는 프로토콜이다.

```text
SFTP
 ↓
SSH
 ↓
TCP 22
```

SFTP는 이름에 FTP가 포함되어 있지만 기존 FTP와는 구조적으로 다른 프로토콜이다.

```text
FTP
 ├── TCP 20
 └── TCP 21

SFTP
 └── SSH
      └── TCP 22
```

따라서 SFTP를 사용하기 위해 FTP 서버인 `vsftpd`를 별도로 구성할 필요가 없다.

OpenSSH 서버는 SFTP 기능을 제공할 수 있으며 대표적으로 다음 방식을 사용할 수 있다.

```text
/usr/libexec/openssh/sftp-server
internal-sftp
```

---

## 4. FTP와 SFTP 비교

| 구분 | FTP | SFTP |
|---|---|---|
| 의미 | File Transfer Protocol | SSH File Transfer Protocol |
| 기본 포트 | TCP 20, 21 | TCP 22 |
| 암호화 | 기본적으로 평문 | SSH를 이용한 암호화 |
| 서버 프로그램 예 | vsftpd | OpenSSH |
| SSH 기반 | X | O |
| 파일 전송 | O | O |

SFTP는 FTP에 단순히 보안 기능을 추가한 것이 아니라 SSH 기반으로 동작하는 별도의 파일 전송 프로토콜이다.

---

# SFTP 기본 사용법

## 5. SFTP 접속

기본 형식:

```bash
sftp 사용자명@서버IP
```

예:

```bash
sftp sftpuser@192.168.111.100
```

정상적으로 접속하면 다음과 같은 Prompt가 나타난다.

```text
sftp>
```

SFTP 접속도 SSH를 사용하기 때문에 최초 접속 시 서버의 SSH Host Key를 확인하게 된다.

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

`yes`를 입력하면 해당 서버의 Host Key 정보가 Client의 `known_hosts`에 저장된다.

이후 같은 서버에 다시 접속할 때 서버의 신원을 확인하는 데 사용된다.

---

# Local과 Remote

## 6. Local과 Remote의 의미

SFTP를 이해할 때 가장 중요한 개념 중 하나가 Local과 Remote이다.

현재 실습 환경에서는 다음과 같다.

```text
Local                   Remote
-----                   ------
Windows                 Rocky Linux
Client                  SFTP Server
C:\Users\이병규         /home/sftpuser 또는 Chroot 영역
```

즉:

```text
Local  = 내가 SFTP 명령을 실행하고 있는 Client
Remote = 접속한 SFTP Server
```

---

## 7. pwd와 lpwd

### Remote 현재 경로 확인

```text
sftp> pwd
```

`pwd`는 서버의 현재 작업 디렉터리를 확인한다.

### Local 현재 경로 확인

```text
sftp> lpwd
```

`lpwd`는 Client의 현재 작업 디렉터리를 확인한다.

정리:

```text
pwd  → Remote 경로 확인
lpwd → Local 경로 확인
```

---

## 8. cd와 lcd

### Remote 디렉터리 이동

```text
sftp> cd 디렉터리
```

예:

```text
sftp> cd upload
```

### Local 디렉터리 이동

```text
sftp> lcd 디렉터리
```

예:

```text
sftp> lcd C:\Users\이병규
```

정리:

```text
cd  → Remote 디렉터리 이동
lcd → Local 디렉터리 이동
```

`cd`와 `lcd`는 서로 다른 시스템의 현재 위치를 변경하기 때문에 서로 독립적이다.

따라서 다음 두 순서는 모두 가능하다.

```text
lcd C:\Users\이병규
cd upload
```

또는:

```text
cd upload
lcd C:\Users\이병규
```

---

## 9. ls와 lls

### Remote 파일 확인

```text
sftp> ls
```

또는:

```text
sftp> ls -l
```

### Local 파일 확인

```text
sftp> lls
```

정리:

```text
ls  → Remote 파일 확인
lls → Local 파일 확인
```

SFTP 명령 앞의 `l`은 Local을 의미한다고 생각하면 이해하기 쉽다.

```text
pwd  ↔ lpwd
cd   ↔ lcd
ls   ↔ lls
```

---

# 파일 전송

## 10. put - 파일 업로드

`put`은 Local의 파일을 Remote 서버로 전송한다.

```text
Local → Remote
      put
```

기본 형식:

```text
sftp> put 파일명
```

예:

```text
sftp> put upload-test.txt
```

현재 Remote 디렉터리에 파일이 업로드된다.

---

## 11. 업로드할 Local 파일의 위치

`put upload-test.txt`라고 입력하면 SFTP Client는 현재 Local 작업 디렉터리에서 `upload-test.txt`를 찾는다.

따라서 다음 명령으로 Local 위치를 확인하는 것이 중요하다.

```text
sftp> lpwd
```

필요하면 `lcd`로 이동한다.

```text
sftp> lcd C:\Users\이병규
```

그 후:

```text
sftp> put upload-test.txt
```

업로드할 수 있다.

즉:

```text
lpwd
 ↓
Local 현재 위치 확인
 ↓
lcd
 ↓
업로드 파일이 있는 위치로 이동
 ↓
put
```

SFTP 접속 후에도 `lcd`를 이용하여 Local 경로를 변경할 수 있다.

---

## 12. Remote 목적지 지정

반드시 `cd`로 이동한 뒤 업로드해야 하는 것은 아니다.

현재 Remote 위치에서:

```text
sftp> put upload-test.txt /upload/
```

처럼 목적지 경로를 직접 지정할 수도 있다.

두 방법 모두 가능하다.

### 방법 1 - Remote 디렉터리 이동 후 업로드

```text
sftp> cd upload
sftp> put upload-test.txt
```

### 방법 2 - Remote 경로 직접 지정

```text
sftp> put upload-test.txt /upload/
```

---

## 13. get - 파일 다운로드

`get`은 Remote 서버의 파일을 Local Client로 다운로드한다.

```text
Remote → Local
       get
```

기본 형식:

```text
sftp> get 파일명
```

예:

```text
sftp> get server-test.txt
```

현재 Local 디렉터리에 저장된다.

다운로드 위치를 변경하고 싶으면 먼저 `lcd`를 사용한다.

```text
sftp> lcd Downloads
sftp> get server-test.txt
```

---

## 14. put과 get 비교

```text
Windows(Local)                      Rocky Linux(Remote)

upload-test.txt   ───── put ─────▶  upload-test.txt

server-test.txt   ◀──── get ──────  server-test.txt
```

정리:

```text
put → Local → Remote
get → Remote → Local
```

---

## 15. 와일드카드를 이용한 전송

여러 파일을 한 번에 전송할 때 와일드카드를 사용할 수 있다.

예:

```text
sftp> put sol-a*
```

`sol-a`로 시작하는 여러 Local 파일을 업로드한다.

```text
sftp> get /temp/host*
```

Remote의 `/temp`에서 `host`로 시작하는 파일들을 다운로드한다.

다른 예:

```text
put *.txt
put *log*
get *.conf
```

---

# SFTP와 Linux 권한

## 16. SFTP도 Linux 파일 권한의 영향을 받는다

SFTP로 접속했다고 해서 Linux 파일 권한을 무시할 수 있는 것은 아니다.

SFTP 사용자는 Linux 사용자이므로 다음 요소의 영향을 받는다.

```text
User
Group
Owner
Permission
Directory Permission
```

예를 들어:

```text
drwx------ sftpuser sftpuser /home/sftpuser
```

권한이 `700`이라면:

```text
소유자(sftpuser) → rwx
그룹             → ---
기타 사용자      → ---
```

따라서 다른 일반 사용자는 해당 디렉터리에 접근할 수 없다.

---

## 17. 업로드 파일의 소유권

SFTP를 이용하여 `sftpuser`가 파일을 업로드하면 일반적으로 파일은 해당 사용자 소유로 생성된다.

실습에서는 다음과 같이 확인했다.

```text
-rw-r--r-- sftpuser sftpuser upload-test.txt
```

즉:

```text
Owner : sftpuser
Group : sftpuser
Mode  : 644
```

실제 파일 권한은 서버에서 다음 명령으로 확인할 수 있다.

```bash
ls -l 파일명
stat 파일명
```

---

# OpenSSH와 SFTP

## 18. SFTP Subsystem

OpenSSH의 `sshd_config`에는 SFTP 기능을 제공하기 위한 `Subsystem` 설정이 존재할 수 있다.

실습 환경에서는 다음 설정을 확인했다.

```text
Subsystem sftp /usr/libexec/openssh/sftp-server
```

의미:

```text
SFTP 요청
   ↓
sshd
   ↓
SFTP Subsystem
   ↓
/usr/libexec/openssh/sftp-server
```

---

## 19. internal-sftp

`internal-sftp`는 외부 `sftp-server` 프로그램을 실행하지 않고 `sshd` 내부에 포함된 SFTP 기능을 사용하는 방식이다.

예:

```text
ForceCommand internal-sftp
```

특정 사용자를 SFTP 전용으로 제한할 때 사용할 수 있다.

---

# SFTP-only 사용자

## 20. 일반 SSH와 SFTP

기본 OpenSSH 사용자라면 같은 계정으로 다음 두 가지가 모두 가능할 수 있다.

```text
SSH Shell
SFTP
```

예:

```bash
ssh sftpuser@192.168.111.100
```

또는:

```bash
sftp sftpuser@192.168.111.100
```

하지만 파일 전송 전용 계정이라면 일반 Shell까지 사용할 필요가 없을 수 있다.

이때 `Match`와 `ForceCommand`를 이용하여 특정 사용자를 SFTP 전용으로 제한할 수 있다.

---

## 21. Match User

`Match`는 특정 조건에 해당하는 SSH 사용자에게 별도의 설정을 적용할 때 사용한다.

예:

```text
Match User sftpuser
```

의미:

```text
아래의 설정은 sftpuser에게만 적용
```

다른 SSH 사용자에게는 해당 `Match User` 설정이 적용되지 않는다.

---

## 22. ForceCommand internal-sftp

```text
ForceCommand internal-sftp
```

해당 사용자에게 일반 Shell 대신 SFTP 기능만 실행하도록 강제한다.

구성 후 일반 SSH 접속을 시도하면 실습에서는 다음과 같이 Shell 접속이 차단되었다.

```text
This service allows sftp connections only.
```

하지만:

```bash
sftp sftpuser@서버IP
```

SFTP 접속은 가능하다.

즉:

```text
SSH 인증
   ↓
sftpuser인가?
   ↓
ForceCommand internal-sftp
   ↓
일반 Shell X
SFTP       O
```

---

# Chroot

## 23. ChrootDirectory란?

`ChrootDirectory`는 사용자가 볼 수 있는 파일 시스템 범위를 특정 디렉터리로 제한하는 데 사용할 수 있다.

실습 설정:

```text
ChrootDirectory /sftp/%u
```

`%u`는 사용자명을 의미한다.

`sftpuser`가 접속하면:

```text
/sftp/%u
       ↓
/sftp/sftpuser
```

가 Chroot 기준 경로가 된다.

---

## 24. Chroot에서 보이는 경로

실제 서버의 경로:

```text
/sftp/sftpuser
```

SFTP 사용자에게는:

```text
/
```

로 보인다.

따라서:

```text
실제 Rocky Linux 경로             SFTP 사용자에게 보이는 경로

/sftp/sftpuser             →     /
/sftp/sftpuser/upload      →     /upload
```

SFTP 사용자는 Chroot 밖의 시스템 경로를 일반적인 SFTP 경로 이동으로 탐색할 수 없다.

---

## 25. Chroot 디렉터리 소유권

OpenSSH Chroot를 사용할 때 Chroot 기준 디렉터리는 일반 사용자가 마음대로 수정할 수 없도록 구성해야 한다.

실습에서는:

```text
/sftp
└── sftpuser
    └── upload
```

소유권을 다음과 같이 구성했다.

```text
/sftp                  root:root
/sftp/sftpuser         root:root
/sftp/sftpuser/upload  sftpuser:sftpuser
```

권한:

```text
/sftp                  755
/sftp/sftpuser         755
/sftp/sftpuser/upload  755
```

핵심 구조:

```text
Chroot 최상위
/sftp/sftpuser
      │
      └── root 소유
           │
           └── upload
                 └── sftpuser 소유
```

즉 사용자는 Chroot 자체를 수정하는 것이 아니라 별도의 쓰기 가능한 하위 디렉터리에서 파일을 관리한다.

---

## 26. SFTP-only 설정 예

실습에서 사용한 설정:

```text
Match User sftpuser
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

각 설정의 의미:

| 설정 | 의미 |
|---|---|
| `Match User sftpuser` | 특정 사용자에게만 설정 적용 |
| `ChrootDirectory /sftp/%u` | 사용자별 Chroot 지정 |
| `ForceCommand internal-sftp` | SFTP만 사용하도록 제한 |
| `AllowTcpForwarding no` | TCP Forwarding 차단 |
| `X11Forwarding no` | X11 Forwarding 차단 |

---

## 27. sshd 설정 변경 시 주의

`/etc/ssh/sshd_config`을 잘못 수정하면 SSH 접속 자체에 문제가 발생할 수 있다.

따라서 설정 변경 전 백업하는 것이 안전하다.

```bash
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.sftp-backup
```

설정 변경 후에는 먼저 문법을 검사한다.

```bash
sshd -t
```

문법에 문제가 없다면 일반적으로 오류가 출력되지 않는다.

그 후 설정을 적용한다.

```bash
systemctl reload sshd
```

원격 SSH 환경에서 설정을 변경할 때는 문제가 발생했을 때 복구할 수 있도록 기존 관리자 SSH 세션을 유지한 상태에서 검증하는 것이 안전하다.

---

# 보안 관련 설정

## 28. AllowTcpForwarding

```text
AllowTcpForwarding no
```

해당 사용자가 SSH TCP Forwarding 기능을 사용하지 못하도록 제한한다.

파일 전송 전용 사용자에게 필요하지 않은 SSH 기능을 줄이는 데 사용할 수 있다.

---

## 29. X11Forwarding

```text
X11Forwarding no
```

해당 사용자의 X11 Forwarding을 차단한다.

SFTP 전용 사용자에게 불필요한 기능을 제한하는 목적이다.

---

# 방화벽

## 30. SFTP와 Firewall

SFTP는 SSH를 사용하므로 기본적으로 TCP 22번 포트가 필요하다.

Rocky Linux의 firewalld에서는 다음과 같이 SSH 서비스 허용 상태를 확인할 수 있다.

```bash
firewall-cmd --list-services
```

예:

```text
ssh
```

`ssh` 서비스가 허용되어 있다면 기본 SSH/SFTP TCP 22번 통신이 가능하다.

FTP의 TCP 20/21번 포트를 SFTP 때문에 별도로 개방할 필요는 없다.

---

# 로그

## 31. SSH/SFTP 인증 로그

SFTP는 SSH를 기반으로 동작하기 때문에 SSH 인증 로그에서 접속 기록을 확인할 수 있다.

예:

```text
Accepted password for sftpuser ...
Failed password for sftpuser ...
```

이를 통해 다음 내용을 확인할 수 있다.

```text
접속 사용자
Client IP
인증 성공
인증 실패
Session 생성
```

Rocky Linux에서는 `journalctl`을 이용하여 `sshd` 관련 로그를 확인할 수 있다.

---

# 주요 SFTP 명령어

## 32. 명령어 정리

| 명령어 | 의미 |
|---|---|
| `pwd` | Remote 현재 경로 확인 |
| `lpwd` | Local 현재 경로 확인 |
| `ls` | Remote 파일 확인 |
| `lls` | Local 파일 확인 |
| `cd` | Remote 디렉터리 이동 |
| `lcd` | Local 디렉터리 이동 |
| `mkdir` | Remote 디렉터리 생성 |
| `put` | Local → Remote 업로드 |
| `get` | Remote → Local 다운로드 |
| `exit` | SFTP 종료 |

가장 중요한 구분:

```text
Remote 명령       Local 명령

pwd               lpwd
ls                lls
cd                lcd
```

---

# 전체 동작 구조

## 33. 일반 SFTP

```text
Windows Client
      │
      │ SFTP / SSH / TCP 22
      ▼
    sshd
      │
      ▼
SFTP Subsystem
      │
      ▼
Linux File System
```

---

## 34. SFTP-only + Chroot

```text
Windows Client
      │
      │ TCP 22
      ▼
    sshd
      │
      ├── Match User sftpuser
      │
      ├── ForceCommand internal-sftp
      │
      └── ChrootDirectory /sftp/%u
                    │
                    ▼
             /sftp/sftpuser
                    │
                    └── upload/
```

결과:

```text
일반 SSH Shell    → 차단
SFTP              → 허용
시스템 전체 탐색   → 제한
/upload 파일 전송 → 허용
```

---

# 핵심 정리

## 35. SFTP 핵심

1. SFTP는 FTP와 다른 프로토콜이다.
2. SFTP는 SSH를 기반으로 동작한다.
3. 기본적으로 TCP 22번 포트를 사용한다.
4. `put`은 Local에서 Remote로 업로드한다.
5. `get`은 Remote에서 Local로 다운로드한다.
6. `pwd`, `cd`, `ls`는 Remote를 대상으로 한다.
7. `lpwd`, `lcd`, `lls`는 Local을 대상으로 한다.
8. SFTP 사용자도 Linux 파일 권한의 영향을 받는다.
9. `ForceCommand internal-sftp`를 이용하여 SFTP 전용 사용자를 구성할 수 있다.
10. `ChrootDirectory`를 이용하여 사용자의 접근 범위를 제한할 수 있다.
11. Chroot 최상위는 `root`가 관리하고 별도의 하위 업로드 디렉터리에 사용자 쓰기 권한을 부여할 수 있다.
12. SFTP 인증 기록은 SSH 로그를 통해 확인할 수 있다.
