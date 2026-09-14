# SSH Security Hardening 이론 정리

## 1. SSH란?

SSH는 Secure Shell의 약자이다.

Network를 통해 원격 Server에 안전하게 접속하고 명령을 실행하기 위한 Protocol이다.

대표적인 구조:

```text
Client
   |
   | SSH
   | TCP 22
   v
Server
```

SSH를 이용하면 원격에서:

```text
Shell Login
Command 실행
File 전송
Port Forwarding
```

등을 수행할 수 있다.

---

# OpenSSH

## 2. OpenSSH란?

OpenSSH는 SSH Protocol을 구현한 Software이다.

Rocky Linux에서는 대표적으로 다음 Package를 사용한다.

```text
openssh-server
openssh-clients
```

---

## 3. openssh-server

```text
openssh-server
```

는 SSH Server 기능을 제공한다.

주요 Daemon:

```text
sshd
```

이다.

---

## 4. openssh-clients

```text
openssh-clients
```

는 SSH Client 명령어를 제공한다.

대표적으로:

```bash
ssh
scp
sftp
ssh-keygen
ssh-copy-id
```

등을 사용할 수 있다.

---

# ssh와 sshd

## 5. ssh

```text
ssh
```

는 Client가 Server에 접속할 때 사용하는 명령이다.

예:

```bash
ssh sysadmin@192.168.111.100
```

구조:

```text
Client
   |
   | ssh
   v
Server
```

---

## 6. sshd

```text
sshd
```

는 Server에서 SSH 연결 요청을 받아 처리하는 Daemon이다.

Rocky Linux Service:

```text
sshd.service
```

확인:

```bash
systemctl is-active sshd
systemctl is-enabled sshd
```

---

# SSH Port

## 7. TCP 22

SSH의 기본 Port는:

```text
TCP 22
```

이다.

확인:

```bash
ss -lntp | grep ':22'
```

이번 실습에서는:

```text
0.0.0.0:22
[::]:22
```

에서 `sshd`가 Listen 중이었다.

---

# SSH 인증

## 8. SSH Login 과정

SSH 접속은 단순히 TCP 연결만 되면 끝나는 것이 아니다.

대략적인 과정:

```text
Client
   ↓
TCP 22 연결
   ↓
SSH Protocol 협상
   ↓
Server Host 확인
   ↓
사용자 인증
   ↓
인증 성공
   ↓
Session 생성
   ↓
Shell / SFTP 등 사용
```

---

# Password Authentication

## 9. Password 인증

가장 익숙한 인증 방식이다.

```text
Client
   ↓
Username 입력
   ↓
Password 입력
   ↓
Server에서 확인
   ↓
Login 허용 / 거부
```

예:

```bash
ssh sysadmin@192.168.111.100
```

Password 방식은 사용하기 간단하지만 인터넷에 공개된 SSH Server에서는 반복적인 Password 추측 공격의 대상이 될 수 있다.

따라서 관리자 SSH 접근에서는 Public Key 인증을 많이 사용한다.

---

# Public Key Authentication

## 10. SSH Key 인증

SSH Key 인증에서는 한 쌍의 Key를 사용한다.

```text
Private Key
+
Public Key
```

이번 실습에서는 ED25519 Key를 사용하였다.

```bash
ssh-keygen -t ed25519
```

---

## 11. Private Key

Private Key는 Client가 보관한다.

이번 실습:

```text
/root/.ssh/id_ed25519_server_a
```

Private Key는 인증의 핵심 비밀 정보이다.

따라서:

```text
GitHub 업로드 X
다른 사용자에게 전달 X
공개 저장소 저장 X
Screenshot 노출 X
```

해야 한다.

---

## 12. Public Key

Public Key는 Server에 등록한다.

이번 실습:

```text
/root/.ssh/id_ed25519_server_a.pub
```

Public Key는 Private Key와 달리 Server에 전달하여 인증에 사용할 수 있다.

---

# Key Pair 구조

## 13. 전체 구조

```text
Client-L
/root/.ssh/id_ed25519_server_a
        |
        | Private Key
        |
        | SSH 인증
        v
Server-A
/home/sysadmin/.ssh/authorized_keys
        |
        | Public Key 등록
        v
인증 성공 여부 판단
```

---

# authorized_keys

## 14. authorized_keys란?

Server에서 해당 사용자에게 허용된 SSH Public Key 목록을 저장하는 파일이다.

이번 실습:

```text
/home/sysadmin/.ssh/authorized_keys
```

Client의 Public Key가 여기에 등록되어 있어야 Key 인증이 가능하다.

---

## 15. Public Key 등록

이번 실습:

```bash
ssh-copy-id -i /root/.ssh/id_ed25519_server_a.pub sysadmin@192.168.111.100
```

`ssh-copy-id`를 이용하여 Client의 Public Key를 Server에 등록하였다.

구조:

