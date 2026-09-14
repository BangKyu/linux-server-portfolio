# NFS Server / Client 구축

Rocky Linux 환경에서 NFS(Network File System)를 이용하여 Server-A의 디렉터리를 Client-L에서 원격 Mount하고, 파일을 공유하도록 구성하였다.

Server-A의 `/nfs/share`를 NFS로 Export하고 Client-L의 `/mnt/nfs-share`에 Mount하여 원격 디렉터리를 Local Directory처럼 사용할 수 있도록 구성하였다.

또한 `/etc/fstab`에 NFS Mount 정보를 등록하여 Client-L 재부팅 후에도 자동으로 Mount되는 것을 확인하였다.

```text
Server-A
192.168.111.100
/nfs/share
        |
        | NFS
        | TCP 2049
        v
Client-L
192.168.111.150
/mnt/nfs-share
```

---

## 1. 실습 환경

| 구분 | Server-A | Client-L |
|---|---|---|
| 역할 | NFS Server | NFS Client |
| IP | 192.168.111.100 | 192.168.111.150 |
| OS | Rocky Linux | Rocky Linux |
| Package | nfs-utils | nfs-utils |
| 공유 Directory | `/nfs/share` | - |
| Mount Point | - | `/mnt/nfs-share` |
| NFS Version | - | NFSv4.2 |

NFS Server:

```text
Server-A
192.168.111.100
```

NFS Client:

```text
Client-L
192.168.111.150
```

---

# 사전 확인

## 2. NFS Package 확인

Server-A:

```bash
rpm -q nfs-utils
```

확인 결과:

```text
nfs-utils-2.5.4-42.el9.x86_64
```

Client-L:

```bash
rpm -q nfs-utils
```

확인 결과:

```text
nfs-utils-2.5.4-42.el9.x86_64
```

Server와 Client 모두 NFS 사용에 필요한 `nfs-utils` Package가 설치되어 있었다.

---

## 3. 기존 NFS Server 상태 확인

Server-A:

```bash
systemctl is-active nfs-server
systemctl is-enabled nfs-server
```

초기 상태:

```text
inactive
disabled
```

NFS Server Service는 아직 실행되지 않은 상태였다.

---

## 4. 기존 Network Port 확인

```bash
ss -lntup | grep -E ':2049|:111'
```

초기에는 `rpcbind`의 Port 111만 확인되었다.

```text
0.0.0.0:111
[::]:111
```

NFS Service Port인 TCP 2049는 아직 Listen 상태가 아니었다.

---

# NFS Server 구축

## 5. 공유 Directory 생성

Server-A:

```bash
mkdir -p /nfs/share
chmod 777 /nfs/share
```

확인:

```bash
ls -ld /nfs/share
```

실제 결과:

```text
drwxrwxrwx. 2 root root 29  9월 14 15:43 /nfs/share
```

이번 실습에서는 Client에서 파일 생성 및 쓰기 테스트를 수행하기 위해 공유 Directory에 쓰기 권한을 부여하였다.

---

## 6. Server 테스트 파일 생성

Server-A:

```bash
vi /nfs/share/server-test.txt
```

내용:

```text
NFS test file from Server-A
```

이 파일을 이후 Client-L에서 확인하여 NFS 공유가 정상적으로 동작하는지 검증하였다.

---

# NFS Export

## 7. /etc/exports 설정

Server-A:

```bash
vi /etc/exports
```

설정:

```conf
/nfs/share 192.168.111.0/24(rw,sync,root_squash)
```

설정 의미:

```text
/nfs/share
→ NFS로 공유할 Server Directory

192.168.111.0/24
→ NFS 접근을 허용할 Network

rw
→ 읽기 / 쓰기 허용

sync
→ Write 작업을 Storage에 반영한 후 응답

root_squash
→ Client의 root 권한을 Server의 root 권한으로 그대로 사용하지 못하도록 제한
```

---

## 8. Export 설정 적용

```bash
exportfs -rav
```

현재 Export 정보 확인:

```bash
exportfs -v
```

실제 결과:

```text
/nfs/share      192.168.111.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```

설정한:

```text
rw
sync
root_squash
```

옵션이 적용된 것을 확인하였다.

---

## 9. /etc/exports 확인

```bash
cat /etc/exports
```

실제 설정:

```text
/nfs/share 192.168.111.0/24(rw,sync,root_squash)
```

---

# NFS Service

## 10. NFS Server 시작

```bash
systemctl enable --now nfs-server
```

확인:

```bash
systemctl is-active nfs-server
systemctl is-enabled nfs-server
```

실제 결과:

```text
active
enabled
```

NFS Server가 현재 실행 중이며 부팅 시 자동 실행되도록 구성하였다.

---

## 11. NFS Port 확인

```bash
ss -lntup | grep -E ':2049|:111'
```

