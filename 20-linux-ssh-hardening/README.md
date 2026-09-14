# SSH Security Hardening

Rocky Linux Server-A의 SSH 환경을 점검하고, 일반 관리자 계정과 SSH Public Key 인증을 이용하여 원격 관리 보안을 강화하였다.

기존에 구성했던 `sftpuser`의 Password 기반 SFTP / Chroot 환경은 유지하면서 다음과 같이 관리자 SSH 접근 정책을 변경하였다.

```text
기존

root
→ SSH 직접 로그인 가능

sysadmin
→ Password 로그인 가능

sftpuser
→ Password 기반 SFTP
```

```text
보안 강화 후

root
→ SSH 직접 로그인 차단

sysadmin
→ SSH Public Key 인증
→ Password 로그인 차단
→ sudo를 통한 관리자 작업

sftpuser
→ 기존 Password 기반 SFTP 유지
→ Chroot 유지
```

최종 구조:

```text
                    Server-A
                 192.168.111.100
                       SSH :22
                          |
            ┌─────────────┴─────────────┐
            |                           |
         sysadmin                    sftpuser
            |                           |
       Public Key                   Password
            |                           |
        SSH Login                    SFTP Only
            |                           |
          sudo                     Chroot
            |
           root

root 직접 SSH Login
        X
```

---

# 1. 실습 환경

| 구분 | 내용 |
|---|---|
| Server | Server-A |
| Server IP | 192.168.111.100 |
| Client | Client-L |
| Client IP | 192.168.111.150 |
| SSH Server | OpenSSH |
| SSH Port | TCP 22 |
| 관리자 계정 | sysadmin |
| SFTP 전용 계정 | sftpuser |
| SELinux | Enforcing |

---

# 초기 SSH 상태 확인

## 2. OpenSSH Package

Server-A:

```bash
rpm -q openssh-server openssh-clients
```

실제 결과:

```text
openssh-server-9.9p1-9.el9_8.rocky.0.1.x86_64
openssh-clients-9.9p1-9.el9_8.rocky.0.1.x86_64
```

---

## 3. sshd Service

```bash
systemctl is-active sshd
systemctl is-enabled sshd
```

실제:

```text
active
enabled
```

---

## 4. SSH Port 확인

```bash
ss -lntp | grep ':22'
```

실제 결과:

```text
LISTEN 0 128 0.0.0.0:22 0.0.0.0:* users:(("sshd",pid=1180,fd=7))
LISTEN 0 128 [::]:22    [::]:*    users:(("sshd",pid=1180,fd=8))
```

TCP 22에서 `sshd`가 정상적으로 Listen 중이었다.

---

## 5. Firewall

```bash
firewall-cmd --list-services
```

실제 결과:

```text
cockpit dhcp dhcpv6-client dns http https mountd nfs ntp rpc-bind samba ssh
```

Firewall에서 SSH Service가 허용되어 있었다.

---

## 6. SELinux

```bash
getenforce
```

실제:

```text
Enforcing
```

SELinux를 비활성화하지 않고 SSH Hardening을 진행하였다.

---

# 초기 SSH 설정

## 7. 주요 sshd 설정 확인

```bash
sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|permitemptypasswords|maxauthtries|x11forwarding|allowtcpforwarding|usepam) '
```

실제:

```text
port 22
usepam yes
maxauthtries 6
permitrootlogin yes
pubkeyauthentication yes
passwordauthentication yes
x11forwarding yes
permitemptypasswords no
allowtcpforwarding yes
```

초기 상태:

```text
PermitRootLogin         yes
PasswordAuthentication yes
PubkeyAuthentication   yes
MaxAuthTries            6
X11Forwarding           yes
AllowTcpForwarding      yes
```

즉 root 직접 SSH 로그인과 Password 인증이 허용되어 있었다.

---

# sshd_config.d

## 8. 추가 설정 파일 확인

```bash
ls -l /etc/ssh/sshd_config.d/
```

실제:

```text
-rw-------. 1 root root 141 ... 01-permitrootlogin.conf
-rw-------. 1 root root 719 ... 50-redhat.conf
```

기존:

```text
/etc/ssh/sshd_config.d/01-permitrootlogin.conf
```

에는:

```conf
PermitRootLogin yes
```

가 설정되어 있었다.

---

# 기존 SFTP 환경

## 9. sftpuser 설정 확인

기존 `/etc/ssh/sshd_config`에는 다음 설정이 존재하였다.

```conf
Match User sftpuser
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

따라서 SSH Hardening 과정에서 기존 SFTP 전용 계정의 기능이 깨지지 않도록 유지하였다.

---

# 관리자 계정 생성

## 10. sysadmin 계정

SSH 원격 관리에 사용할 일반 관리자 계정:

```text
sysadmin
```

을 생성하였다.

```bash
useradd -m -s /bin/bash sysadmin
passwd sysadmin
```

`wheel` Group에 추가하였다.

```bash
usermod -aG wheel sysadmin
```

---

## 11. 관리자 Group 확인

```bash
id sysadmin
```

실제:

```text
uid=1011(sysadmin) gid=1011(sysadmin) groups=1011(sysadmin),10(wheel)
```

`sysadmin`이 `wheel` Group에 포함된 것을 확인하였다.

---

## 12. wheel Group

```bash
getent group wheel
```

실제:

```text
wheel:x:10:guest,sysadmin
```

---

## 13. sudo 권한 확인

```bash
grep -E '^[[:space:]]*%wheel' /etc/sudoers
```

실제:

```text
%wheel  ALL=(ALL)       ALL
```

`wheel` Group 사용자가 `sudo`를 사용할 수 있도록 설정되어 있었다.

---

# 일반 계정 SSH Login 검증

## 14. Client-L에서 접속

```bash
ssh sysadmin@192.168.111.100
```

접속 후:

```bash
whoami
hostname
id
sudo whoami
```

실제 결과:

```text
sysadmin
Server-A
uid=1011(sysadmin) gid=1011(sysadmin) groups=1011(sysadmin),10(wheel)
root
```

따라서:

```text
sysadmin SSH Login
→ 정상

wheel Group
→ 정상

sudo 권한
→ 정상
```

을 확인하였다.

root SSH Login을 차단하기 전에 대체 관리자 계정이 정상 동작하는 것을 먼저 검증하였다.

---

# SSH Public Key 인증

## 15. Client-L에서 SSH Key 생성

Client-L에서 이번 실습 전용 ED25519 Key를 생성하였다.

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519_server_a
```

생성되는 파일:

```text
/root/.ssh/id_ed25519_server_a
→ Private Key

/root/.ssh/id_ed25519_server_a.pub
→ Public Key
```

Private Key는 외부에 공개하거나 GitHub에 업로드하면 안 된다.

---

# Public Key 등록

## 16. ssh-copy-id

Client-L:

```bash
ssh-copy-id -i /root/.ssh/id_ed25519_server_a.pub sysadmin@192.168.111.100
```

Public Key를 Server-A의:

```text
/home/sysadmin/.ssh/authorized_keys
```

에 등록하였다.

---

## 17. authorized_keys Permission

Server-A에서 확인:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

실제:

```text
drwx------. 2 sysadmin sysadmin 29 ... /home/sysadmin/.ssh
-rw-------. 1 sysadmin sysadmin 95 ... /home/sysadmin/.ssh/authorized_keys
```

Permission:

```text
~/.ssh
→ 700

authorized_keys
→ 600
```

Owner:

```text
sysadmin:sysadmin
```

으로 정상 설정되어 있었다.

---

# Public Key Login 검증

## 18. Password 인증 없이 접속

Client-L:

```bash
ssh -i /root/.ssh/id_ed25519_server_a \
-o PreferredAuthentications=publickey \
-o PasswordAuthentication=no \
sysadmin@192.168.111.100
```

Password 인증을 사용하지 않고 Public Key 인증만 허용하여 접속하였다.

접속 후:

```bash
whoami
hostname
```

정상적으로:

```text
sysadmin
Server-A
```

가 확인되었다.

---

# SSH Hardening 설정

## 19. 기존 설정 백업

Server-A:

```bash
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.before-hardening
cp -a /etc/ssh/sshd_config.d/01-permitrootlogin.conf /etc/ssh/sshd_config.d/01-permitrootlogin.conf.before-hardening
```

설정 변경 전에 기존 Configuration을 백업하였다.

---

# Root SSH Login 차단

## 20. PermitRootLogin

파일:

```text
/etc/ssh/sshd_config.d/01-permitrootlogin.conf
```

기존:

```conf
PermitRootLogin yes
```

변경:

```conf
PermitRootLogin no
```

의미:

```text
root 계정의 SSH 직접 Login 차단
```

이다.

관리자는:

```text
sysadmin SSH Login
        ↓
sudo
        ↓
root 권한 사용
```

구조를 사용한다.

---

# 추가 Hardening 설정

## 21. 02-hardening.conf 생성

```bash
vi /etc/ssh/sshd_config.d/02-hardening.conf
```

설정:

```conf
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
X11Forwarding no
AllowTcpForwarding no
AllowUsers sysadmin sftpuser

Match User sysadmin
    PasswordAuthentication no

Match all
```

---

# PubkeyAuthentication

## 22. Public Key 인증 허용

```conf
PubkeyAuthentication yes
```

SSH Public Key 인증을 허용한다.

`sysadmin`의 관리자 SSH 접속에 사용하였다.

---

# PermitEmptyPasswords

## 23. 빈 Password 차단

```conf
PermitEmptyPasswords no
```

Password가 설정되지 않은 계정의 SSH 인증을 허용하지 않는다.

---

# MaxAuthTries

## 24. 인증 시도 횟수 제한

기존:

```text
MaxAuthTries 6
```

변경:

```conf
MaxAuthTries 3
```

한 SSH Connection에서 허용되는 인증 시도 횟수를 줄였다.

---

# X11Forwarding

## 25. X11 Forwarding 차단

기존:

```text
X11Forwarding yes
```

변경:

```conf
X11Forwarding no
```

이번 Server 환경에서는 X11 Forwarding 기능을 사용하지 않으므로 비활성화하였다.

---

# AllowTcpForwarding

## 26. TCP Forwarding 차단

기존:

```text
AllowTcpForwarding yes
```

변경:

```conf
AllowTcpForwarding no
```

이번 실습 환경에서는 SSH Tunnel / Port Forwarding을 사용하지 않으므로 비활성화하였다.

---

# AllowUsers

## 27. SSH 접근 계정 제한

```conf
AllowUsers sysadmin sftpuser
```

SSH Service를 사용할 수 있는 계정을:

```text
sysadmin
sftpuser
```

로 제한하였다.

따라서 root는 허용 목록에도 포함되지 않는다.

---

# sysadmin Password 인증 차단

## 28. Match User sysadmin

```conf
Match User sysadmin
    PasswordAuthentication no
```

`sysadmin`에게 Password 인증을 허용하지 않도록 구성하였다.

최종:

```text
sysadmin
→ Public Key 인증 O
→ Password 인증 X
```

---

# Match all

## 29. Match all

```conf
Match all
```

을 사용하여 `sysadmin`에 대한 Match Block을 종료하였다.

이후 다른 설정이 특정 사용자에게 의도치 않게 계속 적용되는 것을 방지할 수 있다.

---

# 기존 sftpuser 유지

## 30. SFTP 계정 정책

기존 설정:

```conf
Match User sftpuser
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

을 유지하였다.

최종적으로:

```text
sftpuser
→ Password 인증 O
→ SFTP O
→ 일반 Shell Login X
→ Chroot O
```

환경을 보존하였다.

---

# 설정 문법 검사

## 31. sshd -t

설정을 실제 Service에 적용하기 전에:

```bash
sshd -t
```

를 실행하였다.

실제 결과:

```text
출력 없음
```

`sshd -t`에서 출력이 없으므로 SSH Configuration 문법에 오류가 없음을 확인하였다.

---

# 사용자별 실제 적용 설정 검증

## 32. sysadmin

```bash
sshd -T -C user=sysadmin,host=Server-A,addr=192.168.111.150 | \
grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication|maxauthtries|x11forwarding|allowtcpforwarding|allowusers) '
```

실제:

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

의도한 Hardening 설정이 정상 적용되는 것을 확인하였다.

---

## 33. sysadmin 최종 정책

```text
Public Key Authentication
→ yes