```text
Client Public Key
        ↓
ssh-copy-id
        ↓
Server
~/.ssh/authorized_keys
```

---

# SSH Key Permission

## 16. .ssh Directory

이번 실습:

```text
/home/sysadmin/.ssh
```

Permission:

```text
700
```

실제:

```text
drwx------
```

이다.

---

## 17. authorized_keys

Permission:

```text
600
```

실제:

```text
-rw-------
```

이다.

Owner:

```text
sysadmin:sysadmin
```

으로 설정되어 있었다.

---

## 18. Permission이 중요한 이유

SSH는 인증 관련 File의 Permission이 지나치게 개방되어 있으면 보안상 안전하지 않다고 판단하여 Key 인증이 실패할 수 있다.

따라서 일반적으로 다음을 확인한다.

```text
~/.ssh
→ 사용자 소유
→ 700

authorized_keys
→ 사용자 소유
→ 600
```

---

# ED25519

## 19. ED25519 Key

이번 실습에서는:

```bash
ssh-keygen -t ed25519
```

를 사용하였다.

ED25519는 현대 SSH 환경에서 널리 사용되는 Public Key Algorithm 중 하나이다.

생성:

```text
Private Key
id_ed25519_server_a

Public Key
id_ed25519_server_a.pub
```

---

# 관리자 계정 분리

## 20. 왜 root로 직접 접속하지 않는가?

초기 환경:

```text
Client
   ↓
root
   ↓
Server
```

에서는 원격 로그인 계정 자체가 최고 관리자 계정이다.

보안 강화 후:

```text
Client
   ↓
sysadmin
   ↓
sudo
   ↓
root 권한
```

구조로 변경하였다.

---

## 21. 관리자 계정

이번 실습에서는:

```text
sysadmin
```

을 별도의 관리자 계정으로 사용하였다.

```bash
useradd -m -s /bin/bash sysadmin
```

---

# wheel Group

## 22. wheel

Rocky Linux에서는 `wheel` Group을 관리자 권한과 연결하여 사용하는 경우가 많다.

이번 실습:

```bash
usermod -aG wheel sysadmin
```

확인:

```bash
id sysadmin
```

실제:

```text
groups=1011(sysadmin),10(wheel)
```

---

# sudo

## 23. sudo란?

일반 사용자가 허용된 범위에서 다른 사용자의 권한으로 Command를 실행할 수 있도록 해주는 기능이다.

이번 실습:

```bash
sudo whoami
```

결과:

```text
root
```

즉:

```text
SSH Login
→ sysadmin

관리 작업
→ sudo

필요한 경우
→ root 권한
```

구조를 사용할 수 있다.

---

# PermitRootLogin

## 24. root SSH 로그인 정책

설정:

```conf
PermitRootLogin no
```

의미:

```text
root 계정의 SSH 직접 Login 차단
```

이다.

---

## 25. root 계정 자체가 삭제되는 것은 아니다

중요하다.

```conf
PermitRootLogin no
```

라고 해서 root 계정 자체가 비활성화되는 것은 아니다.

다음은 계속 가능하다.

```bash
sudo -i
```

또는 필요한 명령에:

```bash
sudo command
```

을 사용할 수 있다.

즉:

```text
root SSH 직접 Login
→ X

root 권한 사용
→ O
```

이다.

---

# PubkeyAuthentication

## 26. Public Key 인증 허용

```conf
PubkeyAuthentication yes
```

SSH Public Key 인증을 허용한다.

이번 `sysadmin` 관리자 접속의 핵심 인증 방식이다.

---

# PasswordAuthentication

## 27. Password 인증

```conf
PasswordAuthentication yes
```

이면 Password 기반 SSH 인증을 허용한다.

```conf
PasswordAuthentication no
```

이면 Password 기반 SSH 인증을 차단한다.

---

## 28. 이번 실습에서는 전역 차단하지 않았다

기존에:

```text
sftpuser
```

가 Password 기반 SFTP를 사용하고 있었다.

따라서 전역으로:

```conf
PasswordAuthentication no
```

를 설정하면 기존 SFTP 환경에 영향을 줄 수 있다.

이번에는 사용자별로:

```conf
Match User sysadmin
    PasswordAuthentication no
```

를 사용하였다.

---

## 29. 최종 인증 정책

```text
sysadmin

Public Key
→ O

Password
→ X
```

```text
sftpuser

Password
→ O

SFTP
→ O
```

---

# Match User

## 30. Match Block

OpenSSH에서는 특정 조건에 맞는 Connection에 별도의 설정을 적용할 수 있다.

예:

```conf
Match User sysadmin
    PasswordAuthentication no
```

의미:

```text
접속 사용자가 sysadmin이면
PasswordAuthentication no 적용
```

이다.

---

## 31. 기존 sftpuser Match

기존 설정:

```conf
Match User sftpuser
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

의미:

```text
sftpuser에게만
해당 설정 적용
```

이다.

---

# Match all

## 32. Match all

```conf
Match all
```

은 특정 `Match` 조건 영역을 벗어나 다시 일반적인 Match Context로 전환할 때 사용할 수 있다.

SSH 설정에서는 `Match` Block 이후의 Directive가 의도하지 않게 특정 사용자에게만 적용되지 않도록 설정 위치를 주의해야 한다.

---

# ForceCommand

## 33. ForceCommand internal-sftp

기존 SFTP 설정:

```conf
ForceCommand internal-sftp
```

`sftpuser`가 SSH로 접속하더라도 일반 Shell 대신 내부 SFTP 기능만 실행하도록 강제한다.

즉:

```text
sftpuser
        ↓
SSH Service 접속
        ↓
일반 Shell X
        ↓
SFTP 실행
```

이다.

---

# ChrootDirectory

## 34. Chroot

설정:

```conf
ChrootDirectory /sftp/%u
```

사용자가 볼 수 있는 File System 영역을 제한한다.

`sftpuser`의 경우:

```text
/sftp/sftpuser
```

영역이 가상의 `/`처럼 보이게 된다.

실제 SFTP에서:

```text
Remote working directory: /
```

로 보였지만 실제 Server 전체 `/`를 의미하는 것은 아니다.

---

# AllowUsers

## 35. SSH 접근 사용자 제한

이번 설정:

```conf
AllowUsers sysadmin sftpuser
```

의미:

```text
SSH Service 사용 허용

sysadmin
sftpuser
```

이다.

목록에 없는 사용자는 SSH 접근이 거부된다.

---

## 36. root가 차단된 이유

이번 Log에는:

```text
User root from 192.168.111.150 not allowed because not listed in AllowUsers
```

가 기록되었다.

즉 root는:

```text
PermitRootLogin no
```

뿐 아니라:

```text
AllowUsers
```

목록에도 포함되지 않았다.

따라서 여러 정책을 통해 SSH 직접 접근이 제한된 상태이다.

---

# MaxAuthTries

## 37. 인증 시도 횟수

초기:

```text
MaxAuthTries 6
```

Hardening:

```conf
MaxAuthTries 3
```

한 Connection에서 허용되는 인증 시도 횟수를 줄였다.

---

## 38. 실제 검증

root Password 인증을 반복해서 시도한 결과:

```text
Too many authentication failures
```

가 발생하였다.

Server Log:

```text
maximum authentication attempts exceeded
```

도 확인하였다.

즉:

```text
MaxAuthTries 3
```

설정이 실제 동작하였다.

---

# PermitEmptyPasswords

## 39. 빈 Password 제한

```conf
PermitEmptyPasswords no
```

Password가 설정되지 않은 계정의 SSH Password 인증을 허용하지 않는다.

---

# X11Forwarding

## 40. X11 Forwarding이란?

SSH 연결을 통해 원격 Linux GUI Application의 X11 화면을 Client로 전달할 수 있는 기능이다.

구조:

```text
Remote GUI Application
        ↓
SSH Tunnel
        ↓