실제 결과에서 다음 Port를 확인하였다.

```text
TCP 111
TCP 2049
UDP 111
```

특히:

```text
0.0.0.0:2049
[::]:2049
```

가 Listen 상태인 것을 확인하였다.

```text
111
→ rpcbind

2049
→ NFS
```

---

# Firewall

## 12. NFS 관련 Service 허용

Server-A:

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

실제 결과:

```text
cockpit dhcp dhcpv6-client dns http https mountd nfs rpc-bind ssh
```

NFS 관련:

```text
mountd
nfs
rpc-bind
```

Service가 Firewall에서 허용된 것을 확인하였다.

---

# Export 확인

## 13. showmount 확인

Server-A:

```bash
showmount -e 192.168.111.100
```

실제 결과:

```text
Export list for 192.168.111.100:
/nfs/share 192.168.111.0/24
```

Server-A의 `/nfs/share`가 `192.168.111.0/24` Network에 정상적으로 Export된 것을 확인하였다.

---

# NFS Client

## 14. Client에서 Export 조회

Client-L:

```bash
showmount -e 192.168.111.100
```

실제 결과:

```text
Export list for 192.168.111.100:
/nfs/share 192.168.111.0/24
```

Client-L에서도 Server-A가 제공하는 NFS Export를 정상적으로 조회할 수 있었다.

---

## 15. Client Mount Point 생성

Client-L:

```bash
mkdir -p /mnt/nfs-share
```

`/mnt/nfs-share`는 Client-L에서 Server-A의 NFS 공유를 연결하여 사용할 Mount Point이다.

구조:

```text
Server-A
/nfs/share
        |
        | NFS
        v
Client-L
/mnt/nfs-share
```

---

## 16. NFS Mount

Client-L:

```bash
mount -t nfs 192.168.111.100:/nfs/share /mnt/nfs-share
```

명령 의미:

```text
mount
→ File System 연결

-t nfs
→ NFS File System 사용

192.168.111.100:/nfs/share
→ Server-A의 NFS 공유 Directory

/mnt/nfs-share
→ Client-L의 Mount Point
```

---

## 17. Mount 상태 확인

```bash
mount | grep nfs-share
```

실제 결과:

```text
192.168.111.100:/nfs/share on /mnt/nfs-share type nfs4 (rw,relatime,vers=4.2,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.111.150,local_lock=none,addr=192.168.111.100)
```

확인된 주요 항목:

```text
type nfs4
→ NFSv4 사용

vers=4.2
→ NFS Version 4.2

rw
→ 읽기 / 쓰기 가능

proto=tcp
→ TCP 사용

sec=sys
→ UNIX UID/GID 기반 인증 방식
```

---

## 18. File System 확인

```bash
df -hT /mnt/nfs-share
```

실제 결과:

```text
Filesystem                 Type  Size  Used Avail Use% Mounted on
192.168.111.100:/nfs/share nfs4   96G  6.4G   90G   7% /mnt/nfs-share
```

Client-L의 `/mnt/nfs-share`가 Server-A의 NFS File System에 정상적으로 연결된 것을 확인하였다.

---

# Server → Client 파일 공유

## 19. Server 파일 확인

Server-A에 생성한:

```text
/nfs/share/server-test.txt
```

를 Client-L에서 확인하였다.

```bash
ls -l /mnt/nfs-share
cat /mnt/nfs-share/server-test.txt
```

실제 결과:

```text
-rw-r--r--. 1 root root 28  9월 14 15:43 server-test.txt
```

파일 내용:

```text
NFS test file from Server-A
```

Server-A에서 생성한 파일을 Client-L에서 정상적으로 읽을 수 있었다.

---

# Client → Server 파일 공유

## 20. Client에서 파일 생성

Client-L:

```bash
echo "NFS test file from Client-L" > /mnt/nfs-share/client-test.txt
```

확인:

```bash
ls -l /mnt/nfs-share/client-test.txt
cat /mnt/nfs-share/client-test.txt
```

실제 결과:

```text
-rw-r--r--. 1 guest guest 28  9월 14 15:51 /mnt/nfs-share/client-test.txt
```

내용:

```text
NFS test file from Client-L
```

---

## 21. Server에서 Client 파일 확인

Server-A:

```bash
ls -l /nfs/share
cat /nfs/share/client-test.txt
```

실제 결과:

```text
합계 8
-rw-r--r--. 1 guest guest 28  9월 14 15:51 client-test.txt
-rw-r--r--. 1 root  root  28  9월 14 15:43 server-test.txt
```

파일 내용:

```text
NFS test file from Client-L
```

Client-L의 `/mnt/nfs-share`에서 생성한 파일이 실제 Server-A의 `/nfs/share`에 저장되는 것을 확인하였다.

구조:

```text
Client-L

/mnt/nfs-share/client-test.txt
        |
        | NFS Write
        v
Server-A

/nfs/share/client-test.txt
```

---

# NFS 저장 구조

## 22. NFS Mount Point의 의미

Client-L의:

```text
/mnt/nfs-share
```

는 실제 NFS Data를 별도로 복사해서 저장하는 Directory가 아니다.

NFS Mount 상태에서는:

```text
Client-L
/mnt/nfs-share
        ↓
Network
        ↓
Server-A
/nfs/share
```

형태로 Server-A의 원격 File System을 Client에서 Local Directory처럼 사용하는 구조이다.

따라서 NFS Mount 상태에서 Client-L이:

```text
/mnt/nfs-share/test.txt
```

를 생성하면 실제 파일은 Server-A의:

```text
/nfs/share/test.txt
```

에 저장된다.

---

# 영구 Mount

## 23. /etc/fstab 백업

Client-L:

```bash
cp -a /etc/fstab /etc/fstab.before-nfs
```

NFS 자동 Mount 설정 전 기존 `/etc/fstab`을 백업하였다.

---

## 24. /etc/fstab 설정

Client-L:

```bash
vi /etc/fstab
```

다음 설정을 추가하였다.

```fstab
192.168.111.100:/nfs/share  /mnt/nfs-share  nfs  defaults,_netdev  0  0
```

각 항목:

```text
192.168.111.100:/nfs/share
→ NFS Server와 공유 Directory

/mnt/nfs-share
→ Client Mount Point

nfs
→ File System Type

defaults
→ 기본 Mount Option

_netdev
→ Network 연결이 필요한 File System임을 표시

0
→ dump 대상 제외

0
→ fsck 검사 대상 제외
```

---

# fstab 검증

## 25. 기존 NFS Mount 해제

```bash
umount /mnt/nfs-share
```

확인:

```bash
mount | grep nfs-share
```

Mount 정보가 출력되지 않는 것을 확인한 후 `/etc/fstab` 설정을 검증하였다.

---

## 26. mount -a를 이용한 재Mount

```bash
mount -a
```

이후:

```bash
mount | grep nfs-share
```

실제 결과:

```text
192.168.111.100:/nfs/share on /mnt/nfs-share type nfs4 (rw,relatime,vers=4.2,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.111.150,local_lock=none,addr=192.168.111.100,_netdev)
```

`_netdev` Option까지 적용된 것을 확인하였다.

---

## 27. mount -a 이후 File System 확인

```bash
df -hT /mnt/nfs-share
```

실제 결과:

```text
Filesystem                 Type  Size  Used Avail Use% Mounted on
192.168.111.100:/nfs/share nfs4   96G  6.4G   90G   7% /mnt/nfs-share
```

---

## 28. 기존 파일 확인

```bash
ls -l /mnt/nfs-share

cat /mnt/nfs-share/server-test.txt
cat /mnt/nfs-share/client-test.txt
```

실제 결과:

```text
-rw-r--r--. 1 guest guest 28  9월 14 15:51 client-test.txt
-rw-r--r--. 1 root  root  28  9월 14 15:43 server-test.txt
```

내용:

```text
NFS test file from Server-A
NFS test file from Client-L
```

`/etc/fstab`을 이용한 NFS Mount 후에도 기존 파일이 정상적으로 확인되었다.

---

# 재부팅 자동 Mount 검증

## 29. Client-L 재부팅

Client-L:

```bash
reboot
```

재부팅 후 NFS Mount 상태를 다시 확인하였다.

---

## 30. 재부팅 후 Mount 확인

```bash
mount | grep nfs-share
```

실제 결과:

```text
192.168.111.100:/nfs/share on /mnt/nfs-share type nfs4 (rw,relatime,vers=4.2,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.111.150,local_lock=none,addr=192.168.111.100,_netdev)
```

Client-L 재부팅 이후에도 NFS가 자동으로 Mount된 것을 확인하였다.

---

## 31. 재부팅 후 File System 확인

```bash
df -hT /mnt/nfs-share
```

실제 결과:

```text
Filesystem                 Type  Size  Used Avail Use% Mounted on
192.168.111.100:/nfs/share nfs4   96G  6.4G   90G   7% /mnt/nfs-share
```

---

## 32. 재부팅 후 공유 파일 확인

```bash
ls -l /mnt/nfs-share

cat /mnt/nfs-share/server-test.txt
cat /mnt/nfs-share/client-test.txt
```

실제 결과:

```text
합계 8
-rw-r--r--. 1 guest guest 28  9월 14 15:51 client-test.txt
-rw-r--r--. 1 root  root  28  9월 14 15:43 server-test.txt
```

내용:

```text
NFS test file from Server-A
NFS test file from Client-L
```

재부팅 후에도 NFS Mount와 공유 파일이 정상적으로 유지되는 것을 확인하였다.

