# Samba 이론 정리

## 1. Samba란?

Samba는 Linux / UNIX System에서 SMB Protocol을 사용할 수 있도록 해주는 Software이다.

대표적으로:

```text
Linux ↔ Windows
Linux ↔ Linux
```

환경에서 Network File Sharing에 사용할 수 있다.

예:

```text
Server-A
/samba/share
        |
        | Samba
        | SMB
        v
Client-L
/mnt/samba-share
```

Client는 Network를 통해 Server의 공유 Directory에 접근한다.

---

# Samba와 SMB

## 2. SMB란?

SMB는 Server Message Block의 약자이다.

Network를 통해:

```text
File Sharing
Directory Sharing
Printer Sharing
```

등을 제공하는 Protocol이다.

현재 환경에서는 Samba가 SMB Server 역할을 수행한다.

```text
Samba
→ Software

SMB
→ Network Protocol
```

---

## 3. CIFS란?

CIFS는 Common Internet File System의 약자로 SMB 계열의 과거 명칭 및 구현과 관련된 용어이다.

Linux에서 Samba 공유를 File System으로 Mount할 때는 현재도:

```bash
mount -t cifs
```

형태를 사용한다.

하지만:

```text
File System Type 이름
→ cifs

실제로 사용한 SMB Version
→ SMB 3.0
```

처럼 구분할 수 있다.

이번 실습:

```bash
mount -t cifs //192.168.111.100/share /mnt/samba-share -o username=smbuser,vers=3.0
```

에서:

```text
-t cifs
→ Linux CIFS Client File System 사용

vers=3.0
→ SMB Protocol Version 3.0 사용
```

이다.

---

# 전체 구조

## 4. 이번 실습 구성

```text
Server-A
192.168.111.100
        |
        v
/samba/share
        |
        v
Samba [share]
        |
        v
SMB
TCP 445
        |
        v
Client-L
192.168.111.150
        |
        v
/mnt/samba-share
```

---

# 실제 경로와 공유 이름

## 5. /samba/share

Server-A의 실제 Linux Directory:

```text
/samba/share
```

이다.

실제 파일 Data는 이 Directory에 저장된다.

---

## 6. [share]

Samba 설정:

```ini
[share]
        path = /samba/share
```

에서:

```text
[share]
```

는 Client에게 공개되는 공유 이름이다.

즉:

```text
Samba 공유 이름
share

실제 Linux Directory
/samba/share
```

이다.

---

## 7. Client는 실제 Server 경로를 모른다

Client에서는:

```text
/samba/share
```

를 직접 지정하지 않는다.

대신:

```text
//192.168.111.100/share
```

로 접근한다.

구조:

```text
Client
//192.168.111.100/share
        ↓
Samba가 [share] 검색
        ↓
path = /samba/share
        ↓
Server 실제 Directory 접근
```

---

# CIFS Mount Point

## 8. /mnt/samba-share

Client-L의:

```text
/mnt/samba-share
```

는 Samba 공유를 연결하기 위한 Mount Point이다.

Mount 전:

```text
/mnt/samba-share
→ Client-L의 일반 Local Directory
```

Mount 후:

```text
/mnt/samba-share
        ↓
SMB/CIFS
        ↓
Server-A
/samba/share
```

가 된다.

---

## 9. Mount 상태에서 파일 생성

Client-L:

```bash
touch /mnt/samba-share/test.txt
```

실제 저장:

```text
Server-A
/samba/share/test.txt
```

구조:

```text
Client-L
/mnt/samba-share/test.txt
        |
        | SMB
        v
Server-A
/samba/share/test.txt
```

따라서 Client에서는 Local Directory처럼 보여도 실제 Data는 Server-A에 저장된다.

---

## 10. Unmount 상태에서는?

CIFS가 Mount되지 않은 상태에서:

```bash
touch /mnt/samba-share/test.txt
```

를 하면 파일은 Client-L의 Local Disk에 생성된다.

즉:

```text
CIFS Mount 상태

/mnt/samba-share/file.txt
→ Server-A에 저장
```

```text
CIFS Unmount 상태

/mnt/samba-share/file.txt
→ Client-L Local Disk에 저장
```

따라서 작업 전에 Mount 여부를 확인하는 것이 중요하다.

```bash
mount | grep samba-share
```

또는:

```bash
df -hT /mnt/samba-share
```