Client X Server
```

Server에서 GUI 기능을 사용하지 않는다면 불필요한 기능을 비활성화할 수 있다.

이번 설정:

```conf
X11Forwarding no
```

---

# AllowTcpForwarding

## 41. SSH TCP Forwarding

SSH는 Tunnel 기능을 통해 다른 TCP Connection을 전달할 수 있다.

예:

```text
Local Port Forwarding
Remote Port Forwarding
Dynamic Port Forwarding
```

이번 Server 관리 환경에서는 해당 기능을 사용하지 않기 때문에:

```conf
AllowTcpForwarding no
```

로 설정하였다.

---

## 42. 무조건 꺼야 하는 옵션은 아니다

중요하다.

```text
X11Forwarding
AllowTcpForwarding
```

은 항상 무조건 비활성화해야 한다는 의미는 아니다.

운영 목적에 따라 필요한 기능이라면 사용할 수 있다.

Hardening의 핵심은:

```text
사용하지 않는 기능
→ 비활성화
```

하는 것이다.

---

# sshd_config

## 43. SSH Server 기본 설정 파일

주요 설정 파일:

```text
/etc/ssh/sshd_config
```

이다.

Rocky Linux에서는 추가 설정을:

```text
/etc/ssh/sshd_config.d/*.conf
```

에서도 읽을 수 있다.

이번 실습에서도:

```text
01-permitrootlogin.conf
02-hardening.conf
50-redhat.conf
```

형태의 설정을 사용하였다.

---

# Include

## 44. sshd_config.d

기본 설정:

```conf
Include /etc/ssh/sshd_config.d/*.conf
```

을 통해 별도 Configuration File을 불러올 수 있다.

장점:

```text
기본 설정 File 전체 수정 감소
설정 목적별 분리
관리 편의성 증가
```

이다.

---

# sshd -t

## 45. 문법 검사

```bash
sshd -t
```

SSH Server Configuration의 Syntax를 검사한다.

정상:

```text
아무 출력 없음
```

오류가 있으면 Error가 출력된다.

---

## 46. 왜 먼저 sshd -t를 하는가?

SSH 설정은 잘못하면 원격 접속 자체가 불가능해질 수 있다.

따라서:

```text
설정 수정
        ↓
sshd -t
        ↓
문법 정상 확인
        ↓
Reload
```

순서로 작업하는 것이 안전하다.

---

# sshd -T

## 47. 실제 적용 설정 확인

```bash
sshd -T
```

는 sshd가 해석한 최종 Configuration 값을 확인하는 데 사용할 수 있다.

예:

```bash
sshd -T | grep permitrootlogin
```

---

# sshd -T -C

## 48. 특정 Connection 조건 적용값 확인

`Match User` 등이 존재하면 일반 `sshd -T`만으로 사용자별 차이를 확인하기 어려울 수 있다.

이때:

```bash
sshd -T -C user=sysadmin,host=Server-A,addr=192.168.111.150
```

형태를 사용할 수 있다.

---

## 49. sysadmin 검증

이번 실제 결과:

```text
maxauthtries 3
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
x11forwarding no
allowtcpforwarding no
allowusers sysadmin
allowusers sftpuser
```

즉 `sysadmin`에게 실제 적용될 설정을 Service 적용 전에 확인하였다.

---

## 50. sftpuser 검증

실제:

```text
passwordauthentication yes
x11forwarding no
allowtcpforwarding no
forcecommand internal-sftp
chrootdirectory /sftp/%u
```

기존 SFTP 설정도 그대로 적용되는 것을 확인하였다.

---

# Reload와 Restart

## 51. reload

이번 실습:

```bash
systemctl reload sshd
```

를 이용하여 Configuration을 다시 읽도록 하였다.

설정 변경 시 기존 Connection을 유지하면서 새로운 설정을 반영하는 데 사용할 수 있다.

---

## 52. 기존 Session을 유지하는 이유

SSH Hardening 중 가장 중요한 안전 원칙 중 하나이다.

기존 SSH Session을 종료한 뒤 새 설정이 잘못되었음을 발견하면:

```text
새 Login 실패
+
기존 Session 없음
=
원격 관리 불가능
```

상황이 발생할 수 있다.

따라서:

```text
기존 관리 Session 유지
        ↓
설정 변경
        ↓
새로운 SSH Session으로 접속 검증
        ↓
성공 확인
        ↓
기존 Session 종료
```

순서가 안전하다.

---

# SSH Lockout

## 53. Lockout이란?

관리자가 자신의 SSH 접근을 차단하여 Server에 원격으로 접속하지 못하는 상태를 의미한다.

발생 예:

```text
PasswordAuthentication no
        +
Public Key 등록 실패
        ↓
로그인 방법 없음
```

또는:

```text
AllowUsers 설정 오류
        ↓
관리자 계정 누락
        ↓
SSH Login 불가
```

---

# 안전한 Hardening 순서

## 54. 추천 순서

이번 실습에서 사용한 안전한 순서:

```text
1. 현재 SSH 상태 확인

2. 일반 관리자 계정 생성

3. wheel / sudo 확인

4. 일반 계정 SSH Login 확인

5. SSH Key 생성

6. Public Key 등록

7. Public Key Login 검증

8. 기존 설정 Backup

9. Hardening 설정 작성

10. sshd -t

11. sshd -T -C

12. sshd Reload

13. 새로운 Session에서 Key Login 검증

14. Password Login 차단 검증

15. root Login 차단 검증

16. 기존 SFTP 기능 검증

17. Security Log 확인
```

---

# SSH 로그

## 55. /var/log/secure

Rocky Linux에서는 SSH 인증 관련 기록을:

```text
/var/log/secure
```

에서 확인할 수 있다.

예:

```bash
grep sshd /var/log/secure
```

---

# journalctl

## 56. sshd Journal

```bash
journalctl -u sshd
```

systemd Journal에서도 sshd 관련 Log를 확인할 수 있다.

---

# Public Key Login Log

## 57. 성공 기록

이번 실제 Log:

```text
Accepted publickey for sysadmin from 192.168.111.150
```

의미:

```text
Client
192.168.111.150

User
sysadmin

Authentication
Public Key

Result
Success
```

이다.

---

# Password Login 실패

## 58. sysadmin

Hardening 이후 Password 전용 접속:

```bash
ssh \
-o PubkeyAuthentication=no \
-o PreferredAuthentications=password \
sysadmin@192.168.111.100
```

결과:

```text
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

인증 방법 목록에:

```text
password
```

가 나타나지 않았다.

따라서 `sysadmin`에 대한 Password 인증이 비활성화된 것을 확인하였다.

---

# root Log

## 59. root 접근 거부

실제:

```text
User root from 192.168.111.150 not allowed because not listed in AllowUsers
```

root가 SSH 접근 허용 사용자 목록에 없음을 확인하였다.

---

# authentication failure

## 60. 인증 실패 기록

실제:

```text
authentication failure
```

및:

```text
Failed password
```

등이 기록되었다.

SSH 보안 문제나 무차별 대입 공격 등을 분석할 때 이러한 Log를 확인할 수 있다.

---

# Too many authentication failures

## 61. 인증 횟수 초과

실제:

```text
maximum authentication attempts exceeded
```

그리고 Client에서는:

```text
Too many authentication failures
```

가 확인되었다.

이번 설정:

```conf
MaxAuthTries 3
```

이 실제 동작한 결과이다.

---

# 기존 SFTP 유지

## 62. SFTP도 SSH를 사용한다

SFTP는 FTP Server를 사용하는 Protocol이 아니다.

SSH를 기반으로 File Transfer를 수행한다.

```text
SFTP
   ↓
SSH
   ↓
TCP 22
```

따라서 SSH Hardening 설정을 변경하면 기존 SFTP 환경에도 영향을 줄 수 있다.

---

## 63. 이번 실습에서 중요한 점

전역으로 모든 Password 인증을 차단하지 않고:

```text
sysadmin
→ Key Only

sftpuser
→ Password SFTP 유지
```

로 목적에 따라 인증 방식을 분리하였다.

---

# SSH와 SFTP 구조

## 64. 최종 구조

```text
                       sshd
                        |
             TCP 22     |
                        |
         ┌──────────────┴──────────────┐
         |                             |
      sysadmin                      sftpuser
         |                             |
    Public Key                      Password
         |                             |
     SSH Shell                     internal-sftp
         |                             |
       sudo                       ChrootDirectory
         |
       root 권한
```

---

# Password와 Key 인증 비교

## 65. Password

```text
장점
→ 사용이 간단함

주의점
→ Password 추측 공격 가능
→ Password 관리 필요
```

---

## 66. Public Key

```text
장점
→ Password를 Network Login에 직접 사용하지 않아도 됨
→ 자동화 환경에서도 활용 가능
→ 관리자 인증 강화 가능

주의점
→ Private Key 보호 필수
→ Key 분실 / 유출 관리 필요
```

---

# Private Key와 Public Key 차이

## 67. Private Key

```text
Client가 보관
절대 공개하면 안 됨
GitHub Commit 금지
```

---

## 68. Public Key

```text
Server authorized_keys에 등록
Private Key와 한 쌍
인증에 사용
```

---

# Key 인증 개념

## 69. 단순한 Password 비교가 아니다

SSH Key 인증은 Private Key 자체를 Server로 전송하여 비교하는 방식이 아니다.

개념적으로:

```text
Client
Private Key 보유
        ↓
암호학적 증명 수행
        ↓
Server
등록된 Public Key를 이용하여 검증
        ↓
인증 성공
```

한다.

따라서 Private Key는 Client 밖으로 노출하지 않는 것이 중요하다.

---

# Host Key와 User Key

## 70. SSH에는 다른 종류의 Key도 있다

SSH 접속 시 처음 나타나는:

```text
The authenticity of host ...
```

와 관련된 Key는 Server의 Host Key이다.

반면 이번에 만든:

```text
id_ed25519_server_a
id_ed25519_server_a.pub
```

는 사용자의 인증을 위한 User Key이다.

둘은 목적이 다르다.

---

## 71. Host Key

목적:

```text
접속하려는 Server가
예전에 접속했던 그 Server인지 확인
```

Client의:

```text
~/.ssh/known_hosts
```

등과 관련된다.

---

## 72. User Key

목적:

```text
접속하려는 사용자가
올바른 Private Key를 가지고 있는지 확인
```

Server의:

```text
~/.ssh/authorized_keys
```

와 관련된다.

---

# known_hosts와 authorized_keys

## 73. 구분

```text
known_hosts
→ Client가 Server를 확인

authorized_keys
→ Server가 Client/User 인증을 확인
```

쉽게 보면:

```text
Client → Server 신원 확인
known_hosts

Server → 사용자 인증
authorized_keys
```

이다.

---

# Hardening의 의미

## 74. Hardening이란?

Hardening은 System의 공격 가능 영역을 줄이고 보안 설정을 강화하는 작업이다.

SSH에서는 예를 들어:

```text
root 직접 로그인 차단
Password 인증 제한
인증 시도 횟수 제한
접근 사용자 제한
불필요한 Forwarding 차단
Security Log 확인
```

등을 수행할 수 있다.

---

# 최소 권한 원칙

## 75. Principle of Least Privilege

사용자에게 필요한 권한만 제공하는 개념이다.

이번 구조:

```text
sysadmin
→ 일반 사용자로 Login
→ 필요한 관리자 명령만 sudo

sftpuser
→ File Transfer만 사용
→ Shell Login 제한
```

은 역할에 따라 권한을 분리한 예이다.

---

# Defense in Depth

## 76. 여러 보안 정책을 함께 사용

이번 root 접근은 단일 설정만으로 제한하지 않았다.

```text
PermitRootLogin no
+
AllowUsers에 root 제외
```

처럼 여러 Layer를 사용할 수 있다.

이를 통해 하나의 정책만 의존하지 않는 구조를 만들 수 있다.

---

# Troubleshooting

## 77. SSH 연결 자체가 안 될 때

확인 순서:

```text
1. Network
2. sshd Service
3. TCP 22
4. Firewall
5. SSH Configuration
6. 사용자 계정
7. 인증 방식
8. Key File
9. Permission
10. Log
```

---

## 78. Network 확인

Client:

```bash
ping -c 3 192.168.111.100
```

Server와 기본 통신이 가능한지 확인한다.

---

## 79. Service 확인

Server:

```bash
systemctl is-active sshd
```

정상:

```text
active
```

---

## 80. Port 확인

```bash
ss -lntp | grep ':22'
```

TCP 22에서 `sshd`가 Listen 중인지 확인한다.

---

## 81. Firewall 확인

```bash
firewall-cmd --list-services
```

확인:

```text
ssh
```

---

## 82. 설정 문법 확인

```bash
sshd -t
```

오류가 있다면 Reload / Restart 전에 수정한다.

---

## 83. 실제 적용값 확인

```bash
sshd -T
```

또는 사용자 조건까지 포함:

```bash
sshd -T -C user=sysadmin,host=Server-A,addr=192.168.111.150
```

---

# Key Login 실패

## 84. 확인 항목

```text
Private Key 경로
Public Key 등록 여부
authorized_keys
.ssh Permission
File Owner
PubkeyAuthentication
AllowUsers
사용자 계정
SELinux
Log
```

---

## 85. Key를 명확하게 지정

Client:

```bash
ssh -i /root/.ssh/id_ed25519_server_a \
-o IdentitiesOnly=yes \
sysadmin@192.168.111.100
```

`IdentitiesOnly=yes`를 사용하면 지정한 Identity를 중심으로 인증을 시도하도록 할 수 있다.

---

# authorized_keys 문제

## 86. Permission 확인

Server:

```bash
ls -ld /home/sysadmin
ls -ld /home/sysadmin/.ssh
ls -l /home/sysadmin/.ssh/authorized_keys
```

확인:

```text
Owner
→ sysadmin

.ssh
→ 700

authorized_keys
→ 600
```

---

# Password가 계속 허용될 때

## 87. 실제 적용값부터 확인

```bash
sshd -T -C user=sysadmin,host=Server-A,addr=192.168.111.150 | grep passwordauthentication
```

정상 목표:

```text
passwordauthentication no
```

설정 파일에 `no`를 적었다는 사실보다 **sshd가 실제로 어떤 값을 적용하는지** 확인하는 것이 중요하다.

---

# root 접속이 계속 될 때

## 88. 확인

```bash
sshd -T | grep permitrootlogin
```

그리고:

```bash
sshd -T | grep allowusers
```

또는 실제 Connection 조건으로 확인한다.

---

# Match 설정 문제

## 89. Match Block 위치 확인

SSH Configuration에서 `Match` 이후에 작성된 설정은 해당 조건의 영향을 받을 수 있다.

따라서:

```text
어느 Directive가
어느 Match Block에 포함되는지
```

를 확인해야 한다.

특히 기존:

```conf
Match User sftpuser
```

와 신규:

```conf
Match User sysadmin
```

가 함께 존재하므로 사용자별 `sshd -T -C` 확인이 중요하다.

---

# Log를 너무 많이 볼 필요는 없다

## 90. 필요한 항목만 검색

전체 Log를 한 번에 보는 대신 필요한 Event만 검색할 수 있다.

이번 실습 예:

```bash
grep -E 'Accepted publickey|not allowed|maximum authentication|Accepted password for sftpuser' /var/log/secure | tail -n 10
```

이렇게 하면 핵심 Event만 빠르게 확인할 수 있다.

---

# SSH Debug

## 91. Client 상세 Debug

SSH Login 문제가 있을 때 Client에서:

```bash
ssh -v sysadmin@192.168.111.100
```

을 사용할 수 있다.

더 많은 정보:

```bash
ssh -vv sysadmin@192.168.111.100
```

또는:

```bash
ssh -vvv sysadmin@192.168.111.100
```

을 사용할 수 있다.

확인 가능:

```text
어떤 Key를 시도했는지
어떤 인증 방식을 제안받았는지
Authentication 진행 과정
Connection 과정
```

---

# SSH Hardening 시 주의사항

## 92. 무조건 설정을 복사하면 안 된다

SSH Hardening은 Server 용도에 따라 달라진다.

예:

```text
SSH Tunnel이 필요한 Server
→ AllowTcpForwarding 필요할 수 있음

GUI Application을 사용하는 환경
→ X11Forwarding 필요할 수 있음

SFTP Password 계정이 필요한 환경
→ PasswordAuthentication 전역 차단 시 영향 발생
```

따라서:

```text
필요한 기능 확인
        ↓
불필요한 기능 제한
```

방식으로 접근해야 한다.

---

# PasswordAuthentication 전역 차단 주의

## 93. 다른 사용자 영향

예를 들어:

```conf
PasswordAuthentication no
```

를 전역으로 적용하면:

```text
sysadmin
→ Password X

sftpuser
→ Password X
```

가 될 수 있다.

이번 환경에서는 기존 `sftpuser`가 Password SFTP를 사용해야 했기 때문에 사용자별 정책을 적용하였다.

---

# 설정 변경 전 Backup

## 94. Backup

이번 실습:

```bash
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.before-hardening
```

그리고:

```bash
cp -a \
/etc/ssh/sshd_config.d/01-permitrootlogin.conf \
/etc/ssh/sshd_config.d/01-permitrootlogin.conf.before-hardening
```

SSH처럼 원격 관리에 직접 영향을 미치는 설정은 변경 전에 Backup을 만드는 것이 좋다.

---

# 보안 설정과 실제 검증

## 95. 설정만으로 끝내면 안 된다

예:

```conf
PermitRootLogin no
```

를 작성했다고 해서 바로 실습 완료라고 판단하면 안 된다.

실제로:

```text
root 접속 시도
→ 실패 확인
```

해야 한다.

---

## 96. 이번 검증 방식

```text
설정
PermitRootLogin no

실제 접속
root@Server-A

결과
Permission denied
```

---

## 97. PasswordAuthentication 검증

```text
설정
sysadmin PasswordAuthentication no

실제
Public Key 사용 금지
Password 방식만 강제

결과
Permission denied
```

---

## 98. Public Key 검증

```text
Public Key만 사용하도록 강제
        ↓
sysadmin Login 성공
```

따라서 실제 Key 인증이 정상 동작함을 확인하였다.

---

## 99. SFTP Regression Test

SSH Hardening 후 기존 기능도 다시 확인하였다.

```text
sftpuser
        ↓
Password Login
        ↓
SFTP 접속
        ↓
ls
pwd
        ↓
정상
```

새로운 설정 때문에 기존 기능이 깨지지 않았는지 확인하는 것을 Regression Test라고 볼 수 있다.

---

# 주요 명령어

## 100. SSH Service

```bash
systemctl is-active sshd
systemctl is-enabled sshd
```

---

## 101. Port

```bash
ss -lntp | grep ':22'
```

---

## 102. Firewall

```bash
firewall-cmd --list-services
```

---

## 103. SSH 설정 확인

```bash
sshd -T
```

---

## 104. 문법 검사

```bash
sshd -t
```

---

## 105. 사용자별 적용 설정

```bash
sshd -T -C user=sysadmin,host=Server-A,addr=192.168.111.150
```

---

## 106. Key 생성

```bash
ssh-keygen -t ed25519
```

---

## 107. Public Key 등록

```bash
ssh-copy-id -i PUBLIC_KEY USER@SERVER
```

---

## 108. 특정 Private Key Login

```bash
ssh -i PRIVATE_KEY USER@SERVER
```

---

## 109. sudo 확인

```bash
sudo whoami
```

---

## 110. SSH Log

```bash
journalctl -u sshd
```

또는:

```bash
grep sshd /var/log/secure
```

---

# 이번 실습 핵심 설정

## 111. Global Hardening

```conf
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
X11Forwarding no
AllowTcpForwarding no
AllowUsers sysadmin sftpuser
```

---

## 112. root

```conf
PermitRootLogin no
```

---

## 113. sysadmin

```conf
Match User sysadmin
    PasswordAuthentication no
```

---

## 114. sftpuser

```conf
Match User sftpuser
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

---

# 최종 정책

## 115. root

```text
SSH Direct Login
→ X
```

---

## 116. sysadmin

```text
SSH Login
→ O

Public Key
→ O

Password
→ X

sudo
→ O
```

---

## 117. sftpuser

```text
SSH Service
→ O

Password 인증
→ O

일반 Shell
→ X

SFTP
→ O

Chroot
→ O
```

---

# 최종 구조

## 118. 전체 인증 흐름

```text
                         Client-L
                    192.168.111.150
                            |
                            |
                     TCP 22 / SSH
                            |
                            v
                         Server-A
                    192.168.111.100
                            |
                           sshd
                            |
           ┌────────────────┴────────────────┐
           |                                 |
           v                                 v
       sysadmin                          sftpuser
           |                                 |
       Public Key                         Password
           |                                 |
       SSH Session                      internal-sftp
           |                                 |
          sudo                          ChrootDirectory
           |
           v
       root 권한


root
SSH Direct Login
        X
```

---

# Troubleshooting 순서

## 119. 추천 순서

SSH 문제가 발생하면 다음 순서로 확인한다.

```text
1. Client ↔ Server Network 확인
        ↓
2. sshd active 확인
        ↓
3. TCP 22 Listen 확인
        ↓
4. Firewall ssh 확인
        ↓
5. sshd -t
        ↓
6. sshd -T / sshd -T -C
        ↓
7. 사용자 계정 / AllowUsers 확인
        ↓
8. 인증 방식 확인
        ↓
9. Private Key / authorized_keys 확인
        ↓
10. File Permission / Owner 확인
        ↓
11. SELinux 확인
        ↓
12. /var/log/secure / journalctl 확인
        ↓
13. 필요하면 ssh -vvv 사용
```

---

# 핵심 정리

## 120. SSH Hardening

```text
SSH Hardening
=
원격 관리에 필요한 기능은 유지하면서
불필요하거나 위험한 접근 방법을 제한하는 작업
```

---

## 121. root 직접 Login

```text
PermitRootLogin no
```

를 통해 root의 원격 SSH 직접 Login을 제한하였다.

---

## 122. 관리자 접근

```text
Client
        ↓
SSH Public Key
        ↓
sysadmin
        ↓
sudo
        ↓
root 권한
```

구조를 사용하였다.

---

## 123. Key 인증

```text
Client Private Key
        +
Server authorized_keys의 Public Key
        ↓
SSH 인증
```

Private Key 자체는 Server로 공개하지 않는다.

---

## 124. 사용자별 인증 정책

```text
sysadmin
→ Public Key Only

sftpuser
→ Password 기반 SFTP
```

`Match User`를 이용하여 목적이 다른 계정에 서로 다른 정책을 적용하였다.

---

## 125. AllowUsers

```conf
AllowUsers sysadmin sftpuser
```

SSH Service 접근 가능 계정을 명시적으로 제한하였다.

---

## 126. 인증 실패 제한

```conf
MaxAuthTries 3
```

으로 한 Connection에서 허용되는 인증 시도 횟수를 줄였다.

실제:

```text
Too many authentication failures
```

까지 확인하였다.

---

## 127. 사용하지 않는 기능 제한

```conf
X11Forwarding no
AllowTcpForwarding no
```

이번 Server에서 사용하지 않는 SSH 기능을 제한하였다.

---

## 128. 설정 검증

```text
sshd -t
→ 문법 확인

sshd -T
→ 실제 적용값 확인

sshd -T -C
→ 특정 사용자 / Connection 기준 실제 적용값 확인
```

---

## 129. 실제 Login 검증

```text
sysadmin Key Login
→ 성공

sysadmin Password Login
→ 실패

root SSH Login
→ 실패

sftpuser Password SFTP
→ 성공
```

---

## 130. Log 검증

```text
Accepted publickey for sysadmin
→ Key 인증 성공

User root ... not allowed
→ root 접근 제한

maximum authentication attempts exceeded
→ MaxAuthTries 동작

Accepted password for sftpuser
→ 기존 SFTP 인증 유지
```

---

# 최종 이해

## 131. 이번 실습의 핵심

이번 SSH Hardening 실습의 핵심은 단순히:

```text
root Login을 끄는 것
```

이 아니다.

안전한 작업 순서는:

```text
대체 관리자 계정 준비
        ↓
sudo 검증
        ↓
SSH Key 준비
        ↓
Key Login 성공 확인
        ↓
보안 설정 적용
        ↓
sshd -t
        ↓
사용자별 sshd -T 검증
        ↓
Reload
        ↓
새 Session 접속 확인
        ↓
차단 정책 실제 테스트
        ↓
기존 SFTP 기능 재검증
        ↓
Security Log 확인
```

이다.

최종적으로 관리자 접근을:

```text
root + Password 중심
```

구조에서:

```text
sysadmin
        ↓
SSH Public Key
        ↓
sudo
        ↓
root 권한
```

구조로 변경하였다.

또한 기존 `sftpuser`는:

```text
Password Authentication
+
internal-sftp
+
ChrootDirectory
```

구성을 유지하였다.

따라서 이번 실습에서는 단순히 SSH 옵션을 변경한 것이 아니라:

```text
관리자 계정 분리
Public Key 인증
root 직접 Login 차단
사용자별 인증 정책
인증 횟수 제한
불필요한 기능 차단
기존 SFTP 기능 보존
설정 문법 검증
실제 접속 검증
Security Log 검증
```

까지 수행하여 SSH Server Hardening의 전체 흐름을 확인하였다.
