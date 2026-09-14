# Samba File Server 구축

Rocky Linux 환경에서 Samba를 이용하여 Linux Server의 Directory를 Network Share로 구성하고, Client-L에서 SMB/CIFS Protocol을 통해 접근하도록 구축하였다.

Server-A의 `/samba/share` Directory를 Samba의 `[share]` 이름으로 공개하고, Client-L에서 Samba 사용자 인증을 통해 파일을 Upload / Download하였다.

또한 Samba 공유를 Client-L의 `/mnt/samba-share`에 CIFS File System으로 Mount하고 `/etc/fstab`에 등록하여 재부팅 후에도 자동 Mount되는 것을 확인하였다.

```text
Server-A
192.168.111.100

/samba/share
      |
      | Samba [share]
      |
      | SMB/CIFS
      | TCP 445
      v

Client-L
192.168.111.150

/mnt/samba-share
```

---

# 1. 실습 환경

| 구분 | Server-A | Client-L |
|---|---|---|
| 역할 | Samba Server | Samba Client |
| IP | 192.168.111.100 | 192.168.111.150 |
| 공유 Directory | `/samba/share` | - |
| Samba 공유 이름 | `share` | - |
| Samba 사용자 | `smbuser` | `smbuser` 인증 |
| Mount Point | - | `/mnt/samba-share` |
| Protocol | SMB/CIFS | SMB/CIFS |
| 주요 Port | TCP 445 | - |

구조:

```text
Server-A
/samba/share

        ↓

Samba 공유 이름
[share]

        ↓

SMB/CIFS

        ↓

Client-L
/mnt/samba-share
```

---

# Samba Package

## 2. Samba 설치

Server-A:

```bash
dnf install -y samba samba-client
```

설치 확인:

```bash
rpm -q samba
rpm -q samba-client
```

실제 결과:

```text
samba-4.23.5-10.el9_8.x86_64
samba-client-4.23.5-10.el9_8.x86_64
```

---

# 기존 설정 백업

## 3. smb.conf 백업

```bash
cp -a /etc/samba/smb.conf /etc/samba/smb.conf.before-share
```

Samba 설정을 변경하기 전에 기존 설정 파일을 백업하였다.

---

# Samba 사용자

## 4. Linux 사용자 생성

Samba 인증에 사용할 사용자:

```bash
useradd -M -s /sbin/nologin smbuser
```

확인:

```bash
id smbuser
```

실제 결과:

```text
uid=1010(smbuser) gid=1010(smbuser) groups=1010(smbuser)
```

`-M`:

```text
Home Directory 생성하지 않음
```

`-s /sbin/nologin`:

```text
일반 Login Shell 사용 제한
```

---

## 5. Samba Password 등록

```bash
smbpasswd -a smbuser
```

Samba에서 사용할 Password를 별도로 등록하였다.

Samba 사용자 확인:

```bash
pdbedit -L
```

실제 결과:

```text
smbuser:1010:
```

Samba 인증 사용자는 Linux 사용자 계정이 먼저 존재해야 하며, `smbpasswd`를 통해 Samba Password를 별도로 등록하였다.

---

# 공유 Directory

## 6. Samba 공유 Directory 생성

```bash
mkdir -p /samba/share

chown smbuser:smbuser /samba/share

chmod 2770 /samba/share
```

확인:

```bash
ls -ld /samba/share
```

실제 결과:

```text
drwxrws---. 2 smbuser smbuser 29  9월 14 16:15 /samba/share
```

`2770`:

```text
Owner
→ rwx

Group
→ rwx

Other
→ 접근 불가

2
→ setgid
```

공유 Directory에 setgid를 적용하였다.

---

## 7. Server 테스트 파일 생성

```bash
echo "Samba test file from Server-A" > /samba/share/server-test.txt

chown smbuser:smbuser /samba/share/server-test.txt
```

실제 파일:

```text
-rw-r--r--. 1 smbuser smbuser 30  9월 14 16:15 server-test.txt
```

---

# SELinux

## 8. SELinux 상태 확인

```bash
getenforce
```

실제 결과:

```text
Enforcing
```

SELinux가 활성화된 상태에서 Samba 공유를 구성하였다.

---

## 9. Samba SELinux Context 적용

```bash
semanage fcontext -a -t samba_share_t "/samba/share(/.*)?"

restorecon -Rv /samba/share
```

확인:

```bash
ls -Zd /samba/share
ls -Z /samba/share/server-test.txt
```

실제 결과:

```text
unconfined_u:object_r:samba_share_t:s0 /samba/share
unconfined_u:object_r:samba_share_t:s0 /samba/share/server-test.txt
```

Samba가 공유 Directory에 접근할 수 있도록:

```text
samba_share_t
```

SELinux Type을 적용하였다.

---

# Samba 설정

## 10. smb.conf 공유 설정

```bash
vi /etc/samba/smb.conf
```

기존 설정 아래에 다음 내용을 추가하였다.

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

## 11. 주요 설정 의미

```text
[share]
→ Client에게 보이는 Samba 공유 이름

path = /samba/share
→ Server의 실제 공유 Directory

browseable = yes
→ 공유 목록에 표시

read only = no
→ 쓰기 허용

valid users = smbuser
→ smbuser만 접근 허용

create mask = 0660
→ Samba를 통해 생성되는 파일 권한 제한

directory mask = 0770
→ Samba를 통해 생성되는 Directory 권한 제한
```

---

# Samba 설정 검사

## 12. testparm

```bash
testparm -s
```

실제 결과:

```text
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed by GnuTLS (e.g. NTLM as a compatibility fallback)

Server role: ROLE_STANDALONE
```

공유 설정:

```text
[share]
        comment = Samba Share
        create mask = 0660
        directory mask = 0770
        path = /samba/share
        read only = No
        valid users = smbuser
```

`Loaded services file OK.`를 통해 Samba Configuration 문법이 정상임을 확인하였다.

---

# Samba Service

## 13. smb Service 시작

```bash
systemctl enable --now smb
```

확인:

```bash
systemctl is-active smb
systemctl is-enabled smb
```

실제 결과:

```text
active
enabled
```

현재 Service 실행 및 부팅 시 자동 시작을 설정하였다.

---

# Firewall

## 14. Samba Service 허용

```bash
firewall-cmd --permanent --add-service=samba

firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

실제 결과:

```text
cockpit dhcp dhcpv6-client dns http https mountd nfs rpc-bind samba ssh
```

Firewall에:

```text
samba
```

Service가 정상적으로 추가된 것을 확인하였다.

---

# Samba Port

## 15. TCP 445 / 139 확인

```bash
ss -lntp | grep -E ':445|:139'
```

실제 결과:

```text
LISTEN 0 50 0.0.0.0:445 0.0.0.0:* users:(("smbd",pid=8566,fd=29))
LISTEN 0 50 0.0.0.0:139 0.0.0.0:* users:(("smbd",pid=8566,fd=30))
LISTEN 0 50 [::]:445    [::]:*    users:(("smbd",pid=8566,fd=27))
LISTEN 0 50 [::]:139    [::]:*    users:(("smbd",pid=8566,fd=28))
```

주요 Port:

```text
TCP 445
→ SMB Direct Hosting