---

# Samba 사용자

## 11. Samba 사용자 인증

이번 실습에서는:

```text
smbuser
```

를 Samba 인증 사용자로 사용하였다.

Linux User:

```bash
useradd -M -s /sbin/nologin smbuser
```

Samba Password:

```bash
smbpasswd -a smbuser
```

---

## 12. 왜 Linux 사용자를 먼저 만드는가?

Samba Local User 인증 환경에서는 Samba 계정이 Server의 Linux User와 연결된다.

따라서 먼저:

```text
Linux User
smbuser
```

가 존재해야 한다.

그 후:

```bash
smbpasswd -a smbuser
```

를 이용하여 Samba 인증 정보도 등록한다.

구조:

```text
Linux User 생성
        ↓
smbuser
        ↓
Samba Password 등록
        ↓
SMB 인증 가능
```

---

## 13. Linux Password와 Samba Password

둘은 반드시 같은 Password일 필요가 없다.

```text
Linux 계정 Password
→ Linux Login 인증

Samba Password
→ SMB/Samba 인증
```

이번 실습에서는:

```bash
smbpasswd -a smbuser
```

를 이용하여 Samba용 Password를 별도로 설정하였다.

---

## 14. /sbin/nologin

이번 계정:

```bash
useradd -M -s /sbin/nologin smbuser
```

에서:

```text
-s /sbin/nologin
```

을 사용하였다.

의미:

```text
일반 Interactive Shell Login 제한
```

이다.

즉 Samba File Sharing 목적으로만 사용하는 계정으로 구성할 수 있다.

---

## 15. -M

```text
-M
```

은 Home Directory를 자동 생성하지 않는 Option이다.

이번 실습에서는 별도의 Home Directory가 필요하지 않았으므로 사용하였다.

---

# pdbedit

## 16. Samba 사용자 확인

```bash
pdbedit -L
```

이번 결과:

```text
smbuser:1010:
```

Samba 사용자 Database에 `smbuser`가 등록되어 있음을 확인하였다.

---

# smb.conf

## 17. Samba 주요 설정 파일

```text
/etc/samba/smb.conf
```

Samba Server의 주요 Configuration File이다.

이번 설정:

```ini
[share]
        comment = Samba Share
        path = /samba/share
        browseable = yes
        read only = no
        valid users = smbuser
        create mask = 0660
        directory mask = 0770
```

---

# [share]

## 18. 공유 Section

```ini
[share]
```

는 Samba 공유 이름이다.

Client에서는:

```text
//192.168.111.100/share
```

로 접근한다.

---

# path

## 19. path

```ini
path = /samba/share
```

Samba 공유가 실제로 연결되는 Server의 Linux Directory이다.

```text
[share]
        ↓
/samba/share
```

---

# browseable

## 20. browseable = yes

```ini
browseable = yes
```

공유 목록을 조회했을 때 해당 Share가 표시될 수 있도록 한다.

예:

```bash
smbclient -L //192.168.111.100 -U smbuser
```

결과에:

```text
share
```

가 나타났다.

---

# read only

## 21. read only = no

```ini
read only = no
```

는 Samba Client의 쓰기를 허용한다.

즉:

```text
읽기 O
쓰기 O
```

이다.

반대로:

```ini
read only = yes
```

이면 일반적으로 읽기 전용 공유로 사용한다.

---

# valid users

## 22. valid users

```ini
valid users = smbuser
```

해당 공유에 접근할 수 있는 Samba 사용자를 제한한다.

이번 환경:

```text
smbuser
→ 접근 허용
```

이다.

---

# create mask

## 23. create mask = 0660

```ini
create mask = 0660
```

Samba를 통해 새로 생성되는 File Permission에 영향을 준다.

이번 실제 결과:

```text
-rw-rw----.
```

숫자로:

```text
0660
```

이다.

---

## 24. create mask는 기존 파일을 바꾸는가?

아니다.

기존에 Linux에서 직접 만든:

```text
server-test.txt
```

는:

```text
-rw-r--r--
```

이었다.

Samba를 통해 새로 Upload한:

```text
samba-upload.txt
client-l-upload.txt
cifs-client-test.txt
```

는:

```text
-rw-rw----
```

형태로 생성되었다.

즉:

```text
create mask
→ Samba를 통해 새로 생성되는 File에 영향

기존 File Permission
→ 자동 변경하지 않음
```