Password Authentication
→ no

Max Authentication Tries
→ 3

X11 Forwarding
→ no

TCP Forwarding
→ no
```

---

# sftpuser 적용 설정 검증

## 34. sftpuser

```bash
sshd -T -C user=sftpuser,host=Server-A,addr=192.168.111.150 | \
grep -E '^(passwordauthentication|chrootdirectory|forcecommand|x11forwarding|allowtcpforwarding) '
```

실제:

```text
passwordauthentication yes
x11forwarding no
allowtcpforwarding no
forcecommand internal-sftp
chrootdirectory /sftp/%u
```

기존 SFTP 설정이 정상적으로 유지되는 것을 확인하였다.

---

# SSH Configuration 적용

## 35. sshd Reload

설정 검증 후:

```bash
systemctl reload sshd
```

Service 상태 확인:

```bash
systemctl is-active sshd
```

실제:

```text
active
```

기존 SSH Connection을 유지한 상태에서 설정을 Reload하였다.

---

# sysadmin Public Key Login 최종 검증

## 36. Key 전용 접속

Client-L:

```bash
ssh -i /root/.ssh/id_ed25519_server_a \
-o IdentitiesOnly=yes \
-o PreferredAuthentications=publickey \
-o PasswordAuthentication=no \
sysadmin@192.168.111.100
```

접속 후:

```bash
whoami
hostname
sudo whoami
```

실제:

```text
sysadmin
Server-A
root
```

따라서 Hardening 적용 이후에도:

```text
SSH Public Key Login
→ 정상

sudo 관리자 권한
→ 정상
```

임을 확인하였다.

---

# sysadmin Password Login 차단 검증

## 37. Password 인증만 사용

Client-L:

```bash
ssh \
-o PubkeyAuthentication=no \
-o PreferredAuthentications=password \
sysadmin@192.168.111.100
```

실제:

```text
sysadmin@192.168.111.100: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

인증 방법에 `password`가 제공되지 않았다.

따라서:

```text
sysadmin Password SSH Login
→ 차단
```

을 확인하였다.

---

# Root SSH Login 차단

## 38. root 접속 시도

Client-L:

```bash
ssh \
-o PubkeyAuthentication=no \
-o PreferredAuthentications=password \
root@192.168.111.100
```

실제:

```text
root@192.168.111.100's password:
Permission denied, please try again.
root@192.168.111.100's password:
Permission denied, please try again.
root@192.168.111.100's password:
Received disconnect from 192.168.111.100 port 22:2: Too many authentication failures
Disconnected from 192.168.111.100 port 22
```

root의 SSH Login이 실패하였다.

---

# MaxAuthTries 검증

## 39. 인증 횟수 제한

root 인증 실패 후:

```text
Too many authentication failures
```

가 발생하였다.

설정:

```conf
MaxAuthTries 3
```

이 실제로 동작하는 것을 확인하였다.

---

# 기존 SFTP 검증

## 40. sftpuser 접속

Client-L:

```bash
sftp sftpuser@192.168.111.100
```

실제:

```text
sftpuser@192.168.111.100's password:
Connected to 192.168.111.100.
sftp>
```

목록 확인:

```text
sftp> ls
upload
```

현재 위치:

```text
sftp> pwd
Remote working directory: /
```

기존 SFTP Password 인증 및 Chroot 환경이 정상적으로 유지되었다.

---

# 인증 Log 검증

## 41. sysadmin Public Key Login

`/var/log/secure`에서:

```text
Accepted publickey for sysadmin from 192.168.111.150
```

가 기록되었다.

실제:

```text
Accepted publickey for sysadmin from 192.168.111.150 port 59770 ssh2: ED25519 ...
```

즉 Client-L에서 `sysadmin` 계정으로 Public Key 인증에 성공한 기록이다.

---

# Root 차단 Log

## 42. AllowUsers 정책

실제 Log:

```text
User root from 192.168.111.150 not allowed because not listed in AllowUsers
```

root가:

```conf
AllowUsers sysadmin sftpuser
```

목록에 포함되지 않아 접근이 차단된 것을 확인하였다.

또한:

```conf
PermitRootLogin no
```

정책도 적용되어 있으므로 root 직접 SSH Login을 허용하지 않는 구성이다.

---

# 인증 실패 Log

## 43. Failed password

실제:

```text
Failed password for invalid user root from 192.168.111.150
```

root의 인증 시도 실패가 Security Log에 기록되었다.

---

# MaxAuthTries Log

## 44. 최대 인증 횟수 초과

실제:

```text
maximum authentication attempts exceeded for invalid user root from 192.168.111.150
```

그리고:

```text
Too many authentication failures
```

가 기록되었다.

`MaxAuthTries 3` 정책이 실제 인증 과정에서 동작하였다.

---

# SFTP 성공 Log

## 45. sftpuser

실제:

```text
Accepted password for sftpuser from 192.168.111.150
```

기존 SFTP 계정의 Password 인증이 계속 정상적으로 동작함을 확인하였다.

---

# Hardening 전후 비교

## 46. 변경 전

```text
PermitRootLogin
→ yes

PasswordAuthentication
→ yes

PubkeyAuthentication
→ yes

MaxAuthTries
→ 6

X11Forwarding
→ yes

AllowTcpForwarding
→ yes
```

---

## 47. 변경 후

```text
PermitRootLogin
→ no

sysadmin PasswordAuthentication
→ no

PubkeyAuthentication
→ yes

MaxAuthTries
→ 3

X11Forwarding
→ no

AllowTcpForwarding
→ no

AllowUsers
→ sysadmin
→ sftpuser
```

---

# 최종 계정 정책

## 48. root

```text
SSH 직접 Login
→ X
```

root 계정 자체를 삭제하거나 비활성화한 것은 아니다.

관리자는:

```text
sysadmin
        ↓
sudo
        ↓
root 권한
```

구조로 작업한다.

---

## 49. sysadmin

```text
SSH Login
→ O

Public Key Authentication
→ O

Password Authentication
→ X

sudo
→ O
```

관리자 원격 접속용 계정이다.

---

## 50. sftpuser

```text
Password Authentication
→ O

SFTP
→ O

Chroot
→ O

SSH 일반 Shell
→ X

TCP Forwarding
→ X

X11 Forwarding
→ X
```

기존 File Transfer 전용 계정 역할을 유지하였다.

---

# 보안 강화 포인트

## 51. root 직접 Login 제한

기존:

```text
Client
        ↓
root SSH Login
        ↓
Server
```

변경:

```text
Client
        ↓
sysadmin
        ↓
SSH Public Key
        ↓
sudo
        ↓
root
```

관리자 작업과 원격 인증 계정을 분리하였다.

---

## 52. Password 대신 Public Key

Password 방식:

```text
사용자가 Password 입력
        ↓
Server Password 인증
```

Public Key 방식:

```text
Client Private Key
        ↓
암호학적 인증
        ↓
Server authorized_keys의 Public Key와 검증
```

관리자 `sysadmin`은 SSH Key 인증만 사용하도록 구성하였다.

---

## 53. Private Key 보호

Client-L의:

```text
/root/.ssh/id_ed25519_server_a
```

는 Private Key이다.

다음에 노출하면 안 된다.

```text
GitHub
README
Screenshot
공유 Folder
Messenger
Public Repository
```

Public Key인:

```text
id_ed25519_server_a.pub
```

과 Private Key를 구분해야 한다.

---

# SSH 설정 변경 시 안전한 작업 순서

## 54. Lockout 방지

SSH 설정 변경은 잘못하면 원격 Server에 접속할 수 없게 될 수 있다.

따라서 다음 순서로 진행하였다.

```text
기존 SSH Session 유지
        ↓
일반 관리자 계정 생성
        ↓
sudo 동작 확인
        ↓
Public Key 등록
        ↓
Key Login 확인
        ↓
SSH 설정 수정
        ↓
sshd -t
        ↓
sshd -T
        ↓
Service Reload
        ↓
새 SSH Connection으로 검증
        ↓
기존 Session 종료
```