TCP 139
→ NetBIOS Session Service 기반 SMB
```

현대 SMB 통신에서는 TCP 445가 핵심 Port이다.

---

# Samba 공유 목록

## 16. Server-A에서 공유 목록 확인

```bash
smbclient -L //127.0.0.1 -U smbuser
```

실제 결과:

```text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
share           Disk      Samba Share
IPC$            IPC       IPC Service (Samba 4.23.5)
smbuser         Disk      Home Directories
```

우리가 설정한:

```text
share
```

공유가 정상적으로 표시되었다.

추가 출력:

```text
SMB1 disabled -- no workgroup available
```

SMB1이 비활성화되어 있음을 확인하였다.

---

# Samba 공유 접속

## 17. smbclient 접속

```bash
smbclient //127.0.0.1/share -U smbuser
```

접속 후:

```text
smb: \>
```

Prompt에서:

```text
ls
```

실제 공유 파일:

```text
server-test.txt
samba-upload.txt
```

등이 정상적으로 조회되었다.

---

# File Download

## 18. Samba를 통한 파일 Download

Samba Client에서:

```text
get server-test.txt /tmp/server-test-download.txt
```

실제 전송:

```text
getting file \server-test.txt of size 30 as /tmp/server-test-download.txt
```

Samba Protocol을 이용하여 Server의 파일을 Client 측으로 Download하였다.

---

# File Upload

## 19. Samba를 통한 Upload

테스트 파일:

```bash
echo "Samba upload test from Server-A" > /tmp/samba-upload.txt
```

Samba 공유에 접속:

```bash
smbclient //127.0.0.1/share -U smbuser
```

Upload:

```text
put /tmp/samba-upload.txt samba-upload.txt
```

Server 실제 Directory 확인:

```bash
ls -l /samba/share
cat /samba/share/samba-upload.txt
```

실제 결과:

```text
-rw-rw----. 1 smbuser smbuser 32  9월 14 16:32 samba-upload.txt
-rw-r--r--. 1 smbuser smbuser 30  9월 14 16:15 server-test.txt
```

내용:

```text
Samba upload test from Server-A
```

---

# create mask 검증

## 20. create mask = 0660

Samba를 통해 생성한 파일:

```text
-rw-rw----.
```

숫자 Permission:

```text
0660
```

따라서:

```ini
create mask = 0660
```

설정이 실제 Samba Upload 파일에 적용된 것을 확인하였다.

반면 Samba 설정 전에 Linux에서 직접 생성한:

```text
server-test.txt
```

은 기존 Permission:

```text
-rw-r--r--
```

을 유지하였다.

즉 `create mask`는 기존 파일의 Permission을 변경하는 것이 아니라 Samba를 통해 새로 생성되는 파일에 적용된다.

---

# Client-L Samba 접근

## 21. Client-L → Server-A Upload

Client-L에서 Samba 공유에 접속하여 파일을 Upload하였다.

Client-L에서 생성:

```bash
echo "Samba upload test from Client-L" > /tmp/client-l-upload.txt
```

Samba 공유:

```bash
smbclient //192.168.111.100/share -U smbuser
```

Upload:

```text
put /tmp/client-l-upload.txt client-l-upload.txt
```

Server-A에서 확인:

```bash
ls -l /samba/share
cat /samba/share/client-l-upload.txt
```

실제 결과:

```text
합계 12
-rw-rw----. 1 smbuser smbuser 32  9월 14 16:37 client-l-upload.txt
-rw-rw----. 1 smbuser smbuser 32  9월 14 16:32 samba-upload.txt
-rw-r--r--. 1 smbuser smbuser 30  9월 14 16:15 server-test.txt
```

내용:

```text
Samba upload test from Client-L
```

Client-L에서 SMB Protocol을 이용하여 Upload한 파일이 Server-A의 실제:

```text
/samba/share/client-l-upload.txt
```

에 저장되는 것을 확인하였다.

---

# CIFS Mount

## 22. Client Mount Point 생성

Client-L:

```bash
mkdir -p /mnt/samba-share
```

구조:

```text
Server-A

/samba/share
        |
        | Samba [share]
        |
        | SMB/CIFS
        v

Client-L

/mnt/samba-share
```

---

## 23. Samba 공유를 CIFS로 Mount

Client-L:

```bash
mount -t cifs //192.168.111.100/share /mnt/samba-share -o username=smbuser,vers=3.0
```

구조:

```text
//192.168.111.100/share
        ↓
SMB/CIFS
        ↓
/mnt/samba-share
```

여기서 Client는 Server의 실제 경로:

```text
/samba/share
```

를 직접 지정하지 않는다.

Samba 설정:

```ini
[share]
        path = /samba/share
```

에 의해 Client는:

```text
//192.168.111.100/share
```

라는 공유 이름을 통해 접근한다.

---

# CIFS Mount 파일 생성

## 24. Client-L에서 파일 생성

CIFS Mount 상태에서 Client-L:

```bash
echo "CIFS mount test from Client-L" > /mnt/samba-share/cifs-client-test.txt
```

Server-A에서:

```bash
ls -l /samba/share
cat /samba/share/cifs-client-test.txt
```

실제 결과:

```text
합계 16
-rw-rw----. 1 smbuser smbuser 30  9월 14 16:40 cifs-client-test.txt
-rw-rw----. 1 smbuser smbuser 32  9월 14 16:37 client-l-upload.txt
-rw-rw----. 1 smbuser smbuser 32  9월 14 16:32 samba-upload.txt
-rw-r--r--. 1 smbuser smbuser 30  9월 14 16:15 server-test.txt
```

내용:

```text
CIFS mount test from Client-L
```

즉 Client-L의:

```text
/mnt/samba-share/cifs-client-test.txt
```

에 생성한 파일이 실제로는 Server-A의:

```text
/samba/share/cifs-client-test.txt
```

에 저장되는 것을 확인하였다.

---

# Samba 저장 구조

## 25. 실제 파일 저장 위치

CIFS Mount 상태:

```text
Client-L

/mnt/samba-share/file.txt
        |
        | SMB/CIFS
        v
Server-A