이다.

---

# directory mask

## 25. directory mask = 0770

```ini
directory mask = 0770
```

Samba를 통해 생성되는 Directory의 Permission에 영향을 준다.

```text
Owner
rwx

Group
rwx

Other
---
```

형태이다.

---

# Linux Permission

## 26. Samba Permission만 보면 되는가?

아니다.

Samba 공유에서는 여러 Layer를 함께 확인해야 한다.

```text
Samba 설정
        +
Linux File Permission
        +
SELinux
```

모두 접근을 허용해야 정상적으로 사용할 수 있다.

예를 들어:

```text
read only = no
```

로 Samba에서 쓰기를 허용했더라도 Server Linux Directory Permission이 쓰기를 막으면 실제 Write가 실패할 수 있다.

---

# /samba/share Permission

## 27. chmod 2770

이번 실습:

```bash
chmod 2770 /samba/share
```

결과:

```text
drwxrws---
```

이다.

---

## 28. 770

```text
Owner
rwx

Group
rwx

Other
---
```

이다.

---

## 29. 앞의 2

```text
2770
```

에서 첫 번째:

```text
2
```

는 setgid이다.

Directory에 setgid를 적용하면 내부에 생성되는 File / Directory가 해당 Directory의 Group을 상속하는 데 도움이 된다.

---

# SELinux

## 30. SELinux Enforcing

이번 Server-A:

```bash
getenforce
```

결과:

```text
Enforcing
```

였다.

즉 SELinux를 비활성화하지 않고 Samba를 구성하였다.

---

## 31. samba_share_t

Samba 공유 Directory에:

```text
samba_share_t
```

SELinux Type을 적용하였다.

```bash
semanage fcontext -a -t samba_share_t "/samba/share(/.*)?"
restorecon -Rv /samba/share
```

---

## 32. semanage fcontext

```bash
semanage fcontext -a -t samba_share_t "/samba/share(/.*)?"
```

의 의미:

```text
/samba/share
및
그 하위 경로
```

에 대해 Samba 공유용 SELinux Context 규칙을 등록한다.

---

## 33. restorecon

```bash
restorecon -Rv /samba/share
```

등록한 SELinux File Context 정책을 실제 File / Directory에 적용한다.

확인:

```bash
ls -Zd /samba/share
```

실제:

```text
unconfined_u:object_r:samba_share_t:s0
```

---

## 34. chmod 777로 해결하면 안 되는 이유

Samba 접근 문제가 발생했을 때 무조건:

```bash
chmod 777
```

을 하는 것은 좋은 해결 방법이 아니다.

문제의 원인이:

```text
Linux Permission
Samba 설정
SELinux
Firewall
사용자 인증
```

중 어디에 있는지 확인해야 한다.

이번 실습에서는:

```text
Linux Permission 설정
+
SELinux samba_share_t
+
Samba valid users
```

를 함께 구성하였다.

---

# testparm

## 35. testparm이란?

Samba Configuration을 검사하는 명령어이다.

```bash
testparm -s
```

이번 결과:

```text
Loaded services file OK.
```

가 확인되었다.

즉 `smb.conf` 문법을 정상적으로 읽었다는 의미이다.

---

## 36. 설정 변경 후 확인 순서

Samba 설정을 수정한 뒤에는:

```text
smb.conf 수정
        ↓
testparm -s
        ↓
설정 정상 여부 확인
        ↓
Service Reload / Restart
```

순서가 안전하다.

---

# Samba Service

## 37. smb.service

Rocky Linux에서 Samba SMB Service:

```text
smb.service
```

를 사용하였다.

시작:

```bash
systemctl enable --now smb
```

확인:

```bash
systemctl is-active smb
systemctl is-enabled smb
```

실제:

```text
active
enabled
```

---

# smbd

## 38. smbd

Port 확인 결과:

```text
users:(("smbd",...))
```

가 나타났다.

`smbd`는 SMB Client의 File / Printer Sharing 요청을 처리하는 Samba 핵심 Daemon이다.

---

# Samba Port

## 39. TCP 445

현대 SMB에서 핵심적으로 사용하는 Port:

```text
TCP 445
```

이다.

확인:

```bash
ss -lntp | grep ':445'
```

---

## 40. TCP 139

이번 환경에서는:

```text
TCP 139
```

도 Listen 중이었다.