---

## 55. sshd -t의 중요성

설정 파일을 수정한 뒤 바로:

```bash
systemctl restart sshd
```

하는 것보다 먼저:

```bash
sshd -t
```

를 실행하여 문법을 검사하는 것이 안전하다.

```text
sshd -t
→ Configuration Syntax 검사
```

오류가 있다면 Service에 적용하기 전에 수정한다.

---

## 56. sshd -T의 중요성

```bash
sshd -T
```

는 실제 적용되는 SSH 설정을 확인할 수 있다.

특히 `Match User`가 존재할 경우:

```bash
sshd -T -C user=사용자,host=호스트,addr=IP
```

형태로 특정 사용자에게 적용되는 최종 설정을 확인하는 것이 중요하다.

---

# 주요 명령어

## 57. SSH Service

```bash
systemctl is-active sshd
systemctl is-enabled sshd
```

---

## 58. SSH Port

```bash
ss -lntp | grep ':22'
```

---

## 59. 설정 문법 검사

```bash
sshd -t
```

---

## 60. 전체 적용 설정

```bash
sshd -T
```

---

## 61. 사용자별 적용 설정

```bash
sshd -T -C user=sysadmin,host=Server-A,addr=192.168.111.150
```

---

## 62. SSH Key 생성

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519_server_a
```

---

## 63. Public Key 등록

```bash
ssh-copy-id -i /root/.ssh/id_ed25519_server_a.pub sysadmin@192.168.111.100
```

---

## 64. Key Login

```bash
ssh -i /root/.ssh/id_ed25519_server_a \
-o IdentitiesOnly=yes \
sysadmin@192.168.111.100
```

---

## 65. SSH Log

```bash
journalctl -u sshd
```

또는:

```bash
tail /var/log/secure
```

필요한 인증 기록만 확인:

```bash
grep -E 'Accepted publickey|not allowed|maximum authentication|Accepted password for sftpuser' /var/log/secure | tail -n 10
```

---

# Troubleshooting

## 66. Key Login 실패

다음 항목을 확인한다.

```text
Public Key 등록 여부
authorized_keys 경로
.ssh Permission
authorized_keys Permission
Owner / Group
PubkeyAuthentication
Client Private Key 경로
```

확인:

```bash
ls -ld /home/sysadmin/.ssh
ls -l /home/sysadmin/.ssh/authorized_keys
```

정상 구성:

```text
.ssh
→ 700

authorized_keys
→ 600

Owner
→ sysadmin
```

---

## 67. SSH 설정 변경 후 접속 실패

기존 Session을 닫지 않은 상태에서 확인한다.

```bash
sshd -t
```

그리고:

```bash
sshd -T -C user=sysadmin,host=Server-A,addr=192.168.111.150
```

실제 적용 값을 확인한다.

---

## 68. Permission denied

다음 항목을 확인한다.

```text
AllowUsers
PermitRootLogin
PasswordAuthentication
PubkeyAuthentication
authorized_keys
Key File
Account 존재 여부
```

---

## 69. SFTP가 갑자기 동작하지 않을 때

```bash
sshd -T -C user=sftpuser,host=Server-A,addr=192.168.111.150
```

에서:

```text
passwordauthentication
forcecommand
chrootdirectory
```

를 확인한다.

이번 정상 값:

```text
passwordauthentication yes
forcecommand internal-sftp
chrootdirectory /sftp/%u
```

---

## 70. Log 확인 순서

SSH 인증 문제가 발생하면:

```bash
journalctl -u sshd
```

또는:

```bash
grep sshd /var/log/secure | tail
```

을 이용하여 실제 Server 측 인증 기록을 확인한다.

---

# 실습 결과

```text
OpenSSH
✓ openssh-server 설치 확인
✓ openssh-clients 설치 확인
✓ sshd active
✓ sshd enabled
✓ TCP 22 Listen
✓ Firewall ssh 허용
✓ SELinux Enforcing

관리자 계정
✓ sysadmin 생성
✓ wheel Group 추가
✓ sudo 사용 확인