/samba/share/file.txt
```

Client에서는 Local Directory처럼 사용하지만 실제 Data는 Samba Server의 Storage에 저장된다.

---

# Credentials File

## 26. Samba 인증 정보 분리

`/etc/fstab`에 Password를 직접 입력하지 않고 별도의 Credentials File을 사용하였다.

Client-L:

```bash
vi /root/.smbcredentials
```

형식:

```text
username=smbuser
password=Samba_Password
```

Permission:

```bash
chmod 600 /root/.smbcredentials
```

Credentials File에는 Password가 저장되므로 root만 접근할 수 있도록 Permission을 제한하였다.

> Credentials File의 실제 Password 내용은 GitHub에 업로드하지 않는다.

---

# /etc/fstab

## 27. 기존 fstab 백업

```bash
cp -a /etc/fstab /etc/fstab.before-samba
```

---

## 28. CIFS 자동 Mount 등록

Client-L:

```bash
vi /etc/fstab
```

추가:

```fstab
//192.168.111.100/share  /mnt/samba-share  cifs  credentials=/root/.smbcredentials,vers=3.0,_netdev  0  0
```

각 항목:

```text
//192.168.111.100/share
→ Server-A Samba 공유

/mnt/samba-share
→ Client-L Mount Point

cifs
→ SMB/CIFS File System

credentials=/root/.smbcredentials
→ Samba 인증 정보 File

vers=3.0
→ SMB 3.0 사용

_netdev
→ Network 연결이 필요한 File System

0 0
→ dump / fsck 대상 제외
```

---

# fstab 검증

## 29. CIFS Mount 해제

```bash
umount /mnt/samba-share
```

처음에는 현재 Shell이 Mount Point 내부에 있어:

```text
umount: /mnt/samba-share: target is busy.
```

오류가 발생하였다.

당시 현재 위치:

```text
[root@Client-L samba-share]#
```

Mount Point를 현재 Shell이 사용하고 있었기 때문에 발생한 문제였다.

Mount Point 밖으로 이동:

```bash
cd /root
```

다시 실행:

```bash
umount /mnt/samba-share
```

정상적으로 Unmount되었다.

---

## 30. target is busy

`target is busy`는 Mount Point를 현재 Process가 사용하고 있을 때 발생할 수 있다.

이번 실습에서는 현재 Shell의 Working Directory가:

```text
/mnt/samba-share
```

였기 때문에 발생하였다.

구조:

```text
Shell
        ↓
/mnt/samba-share 사용 중
        ↓
umount
        ↓
target is busy
```

해결:

```bash
cd /root
umount /mnt/samba-share
```

---

# mount -a

## 31. fstab 기반 재Mount

Unmount 후:

```bash
mount -a
```

실행 시:

```text
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
```

안내가 출력되었다.

Mount 자체는 정상적으로 수행되었으며 systemd가 변경된 `/etc/fstab` 정보를 다시 읽도록:

```bash
systemctl daemon-reload
```

를 실행하였다.

---

## 32. CIFS Mount 확인

```bash
mount | grep samba-share
```

실제 결과:

```text
//192.168.111.100/share on /mnt/samba-share type cifs (rw,relatime,vers=3.0,cache=strict,upcall_target=app,username=smbuser,uid=0,noforceuid,gid=0,noforcegid,addr=192.168.111.100,file_mode=0755,dir_mode=0755,soft,nounix,serverino,mapposix,reparse=nfs,nativesocket,symlink=native,rsize=4194304,wsize=4194304,bsize=1048576,retrans=1,echo_interval=60,actimeo=1,closetimeo=1)
```

주요 항목:

```text
type cifs
→ CIFS File System

vers=3.0
→ SMB 3.0

username=smbuser
→ Samba 인증 사용자