139는 NetBIOS Session Service 기반의 과거 SMB 통신과 관련된다.

현대 SMB에서는 TCP 445가 핵심이다.

---

# Firewall

## 41. Samba Firewall Service

```bash
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

결과에:

```text
samba
```

가 포함되었다.

---

# smbclient

## 42. smbclient란?

Command Line 환경에서 SMB Server에 접속할 수 있는 Client Program이다.

FTP Client와 비슷한 형태로 사용할 수 있다.

---

## 43. 공유 목록 조회

```bash
smbclient -L //192.168.111.100 -U smbuser
```

`-L`:

```text
Server가 제공하는 Share 목록 조회
```

`-U smbuser`:

```text
smbuser로 인증
```

---

## 44. 공유 직접 접속

```bash
smbclient //192.168.111.100/share -U smbuser
```

정상 접속:

```text
smb: \>
```

형태의 Prompt가 나타난다.

---

## 45. ls

Samba Prompt:

```text
smb: \> ls
```

Server의 공유 파일 목록 확인.

---

## 46. get

```text
get server-test.txt /tmp/server-test-download.txt
```

Server → Client Download.

```text
Samba Server
        ↓
SMB
        ↓
Client Local File
```

---

## 47. put

```text
put /tmp/client-l-upload.txt client-l-upload.txt
```

Client → Server Upload.

```text
Client Local File
        ↓
SMB
        ↓
Server /samba/share
```

---

# SMB1 disabled

## 48. SMB1 disabled 메시지

이번 출력:

```text
SMB1 disabled -- no workgroup available
```

이 표시되었다.

이번 실습에서 이것은 Samba 공유 실패를 의미하지 않았다.

실제로:

```text
share 목록 조회 성공
SMB 인증 성공
Upload 성공
Download 성공
```

했기 때문에 Service는 정상적으로 동작하였다.

SMB1은 오래된 Protocol Version이므로 현대 환경에서는 일반적으로 사용을 피하는 것이 바람직하다.

---

# CIFS Mount

## 49. Samba 공유를 Mount하는 이유

`smbclient`를 이용하면:

```text
smb: \>
```

Prompt에서 File을 Upload / Download할 수 있다.

하지만 CIFS Mount를 사용하면:

```text
/mnt/samba-share
```

를 일반 Linux Directory처럼 사용할 수 있다.

예:

```bash
cd /mnt/samba-share
ls
cat server-test.txt
touch test.txt
```

---

## 50. CIFS Mount 명령

```bash
mount -t cifs //192.168.111.100/share /mnt/samba-share -o username=smbuser,vers=3.0
```

구조:

```text
mount
→ File System 연결

-t cifs
→ CIFS Client File System

//192.168.111.100/share
→ Samba 공유

/mnt/samba-share
→ Client Mount Point

username=smbuser
→ Samba 인증 사용자

vers=3.0
→ SMB 3.0 사용
```

---

# SMB Version

## 51. vers=3.0

이번 실습에서는:

```text
vers=3.0
```

을 지정하였다.

즉 Mount 시 SMB 3.0 Protocol을 사용하도록 설정하였다.

확인:

```bash
mount | grep samba-share
```

실제:

```text
vers=3.0
```

이 확인되었다.

---

# Credentials File

## 52. 왜 Credentials File을 사용하는가?

다음처럼 `/etc/fstab`에 Password를 직접 작성할 수도 있지만:

```text
username=smbuser,password=실제비밀번호
```

민감한 인증 정보가 `/etc/fstab`에 그대로 노출된다.

그래서 별도 Credentials File을 사용하였다.

---

## 53. /root/.smbcredentials

예:

```text
username=smbuser
password=Samba_Password
```

실제 Password가 들어 있으므로 GitHub 등에 업로드해서는 안 된다.

---

## 54. chmod 600

```bash
chmod 600 /root/.smbcredentials
```

의미:

```text
root
→ Read / Write

Group
→ 접근 불가

Other
→ 접근 불가
```

즉:

```text
-rw-------
```

이다.

---

# /etc/fstab

## 55. CIFS 자동 Mount 설정

```fstab
//192.168.111.100/share  /mnt/samba-share  cifs  credentials=/root/.smbcredentials,vers=3.0,_netdev  0  0
```

---

## 56. 각 항목

```text
//192.168.111.100/share
→ Samba Server 공유