SSH Public Key
✓ ED25519 Key 생성
✓ Public Key 등록
✓ authorized_keys 생성
✓ .ssh 700 확인
✓ authorized_keys 600 확인
✓ Public Key Login 성공

SSH Hardening
✓ PermitRootLogin no
✓ sysadmin PasswordAuthentication no
✓ PubkeyAuthentication yes
✓ MaxAuthTries 3
✓ X11Forwarding no
✓ AllowTcpForwarding no
✓ AllowUsers sysadmin sftpuser

Configuration 검증
✓ sshd -t 성공
✓ sysadmin 적용값 확인
✓ sftpuser 적용값 확인
✓ sshd Reload 후 active

실제 인증 검증
✓ sysadmin Public Key Login 성공
✓ sysadmin Password Login 실패
✓ root SSH Login 실패
✓ MaxAuthTries 3 동작
✓ sftpuser Password SFTP 성공
✓ 기존 Chroot 유지

Log 검증
✓ Accepted publickey for sysadmin
✓ root not allowed 확인
✓ authentication failure 확인
✓ maximum authentication attempts exceeded 확인
✓ Accepted password for sftpuser 확인
```

---

# 최종 구조

```text
                         Client-L
                    192.168.111.150
                           |
                           |
                  SSH Public Key
                           |
                           v
                        Server-A
                    192.168.111.100
                           |
                        TCP 22
                           |
                          sshd
                           |
            ┌──────────────┴──────────────┐
            |                             |
            v                             v
         sysadmin                      sftpuser
            |                             |
    Public Key Only                 Password Auth
            |                             |
       SSH Shell                       SFTP
            |                             |
          sudo                       Chroot /
            |
            v
           root


root 직접 SSH Login
        X

sysadmin Password Login
        X

sysadmin Key Login
        O

sftpuser SFTP
        O
```

---

# 최종 결과

Server-A의 기존 SSH 환경을 점검한 후 일반 관리자 계정 `sysadmin`을 생성하고 `wheel` Group을 통해 sudo 권한을 부여하였다.

root의 SSH 직접 Login을 차단하기 전에 `sysadmin` 계정의 SSH 및 sudo 동작을 먼저 검증하였다.

Client-L에서 ED25519 SSH Key를 생성하고 Public Key를 Server-A의:

```text
/home/sysadmin/.ssh/authorized_keys
```

에 등록하였다.

Public Key 인증이 정상적으로 동작하는 것을 확인한 후 SSH Hardening을 적용하였다.

주요 설정:

```conf
PermitRootLogin no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
X11Forwarding no
AllowTcpForwarding no
AllowUsers sysadmin sftpuser
```

그리고 `sysadmin`에 대해서는:

```conf
Match User sysadmin
    PasswordAuthentication no
```

를 적용하여 관리자 계정은 SSH Public Key 인증만 사용하도록 구성하였다.

적용 전:

```bash
sshd -t
```

를 통해 Configuration 문법을 확인하고:

```bash
sshd -T -C ...
```

를 이용하여 사용자별 실제 적용값까지 검증하였다.

Hardening 이후 Client-L에서:

```text
sysadmin Public Key Login
→ 성공

sysadmin Password Login
→ 실패

root SSH Login
→ 실패

sftpuser Password SFTP
→ 성공
```

을 확인하였다.

또한 `/var/log/secure`에서:

```text
Accepted publickey for sysadmin
User root ... not allowed
maximum authentication attempts exceeded
Accepted password for sftpuser
```

등의 실제 인증 Log를 확인하였다.

최종적으로 관리자 SSH 접근은:

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

구조로 변경하였으며, 기존 SFTP 전용 계정의 Password 인증 및 Chroot 환경은 유지하였다.

---

## 관련 이론

SSH Public / Private Key, `authorized_keys`, `PermitRootLogin`, `PasswordAuthentication`, `AllowUsers`, `MaxAuthTries`, `Match User`, X11 / TCP Forwarding, SSH 인증 과정 및 SSH Troubleshooting에 대한 자세한 내용은 `ssh-hardening-notes.md`에서 정리한다.