---

# 전체 동작 구조

## 33. 최종 구성

```text
                         Server-A
                      192.168.111.100
                              |
                              |
                       /nfs/share
                              |
                    /etc/exports
                              |
         192.168.111.0/24(rw,sync,root_squash)
                              |
                              |
                         NFS Server
                              |
                         TCP 2049
                              |
                 -------------------------
                              |
                              v
                         Client-L
                      192.168.111.150
                              |
                              |
                     /mnt/nfs-share
                              |
                         NFSv4.2
                              |
                         rw / TCP
                              |
                       /etc/fstab
                              |
                        자동 Mount
```

---

# 파일 공유 흐름

## 34. Server에서 파일 생성

```text
Server-A
/nfs/share/server-test.txt
        |
        | NFS
        v
Client-L
/mnt/nfs-share/server-test.txt
```

Client-L에서 정상적으로 읽기 확인.

---

## 35. Client에서 파일 생성

```text
Client-L
/mnt/nfs-share/client-test.txt
        |
        | NFS
        v
Server-A
/nfs/share/client-test.txt
```

Server-A에서 동일한 파일이 생성된 것을 확인.

---

# 주요 설정 파일

## 36. Server-A

### /etc/exports

```conf
/nfs/share 192.168.111.0/24(rw,sync,root_squash)
```

---

## 37. Client-L

### /etc/fstab

```fstab
192.168.111.100:/nfs/share  /mnt/nfs-share  nfs  defaults,_netdev  0  0
```

---

# 주요 명령어

## 38. NFS Export 확인

```bash
exportfs -v
```

---

## 39. Export 목록 조회

```bash
showmount -e 192.168.111.100
```

---

## 40. NFS Service 확인

```bash
systemctl is-active nfs-server
systemctl is-enabled nfs-server
```

---

## 41. NFS Port 확인

```bash
ss -lntup | grep -E ':2049|:111'
```

---

## 42. NFS 수동 Mount

```bash
mount -t nfs 192.168.111.100:/nfs/share /mnt/nfs-share
```

---

## 43. Mount 상태 확인

```bash
mount | grep nfs-share
```

---

## 44. NFS File System 확인

```bash
df -hT /mnt/nfs-share
```

---

## 45. Mount 해제

```bash
umount /mnt/nfs-share
```

---

## 46. /etc/fstab 전체 Mount 적용

```bash
mount -a
```

---

# 실습 결과

```text
NFS Server
✓ nfs-utils 설치 확인
✓ /nfs/share 생성
✓ /etc/exports 설정
✓ rw / sync / root_squash 적용
✓ nfs-server active / enabled
✓ TCP 2049 Listen 확인
✓ Firewall nfs / rpc-bind / mountd 허용
✓ showmount Export 확인

NFS Client
✓ Server Export 조회
✓ /mnt/nfs-share Mount Point 생성
✓ NFS Mount 성공
✓ NFSv4.2 확인
✓ TCP 사용 확인
✓ rw Mount 확인

File Sharing
✓ Server-A 파일 → Client-L 읽기 성공
✓ Client-L 파일 → Server-A 생성 확인
✓ 양방향 파일 공유 성공

Persistent Mount
✓ /etc/fstab 등록
✓ _netdev 적용
✓ umount 후 mount -a 재Mount 성공
✓ Client-L 재부팅
✓ 재부팅 후 NFS 자동 Mount 성공
✓ 재부팅 후 공유 파일 정상 확인
```

---

# 최종 결과

Server-A의 `/nfs/share` Directory를 NFS로 Export하고 Client-L의 `/mnt/nfs-share`에 Mount하여 Network File System 환경을 구축하였다.

```text
Server-A
/nfs/share
        |
        | NFS
        v
Client-L
/mnt/nfs-share
```

Server-A에서 생성한 파일을 Client-L에서 확인하고, Client-L에서 생성한 파일이 Server-A의 공유 Directory에 실제로 저장되는 것을 확인하여 양방향 파일 공유를 검증하였다.

또한 Client-L의 `/etc/fstab`에:

```fstab
192.168.111.100:/nfs/share  /mnt/nfs-share  nfs  defaults,_netdev  0  0
```

을 등록하고 `mount -a` 및 Client 재부팅을 통해 NFS File System이 자동으로 Mount되는 것을 검증하였다.

최종적으로:

```text
NFS Export
        ↓
Network Mount
        ↓
양방향 파일 공유
        ↓
fstab 영구 설정
        ↓
재부팅 후 자동 Mount
```

전체 NFS Server / Client 동작을 확인하였다.

---

## 관련 이론

NFS의 동작 원리, Export Option, `root_squash`, UID/GID, NFSv4, `sec=sys`, `_netdev` 및 Mount Option에 대한 자세한 내용은 `nfs-notes.md`에서 정리한다.