/mnt/samba-share
→ Client Mount Point

cifs
→ File System Type

credentials=/root/.smbcredentials
→ 인증 정보 File

vers=3.0
→ SMB 3.0

_netdev
→ Network File System임을 표시

0
→ dump 제외

0
→ fsck 제외
```

---

# _netdev

## 57. _netdev의 의미

Samba Share는 Network가 없으면 접근할 수 없다.

따라서:

```text
_netdev
```

를 사용하여 해당 File System이 Network 연결을 필요로 함을 나타낸다.

```text
Network 준비
        ↓
Remote SMB Server 접근
        ↓
CIFS Mount
```

의존성을 가진다.

---

# mount -a

## 58. 재부팅 전 fstab 검사

`/etc/fstab` 수정 후 바로 재부팅하기보다:

```bash
umount /mnt/samba-share
mount -a
```

로 먼저 검증하는 것이 안전하다.

정상이라면:

```bash
mount | grep samba-share
```

에서 다시 CIFS가 Mount된다.

---

# systemctl daemon-reload

## 59. fstab 수정 후 출력된 메시지

이번 실습:

```text
your fstab has been modified, but systemd still uses
the old version
```

메시지가 나타났다.

해결:

```bash
systemctl daemon-reload
```

systemd가 변경된 설정 정보를 다시 읽도록 하였다.

---

# target is busy

## 60. Unmount 실패

이번 실습:

```bash
umount /mnt/samba-share
```

실행 시:

```text
target is busy
```

가 발생하였다.

---

## 61. 원인

당시 Shell Prompt:

```text
[root@Client-L samba-share]#
```

였다.

즉 현재 Shell의 Working Directory가:

```text
/mnt/samba-share
```

안에 있었다.

구조:

```text
Shell
        ↓
/mnt/samba-share 사용 중
        ↓
umount 시도
        ↓
target is busy
```

---

## 62. 해결

Mount Point 밖으로 이동:

```bash
cd /root
```

다시:

```bash
umount /mnt/samba-share
```

정상 해제되었다.

---

## 63. 다른 Process가 사용 중일 때

항상 현재 Shell이 원인인 것은 아니다.

다른 Process가 Mount Point를 사용하고 있을 수도 있다.

확인:

```bash
fuser -vm /mnt/samba-share
```

를 이용할 수 있다.

---

# Client Permission 표시

## 64. 왜 Client에서는 root:root로 보였는가?

Server-A 실제 파일:

```text
-rw-rw----. smbuser smbuser cifs-client-test.txt
```

Client-L:

```text
-rwxr-xr-x. root root cifs-client-test.txt
```

처럼 다르게 표시되었다.

---

## 65. Mount Option 확인

Client CIFS Mount 정보에:

```text
uid=0
gid=0
file_mode=0755
dir_mode=0755
```

가 포함되어 있었다.

따라서 Client-L에서는:

```text
uid=0
→ root

gid=0
→ root

file_mode=0755
→ File을 0755 형태로 표시

dir_mode=0755
→ Directory를 0755 형태로 표시
```

하고 있었다.

---

## 66. Server 실제 Permission이 바뀐 것은 아니다

중요하다.

Client에서:

```text
root root
0755
```

로 보여도 Server-A의 실제 Linux File은:

```text
smbuser smbuser
0660
```

등으로 유지될 수 있다.

즉:

```text
Server-side 실제 File Permission
        ≠
Client CIFS Mount에서 표시되는 Permission
```

일 수 있다.

---

# CIFS와 UNIX Permission

## 67. 권한 판단의 여러 단계

Samba 사용 시 권한은 단순히 Client의 `ls -l` 결과 하나만 보면 안 된다.

전체적으로:

```text
SMB 사용자 인증
        ↓
smb.conf 접근 정책
        ↓
Samba Server Process
        ↓
Server Linux Permission
        ↓
SELinux
        ↓
실제 File 접근
```

이 함께 작동한다.

---

# Samba와 NFS 비교

## 68. 공통점

둘 다 Network File Sharing 기술이다.

```text
원격 Server의 Storage
        ↓
Network
        ↓
Client에서 사용
```

할 수 있다.

---

## 69. NFS 구조

```text
Server-A
/nfs/share
        ↓
NFS
        ↓