addr=192.168.111.100
→ Samba Server
```

---

## 33. File System 확인

```bash
df -hT /mnt/samba-share
```

실제 결과:

```text
Filesystem              Type  Size  Used Avail Use% Mounted on
//192.168.111.100/share cifs   96G  6.4G   90G   7% /mnt/samba-share
```

CIFS File System이 정상적으로 Mount된 것을 확인하였다.

---

# Client에서 보이는 Permission

## 34. CIFS File 표시

Client-L:

```bash
ls -l /mnt/samba-share
```

실제 결과:

```text
-rwxr-xr-x. 1 root root 30  9월 14 16:40 cifs-client-test.txt
-rwxr-xr-x. 1 root root 32  9월 14 16:37 client-l-upload.txt
-rwxr-xr-x. 1 root root 32  9월 14 16:32 samba-upload.txt
-rwxr-xr-x. 1 root root 30  9월 14 16:15 server-test.txt
```

현재 CIFS Mount Option:

```text
uid=0
gid=0
file_mode=0755
dir_mode=0755
```

에 의해 Client-L에서는 파일이:

```text
root:root
0755
```

형태로 표시된다.

그러나 Server-A의 실제 파일은:

```text
smbuser:smbuser
0660
```

등 Server의 Linux Permission을 기준으로 관리된다.

즉 Client에서 표시되는 CIFS Permission과 Server의 실제 File Permission은 구분해서 확인해야 한다.

---

# 재부팅 자동 Mount

## 35. Client-L 재부팅

```bash
reboot
```

재부팅 후 CIFS Mount 상태를 확인하였다.

```bash
mount | grep samba-share
```

실제 결과:

```text
//192.168.111.100/share on /mnt/samba-share type cifs (rw,relatime,vers=3.0,cache=strict,upcall_target=app,username=smbuser,uid=0,noforceuid,gid=0,noforcegid,addr=192.168.111.100,file_mode=0755,dir_mode=0755,soft,nounix,serverino,mapposix,reparse=nfs,nativesocket,symlink=native,rsize=4194304,wsize=4194304,bsize=1048576,retrans=1,echo_interval=60,actimeo=1,closetimeo=1)
```

재부팅 이후에도 Samba 공유가 자동으로 Mount된 것을 확인하였다.

---

## 36. 재부팅 후 File System 확인

```bash
df -hT /mnt/samba-share
```

실제 결과:

```text
Filesystem              Type  Size  Used Avail Use% Mounted on
//192.168.111.100/share cifs   96G  6.4G   90G   7% /mnt/samba-share
```

---

## 37. 재부팅 후 파일 확인

```bash
ls -l /mnt/samba-share

cat /mnt/samba-share/server-test.txt
cat /mnt/samba-share/cifs-client-test.txt
```

실제 결과:

```text
합계 16
-rwxr-xr-x. 1 root root 30  9월 14 16:40 cifs-client-test.txt
-rwxr-xr-x. 1 root root 32  9월 14 16:37 client-l-upload.txt
-rwxr-xr-x. 1 root root 32  9월 14 16:32 samba-upload.txt
-rwxr-xr-x. 1 root root 30  9월 14 16:15 server-test.txt
```

파일 내용:

```text
Samba test file from Server-A
CIFS mount test from Client-L
```

재부팅 후에도 기존 Samba 공유 파일에 정상적으로 접근할 수 있었다.

---

# NFS와 Samba 비교

## 38. NFS

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

Server의 실제 Export Path를 Client에서 지정한다.

---

## 39. Samba

```text
Server-A
/samba/share
        ↓
Samba [share]
        ↓
SMB/CIFS
        ↓
Client-L
/mnt/samba-share
```

Mount:

```bash
mount -t cifs //192.168.111.100/share /mnt/samba-share -o username=smbuser,vers=3.0
```

Client는 Server의 실제 `/samba/share` 경로 대신 Samba에서 설정한 공유 이름:

```text
share
```

를 이용하여 접근한다.

---

# 주요 설정 파일

## 40. Server-A

### /etc/samba/smb.conf

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

## 41. Client-L

### /etc/fstab

```fstab
//192.168.111.100/share  /mnt/samba-share  cifs  credentials=/root/.smbcredentials,vers=3.0,_netdev  0  0
```

### Credentials

```text
/root/.smbcredentials
```

Permission:

```text
600
```

Credentials File은 보안 정보이므로 Repository에 업로드하지 않는다.

---

# 주요 명령어

## 42. Samba 사용자 등록

```bash
smbpasswd -a smbuser
```

---

## 43. Samba 사용자 확인

```bash
pdbedit -L
```

---

## 44. Samba 설정 검사

```bash
testparm -s
```

---

## 45. Samba Service 확인

```bash
systemctl is-active smb
systemctl is-enabled smb
```

---

## 46. Samba Port 확인

```bash
ss -lntp | grep -E ':445|:139'
```

---

## 47. 공유 목록 조회

```bash
smbclient -L //192.168.111.100 -U smbuser
```

---

## 48. 공유 접속

```bash
smbclient //192.168.111.100/share -U smbuser
```

---

## 49. CIFS Mount

```bash
mount -t cifs //192.168.111.100/share /mnt/samba-share -o username=smbuser,vers=3.0
```

---

## 50. CIFS Mount 확인

```bash
mount | grep samba-share
```

---

## 51. File System 확인

```bash
df -hT /mnt/samba-share
```

---

## 52. Mount 해제

```bash
umount /mnt/samba-share
```

---

## 53. fstab 적용

```bash
systemctl daemon-reload
mount -a
```

---

# 전체 구조

## 54. 최종 구성

```text
                         Server-A
                      192.168.111.100
                             |
                             v
                       /samba/share
                             |
                             v
                    Samba Configuration
                             |
                          [share]
                             |
              valid users = smbuser
              create mask = 0660
              directory mask = 0770
                             |
                             v
                           smbd
                             |
                          TCP 445
                             |
                      SMB / CIFS
                             |
                             v
                         Client-L
                      192.168.111.150
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
                  재부팅 후 자동 Mount