Client-L
/mnt/nfs-share
```

Mount:

```bash
mount -t nfs 192.168.111.100:/nfs/share /mnt/nfs-share
```

---

## 70. Samba 구조

```text
Server-A
/samba/share
        ↓
Samba [share]
        ↓
SMB
        ↓
Client-L
/mnt/samba-share
```

Mount:

```bash
mount -t cifs //192.168.111.100/share /mnt/samba-share -o username=smbuser,vers=3.0
```

---

## 71. 가장 큰 차이

NFS:

```text
Server Export Path
192.168.111.100:/nfs/share
```

를 직접 지정한다.

Samba:

```text
//192.168.111.100/share
```

처럼 Samba가 정의한 Share Name을 사용한다.

---

## 72. 인증 방식 차이

이번 NFS 실습:

```text
sec=sys
→ UID/GID 기반
```

이번 Samba 실습:

```text
smbuser
+
Samba Password
```

를 이용한 SMB User 인증.

---

## 73. 주 사용 환경

일반적으로:

```text
NFS
→ Linux / UNIX 환경에서 많이 사용

Samba / SMB
→ Windows와 Linux 간 File Sharing에 많이 사용
```

한다.

Samba는 Linux Client에서도 사용할 수 있다.

---

# 파일 복사와 Network Mount 차이

## 74. Samba Mount는 File Copy가 아니다

CIFS Mount는 Server의 File을 Client Disk에 전부 복사하는 방식이 아니다.

```text
Client Application
        ↓
/mnt/samba-share
        ↓
SMB Request
        ↓
Network
        ↓
Server-A
/samba/share
```

형태로 원격 Data를 사용한다.

---

# Windows에서의 접근 형태

## 75. Windows UNC Path

Samba Share는 Windows에서는 일반적으로:

```text
\\192.168.111.100\share
```

형태로 접근한다.

Linux에서는:

```text
//192.168.111.100/share
```

형태를 사용한다.

개념적으로 같은 SMB Share를 가리킨다.

---

# 장애 처리

## 76. Samba 연결 문제 확인 순서

추천 확인 순서:

```text
1. Network
        ↓
2. smb Service
        ↓
3. TCP 445
        ↓
4. Firewall
        ↓
5. smb.conf
        ↓
6. Samba User
        ↓
7. Linux Permission
        ↓
8. SELinux
        ↓
9. Client Authentication
        ↓
10. CIFS Mount
```

---

## 77. Network 확인

Client-L:

```bash
ping -c 3 192.168.111.100
```

Server와 기본 Network 통신이 되는지 확인한다.

---

## 78. Service 확인

Server-A:

```bash
systemctl is-active smb
```

---

## 79. Port 확인

```bash
ss -lntp | grep ':445'
```

TCP 445가 Listen 상태인지 확인한다.

---

## 80. Firewall 확인

```bash
firewall-cmd --list-services
```

확인:

```text
samba
```

---

## 81. Configuration 확인

```bash
testparm -s
```

Samba 설정 오류가 있는지 확인한다.

---

## 82. Samba User 확인

```bash
pdbedit -L
```

해당 사용자가 Samba User Database에 등록되어 있는지 확인한다.

---

## 83. Linux User 확인

```bash
id smbuser
```

Linux User 존재 및 UID/GID를 확인한다.

---

## 84. Directory Permission

```bash
ls -ld /samba/share
```

Directory의 Owner / Group / Permission 확인.

---

## 85. SELinux 확인

```bash
getenforce
ls -Zd /samba/share
```

이번 구성에서는:

```text
Enforcing
samba_share_t
```

가 핵심이다.

---

## 86. 공유 목록 확인

Client:

```bash
smbclient -L //192.168.111.100 -U smbuser
```

이 단계가 실패한다면 CIFS Mount보다 먼저 Samba Service / 인증 문제를 확인하는 것이 좋다.

---

## 87. 공유 직접 접속

```bash
smbclient //192.168.111.100/share -U smbuser
```

Samba Share 자체에 정상 접근 가능한지 확인한다.

---

## 88. CIFS Mount 확인

```bash
mount | grep samba-share
```

---

## 89. File System 확인

```bash
df -hT /mnt/samba-share
```

정상이라면:

```text
Type
cifs
```

가 확인된다.

---

# 인증 실패

## 90. NT_STATUS_LOGON_FAILURE

Samba 인증 실패가 발생한다면 다음을 확인한다.

```text
Samba Username
Samba Password
pdbedit 등록 여부
valid users
```

Password를 다시 설정하려면:

```bash
smbpasswd smbuser
```

등을 이용할 수 있다.

---

# Permission denied

## 91. Write 실패

공유 접속은 되지만 파일 생성이 실패하면:

```text
read only 설정
valid users
Linux Directory Permission
Owner / Group
SELinux Context
```

를 확인한다.

이번 실습:

```ini
read only = no
valid users = smbuser
```

그리고:

```text
/samba/share
smbuser:smbuser
2770
samba_share_t
```

를 구성하였다.

---

# mount error

## 92. CIFS Mount 실패 시 확인

```text
cifs-utils 설치 여부
Server IP
Share Name
Username
Password
SMB Version
Firewall
TCP 445
```

를 확인한다.

Package:

```bash
rpm -q cifs-utils
```

---

# Mount와 Credentials

## 93. Credentials File 보안

다음 파일은:

```text
/root/.smbcredentials
```

Password를 포함한다.

따라서:

```text
GitHub
Public Repository
README
Screenshot
```

등에 실제 내용을 노출하지 않아야 한다.

Permission도:

```bash
chmod 600 /root/.smbcredentials
```

으로 제한한다.

---

# GitHub에 올리면 안 되는 정보

## 94. 민감 정보

Portfolio에는 설정 형식을 설명할 수 있지만 실제 Password는 넣지 않는다.

가능:

```text
username=smbuser
password=Samba_Password
```

불가능:

```text
password=실제사용비밀번호
```

Credentials File 자체도 Repository에 Commit하지 않는 것이 안전하다.

---

# 이번 실습에서 확인한 실제 흐름

## 95. smbclient Upload

```text
Client-L
/tmp/client-l-upload.txt
        ↓
smbclient
        ↓
SMB
        ↓
Server-A
/samba/share/client-l-upload.txt
```

---

## 96. CIFS Mount Write

```text
Client-L
/mnt/samba-share/cifs-client-test.txt
        ↓
CIFS Mount
        ↓
SMB
        ↓
Server-A
/samba/share/cifs-client-test.txt
```

---

## 97. Persistent Mount

```text
Client-L Boot
        ↓
/etc/fstab
        ↓
Credentials File
        ↓
Network 준비
        ↓
SMB Server 연결
        ↓
/mnt/samba-share
자동 Mount
```

---

# 주요 명령어

## 98. Samba Package

```bash
rpm -q samba
rpm -q samba-client
```

---

## 99. Samba 사용자

```bash
smbpasswd -a smbuser
pdbedit -L
```

---

## 100. 설정 검사

```bash
testparm -s
```

---

## 101. Service

```bash
systemctl is-active smb
systemctl is-enabled smb
```

---

## 102. Port

```bash
ss -lntp | grep -E ':445|:139'
```

---

## 103. SELinux

```bash
getenforce
ls -Zd /samba/share
```

---

## 104. 공유 목록

```bash
smbclient -L //192.168.111.100 -U smbuser
```

---

## 105. 공유 접속

```bash
smbclient //192.168.111.100/share -U smbuser
```

---

## 106. CIFS Mount

```bash
mount -t cifs //192.168.111.100/share /mnt/samba-share -o username=smbuser,vers=3.0
```

---

## 107. Mount 확인

```bash
mount | grep samba-share
df -hT /mnt/samba-share
```

---

## 108. Unmount

```bash
umount /mnt/samba-share
```

---

## 109. 사용 Process 확인

```bash
fuser -vm /mnt/samba-share
```

---

## 110. fstab 적용

```bash
systemctl daemon-reload
mount -a
```

---

# 핵심 정리

## 111. Samba

```text
Samba
=
Linux에서 SMB Protocol을 이용한
Network File Sharing 제공
```

---

## 112. SMB

```text
SMB
=
Network를 통한
File / Directory Sharing Protocol
```

---

## 113. CIFS Mount

```text
CIFS
=
Linux에서 SMB Share를
File System 형태로 Mount할 때 사용하는 Client File System
```

이번에는:

```text
vers=3.0
```

으로 SMB 3.0을 사용하였다.

---

## 114. 공유 이름과 실제 Directory

```text
Client가 접근하는 이름

//192.168.111.100/share
```

```text
Server 실제 Directory