```

---

# 실습 결과

```text
Samba Server
✓ Samba Package 설치
✓ smbuser 생성
✓ Samba Password 등록
✓ /samba/share 생성
✓ setgid 적용
✓ SELinux Enforcing 상태에서 구축
✓ samba_share_t 적용
✓ testparm Configuration 검사 성공
✓ smb Service active / enabled
✓ Firewall samba 허용
✓ TCP 445 / 139 Listen 확인

SMB
✓ Samba 공유 [share] 생성
✓ smbuser 인증
✓ 공유 목록 조회
✓ File Download
✓ File Upload
✓ create mask 0660 적용 확인
✓ Client-L → Server-A Upload 확인

CIFS
✓ /mnt/samba-share 생성
✓ CIFS Mount 성공
✓ SMB 3.0 사용
✓ Client-L에서 일반 Directory처럼 접근
✓ CIFS Mount에서 파일 생성
✓ Server-A /samba/share 실제 저장 확인

Persistent Mount
✓ Credentials File 분리
✓ /etc/fstab 등록
✓ _netdev 적용
✓ mount -a 검증
✓ systemctl daemon-reload 적용
✓ Client-L 재부팅
✓ 재부팅 후 CIFS 자동 Mount 확인
✓ 재부팅 후 공유 파일 정상 접근
```

---

# Troubleshooting 경험

이번 실습 중 CIFS Mount를 해제할 때:

```text
umount: /mnt/samba-share: target is busy.
```

오류가 발생하였다.

원인은 현재 Shell의 Working Directory가:

```text
/mnt/samba-share
```

내부에 있었기 때문이다.

해결:

```bash
cd /root
umount /mnt/samba-share
```

Mount Point를 사용 중인 Process가 없어지면서 정상적으로 Unmount할 수 있었다.

또한 `/etc/fstab` 수정 후:

```text
systemd still uses the old version
```

안내가 출력되어:

```bash
systemctl daemon-reload
```

를 실행하여 systemd가 변경된 설정을 다시 읽도록 하였다.

---

# 최종 결과

Server-A의:

```text
/samba/share
```

Directory를 Samba의:

```text
[share]
```

이름으로 Network에 공유하였다.

Client-L은:

```text
smbuser
```

계정으로 인증한 후 SMB Protocol을 이용하여 Server의 파일을 Download하고 Client 파일을 Upload하였다.

또한 Samba 공유를:

```text
//192.168.111.100/share
```

형태로 접근하여 Client-L의:

```text
/mnt/samba-share
```

에 CIFS File System으로 Mount하였다.

CIFS Mount 상태에서 Client-L에 생성한 파일이 실제 Server-A의:

```text
/samba/share
```

에 저장되는 것을 확인하였다.

마지막으로 Samba Password를 `/etc/fstab`에 직접 기록하지 않고 Credentials File로 분리하고:

```fstab
//192.168.111.100/share  /mnt/samba-share  cifs  credentials=/root/.smbcredentials,vers=3.0,_netdev  0  0
```

을 설정하여 Client-L 재부팅 이후에도 Samba 공유가 자동으로 Mount되는 것을 검증하였다.

```text
Samba Server 구축
        ↓
Samba 사용자 인증
        ↓
SMB Upload / Download
        ↓
CIFS Mount
        ↓
Client → Server 파일 저장
        ↓
Credentials 분리
        ↓
/etc/fstab
        ↓
재부팅
        ↓
자동 Mount 성공
```

---

## 관련 이론

Samba와 SMB/CIFS의 차이, Samba 사용자 인증, `smb.conf`, `create mask`, `directory mask`, SELinux `samba_share_t`, CIFS Mount, Credentials File 및 NFS와 Samba의 차이에 대한 자세한 내용은 `samba-notes.md`에서 정리한다.