/samba/share
```

연결 설정:

```ini
[share]
path = /samba/share
```

---

## 115. Samba 인증

```text
Linux User
        ↓
Samba User 등록
        ↓
Samba Password
        ↓
SMB Client 인증
```

---

## 116. Permission

```text
Samba 설정
+
Linux Permission
+
SELinux
```

모두 함께 확인해야 한다.

---

## 117. SELinux

```text
/samba/share
        ↓
samba_share_t
```

를 적용하여 SELinux Enforcing 상태에서도 공유를 구성하였다.

---

## 118. create mask

```text
create mask = 0660
```

Samba를 통해 새로 생성되는 File의 Permission에 영향을 준다.

실제:

```text
-rw-rw----
```

로 확인하였다.

---

## 119. directory mask

```text
directory mask = 0770
```

Samba를 통해 새로 생성되는 Directory Permission에 영향을 준다.

---

## 120. Credentials

```text
/root/.smbcredentials
```

에 인증 정보를 분리하고:

```text
chmod 600
```

으로 보호한다.

실제 Password는 GitHub에 올리지 않는다.

---

## 121. Persistent Mount

```fstab
//192.168.111.100/share  /mnt/samba-share  cifs  credentials=/root/.smbcredentials,vers=3.0,_netdev  0  0
```

을 이용하여 재부팅 후에도 자동 Mount되도록 구성하였다.

---

# NFS와 Samba 핵심 비교

## 122. NFS

```text
Server Path
192.168.111.100:/nfs/share
        ↓
NFS
        ↓
Client Mount Point
/mnt/nfs-share
```

주요 특징:

```text
Linux / UNIX 환경에서 많이 사용
UID/GID 중요
NFS Protocol
```

---

## 123. Samba

```text
Server 실제 Path
/samba/share
        ↓
Samba Share
[share]
        ↓
SMB
        ↓
Client Mount Point
/mnt/samba-share
```

주요 특징:

```text
Windows / Linux File Sharing
Samba User 인증
SMB Protocol
CIFS Mount
```

---

# 최종 구조

## 124. 전체 동작

```text
                           Server-A
                        192.168.111.100
                               |
                               v
                         /samba/share
                               |
                               | Linux Permission
                               | smbuser:smbuser
                               | 2770
                               |
                               | SELinux
                               | samba_share_t
                               |
                               v
                       /etc/samba/smb.conf
                               |
                               v
                           [share]
                               |
                     valid users = smbuser
                     read only = no
                     create mask = 0660
                     directory mask = 0770
                               |
                               v
                            smbd
                               |
                            TCP 445
                               |
                               v
                           SMB 3.0
                               |
                               v
                           Client-L
                        192.168.111.150
                               |
                               v
                   //192.168.111.100/share
                               |
                               v
                      /mnt/samba-share
                               |
                               v
                         CIFS Mount
                               |
                               v
                          /etc/fstab
                               |
                               v
                    /root/.smbcredentials
                               |
                               v
                       재부팅 후 자동 Mount
```

---

# 최종 이해

## 125. 핵심 흐름

이번 실습의 핵심은 다음과 같다.

```text
Server Linux Directory
/samba/share
        ↓
Samba에서 [share]로 공개
        ↓
smbuser 인증
        ↓
SMB Protocol
        ↓
Client-L 접근
        ↓
CIFS Mount
/mnt/samba-share
        ↓
Client에서 Local Directory처럼 사용
        ↓
실제 Data는 Server-A에 저장
```

Samba는 단순히 Directory를 Network에 노출하는 것만으로 끝나는 것이 아니라:

```text
Samba 인증
smb.conf
Linux Permission
SELinux
Firewall
SMB Port
Client Mount
```

가 함께 정상적으로 구성되어야 한다.

이번 실습에서는 SELinux를 `Enforcing` 상태로 유지하면서 `/samba/share`에 `samba_share_t`를 적용하고, `smbuser` 인증을 통해 File Upload / Download를 수행하였다.

또한 CIFS Mount를 이용하여 Samba Share를 Client-L의 `/mnt/samba-share`에 연결하고, Client에서 생성한 파일이 실제 Server-A의 `/samba/share`에 저장되는 것을 확인하였다.

마지막으로 Credentials File과 `/etc/fstab`을 이용하여 인증 정보를 분리하고 Client-L 재부팅 이후에도 Samba Share가 자동 Mount되는 것을 검증하였다.
