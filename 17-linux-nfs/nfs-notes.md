# NFS 이론 정리

## 1. NFS란?

NFS는 Network File System의 약자이다.

Network를 통해 다른 Linux Server의 Directory를 자신의 Local Directory처럼 사용할 수 있게 해주는 File Sharing 방식이다.

예:

```text
Server-A
/nfs/share
        |
        | Network
        | NFS
        v
Client-L
/mnt/nfs-share
```

Client-L 입장에서는:

```text
/mnt/nfs-share
```

라는 Local Directory를 사용하는 것처럼 보이지만, 실제 Data는 Server-A의:

```text
/nfs/share
```

에 저장된다.

---

# NFS 구조

## 2. NFS Server

NFS Server는 특정 Directory를 Network에 공유하는 역할을 한다.

이번 실습:

```text
Server-A
192.168.111.100
```

공유 Directory:

```text
/nfs/share
```

Server에서는 `/etc/exports`를 이용하여 어떤 Directory를 어떤 Client에게 공유할지 설정한다.

---

## 3. NFS Client

NFS Client는 NFS Server가 공유한 Directory를 자신의 File System에 Mount해서 사용하는 Host이다.

이번 실습:

```text
Client-L
192.168.111.150
```

Mount Point:

```text
/mnt/nfs-share
```

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

# NFS Mount

## 4. NFS Mount란?

NFS Mount는 원격 Server의 File System을 Client의 특정 Directory에 연결하는 것이다.

이번 실습:

```bash
mount -t nfs 192.168.111.100:/nfs/share /mnt/nfs-share
```

각 항목:

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

## 5. Mount Point란?

Mount Point는 File System이 연결되는 Local Directory이다.

이번 실습:

```text
/mnt/nfs-share
```

Mount 전에는 그냥 일반 Local Directory이다.

```text
Client-L
/mnt/nfs-share
→ Local Directory
```

NFS Mount 후:

```text
Client-L
/mnt/nfs-share
        ↓
Server-A
/nfs/share
```

로 연결된다.

---

## 6. NFS Mount 상태에서 파일 생성

Client-L에서:

```bash
touch /mnt/nfs-share/test.txt
```

를 실행하면 실제 파일은 Server-A의:

```text
/nfs/share/test.txt
```

에 생성된다.

즉:

```text
Client-L
/mnt/nfs-share/test.txt
        |
        | NFS Write
        v
Server-A
/nfs/share/test.txt
```

이다.

---

## 7. Unmount 상태에서 파일 생성

중요한 차이점이다.

NFS가 Mount되지 않은 상태에서 Client-L의:

```text
/mnt/nfs-share
```

안에 파일을 생성하면 그 파일은 Client-L의 Local Disk에 저장된다.

즉:

```text
NFS Mount 상태
→ Server-A에 저장

NFS Unmount 상태
→ Client-L Local Disk에 저장
```

따라서 NFS Directory를 사용하기 전에는 실제 Mount 상태를 확인하는 것이 중요하다.

확인:

```bash
mount | grep nfs-share
```

또는:

```bash
df -hT /mnt/nfs-share
```

---

# /etc/exports

## 8. /etc/exports란?

NFS Server에서 어떤 Directory를 어떤 Client에게 공유할지 정의하는 설정 파일이다.

이번 실습:

```conf
/nfs/share 192.168.111.0/24(rw,sync,root_squash)
```

구조:

```text
공유 Directory
        +
접근 허용 대상
        +
Export Option
```

---

## 9. /nfs/share

```text
/nfs/share
```

는 Server-A에서 NFS로 공유할 Directory이다.

Client가 접근하면 이 Directory 내부의 파일을 읽거나 설정에 따라 쓸 수 있다.

---

## 10. 192.168.111.0/24

```text
192.168.111.0/24
```

는 NFS 접근을 허용할 Network 범위이다.

즉 같은 Network의:

```text
192.168.111.x
```

Host들이 접근 가능하도록 구성한 것이다.

---

# Export Option

## 11. rw

```text
rw
```

는 Read / Write를 의미한다.

Client에서:

```text
읽기 가능
쓰기 가능
```

하도록 허용한다.

이번 실습에서 Client-L이:

```text
client-test.txt
```

를 생성할 수 있었던 이유 중 하나이다.

---

## 12. ro

반대 옵션:

```text
ro
```

는 Read Only이다.

Client는 파일 읽기는 가능하지만 쓰기는 제한된다.

예:

```conf
/nfs/share 192.168.111.0/24(ro)
```

---

## 13. sync

```text
sync
```

는 Write 요청을 실제 Storage에 반영한 뒤 Client에게 작업 완료를 응답하는 방식이다.

쉽게 말하면:

```text
Client Write 요청
        ↓
Server Storage 반영
        ↓
완료 응답
```

구조이다.

Data 안정성을 중요하게 볼 때 사용하는 옵션이다.

---

## 14. async

반대 개념:

```text
async
```

는 Storage에 완전히 반영되기 전에 Client에게 응답할 수 있다.

성능상 이점이 있을 수 있지만 장애 발생 시 Data 손실 가능성을 고려해야 한다.

---

# root_squash

## 15. root_squash란?

NFS에서 중요한 보안 옵션이다.

Client의 `root` 사용자가 NFS Server에서도 그대로 Server의 `root` 권한을 가지게 되면 보안상 위험하다.

그래서:

```text
root_squash
```

를 사용하면 Client의 root 권한을 Server에서 제한한다.

---

## 16. 왜 root_squash가 필요한가?

예를 들어 Client-L의 root가:

```bash
touch /mnt/nfs-share/test.txt
```

를 실행했다고 하자.

Client root:

```text
UID 0
```

이다.

만약 Client root가 Server에서도 그대로 UID 0 권한을 사용할 수 있다면:

```text
Client root
        ↓
NFS
        ↓
Server root 권한
```

이 되어 보안 위험이 커진다.

`root_squash`는 이를 제한한다.

---

## 17. root_squash 동작 개념

```text
Client root
UID 0
        ↓
NFS
        ↓
Server에서 root 권한으로 그대로 처리 X
        ↓
Anonymous User 형태로 변환
```

기본적으로 NFS Server에서는 root 사용자를 제한된 anonymous 권한으로 처리한다.

---

## 18. no_root_squash

반대 옵션:

```text
no_root_squash
```

는 Client root를 Server에서도 root 권한으로 그대로 인정한다.

보안 위험이 있기 때문에 특별한 이유가 없다면 일반적인 공유 환경에서는 신중하게 사용해야 한다.

---

# UID / GID

## 19. NFS와 사용자 권한

NFS에서는 사용자 이름 자체보다 Linux의:

```text
UID
GID
```

가 중요하다.

예:

```text
guest
UID 1005
GID 1005
```

같은 방식으로 실제 권한이 처리된다.

---

## 20. 같은 사용자 이름만 있으면 되는가?

아니다.

중요한 것은 이름보다 숫자 UID/GID이다.

예를 들어:

```text
Server-A

guest
UID 1005
```

Client-L:

```text
guest
UID 1005
```

이면 같은 사용자로 자연스럽게 보일 수 있다.

하지만:

```text
Server-A
guest → UID 1005

Client-L
guest → UID 2000
```

처럼 UID가 다르면 같은 이름이라도 권한 문제가 발생할 수 있다.

---

## 21. UID/GID 확인

```bash
id guest
```

또는:

```bash
getent passwd guest
```

로 확인할 수 있다.

NFS에서 권한 문제가 생기면 Server와 Client의 UID/GID가 일치하는지 확인해야 한다.

---

# sec=sys

## 22. sec=sys란?

이번 Mount 정보:

```text
sec=sys
```

가 확인되었다.

`sec=sys`는 NFS에서 전통적으로 사용하는 UNIX 인증 방식이다.

쉽게 말하면:

```text
Client의 UID/GID 정보
        ↓
NFS Request
        ↓
Server가 해당 UID/GID를 기준으로 권한 판단
```

하는 방식이다.

---

## 23. sec=sys의 특징

```text
사용자 이름
보다
UID/GID 숫자
중요
```

하다.

따라서 여러 Linux Server에서 NFS를 운영할 때 사용자 ID 관리가 중요하다.

---

# NFS Version

## 24. NFSv4

이번 실습의 Mount 결과:

```text
type nfs4
```

였고:

```text
vers=4.2
```

가 확인되었다.

즉 NFS Version 4.2를 사용하였다.

---

## 25. NFSv4.2

NFSv4 계열은 이전 NFS Version에 비해 여러 기능이 통합되고 개선된 형태이다.

이번 실습에서는 Client와 Server가 자동 협상하여:

```text
NFSv4.2
```

를 사용하였다.

확인:

```bash
mount | grep nfs-share
```

실제:

```text
vers=4.2
```

---

# NFS Port

## 26. TCP 2049

NFS의 핵심 Port:

```text
TCP 2049
```

이다.

Server-A에서:

```bash
ss -lntup | grep 2049
```

로 확인하였다.

---

## 27. rpcbind Port 111

이번 환경에서:

```text
TCP/UDP 111
```

도 확인되었다.

Port 111은 `rpcbind`에서 사용한다.

RPC 기반 Service들이 사용하는 Program 번호와 Port 정보를 연결하는 역할을 한다.

---

## 28. mountd

Firewall에서도:

```text
mountd
```

를 허용하였다.

`mountd`는 NFS Export와 Mount 관련 RPC 처리에 사용된다.

---

# Firewall

## 29. NFS Firewall Service

이번 실습:

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
```

적용 후:

```text
nfs
rpc-bind
mountd
```

가 허용되었다.

---

## 30. Firewall과 NFS 문제

NFS Server Service가 정상이어도 Firewall에서 필요한 Service가 차단되면 Client가 접근하지 못할 수 있다.

확인:

```bash
firewall-cmd --list-services
```

---

# exportfs

## 31. exportfs란?

`exportfs`는 NFS Export 설정을 관리하거나 확인하는 명령이다.

설정 적용:

```bash
exportfs -rav
```

현재 Export 확인:

```bash
exportfs -v
```

---

## 32. exportfs -rav

```text
-r
→ /etc/exports 다시 읽기

-a
→ 모든 Export 대상

-v
→ 상세 출력
```

설정 변경 후 NFS 공유 설정을 다시 적용할 때 사용할 수 있다.

---

# showmount

## 33. showmount란?

NFS Server에서 어떤 Directory를 Export하고 있는지 확인할 수 있다.

이번 실습:

```bash
showmount -e 192.168.111.100
```

결과:

```text
Export list for 192.168.111.100:
/nfs/share 192.168.111.0/24
```

---

## 34. showmount -e

```text
-e
→ Export List 확인
```

Server와 Client 양쪽에서 사용할 수 있다.

Client에서 실행하면 Server의 공유 정보를 Network를 통해 조회할 수 있는지 확인하는 데 유용하다.

---

# /etc/fstab

## 35. 왜 /etc/fstab이 필요한가?

일반 `mount` 명령은 현재 실행 중인 System에서만 Mount 상태를 만든다.

예:

```bash
mount -t nfs 192.168.111.100:/nfs/share /mnt/nfs-share
```

이렇게만 하면 재부팅 후 Mount가 사라질 수 있다.

그래서 `/etc/fstab`에 등록하여 자동 Mount하도록 설정한다.

---

## 36. NFS fstab 설정

이번 실습:

```fstab
192.168.111.100:/nfs/share  /mnt/nfs-share  nfs  defaults,_netdev  0  0
```

구조:

```text
NFS Server:/공유경로
Mount Point
File System Type
Mount Option
dump
fsck
```

---

# _netdev

## 37. _netdev란?

```text
_netdev
```

는 해당 File System이 Network 연결을 필요로 한다는 것을 나타낸다.

NFS는 Local Disk가 아니라 Network File System이므로 Network가 준비되어야 Mount할 수 있다.

---

## 38. 왜 _netdev가 필요한가?

부팅 시:

```text
Local File System Mount
Network 시작
NFS Mount
```

순서가 중요할 수 있다.

NFS Server와 통신하려면 Network가 먼저 사용할 수 있어야 한다.

`_netdev`는 System에게:

```text
이 File System은 Network가 필요하다
```

는 정보를 전달한다.

---

# defaults

## 39. defaults

```text
defaults
```

는 일반적인 기본 Mount Option을 사용한다는 의미이다.

세부 옵션은 System과 File System에 따라 적용된다.

---

# 마지막 0 0

## 40. 첫 번째 0

fstab:

```text
0
```

첫 번째 숫자는 `dump` 관련 설정이다.

이번 NFS File System은 dump 대상에서 제외하였다.

---

## 41. 두 번째 0

두 번째:

```text
0
```

은 부팅 시 `fsck` 검사 순서와 관련된다.

NFS는 원격 File System이므로 Local Disk처럼 `fsck` 검사 대상으로 사용하지 않는다.

---

# mount -a

## 42. mount -a란?

```bash
mount -a
```

는 `/etc/fstab`에 정의된 Mount 대상들을 적용한다.

NFS 설정 후 재부팅 전에 fstab 문법과 Mount 가능 여부를 확인할 때 유용하다.

---

## 43. 왜 바로 reboot하지 않았는가?

`/etc/fstab`에 오타가 있으면 부팅 과정에서 문제가 발생할 수 있다.

따라서:

```text
fstab 수정
        ↓
umount
        ↓
mount -a
        ↓
정상 Mount 확인
        ↓
reboot
```

순서로 검증하였다.

---

# NFS Mount Option

## 44. rw

Mount 결과:

```text
rw
```

읽기 / 쓰기 가능.

---

## 45. hard

이번 실제 Mount 정보:

```text
hard
```

가 포함되었다.

`hard` Mount는 NFS Server가 응답하지 않을 때 NFS 요청을 계속 재시도하는 방식이다.

Data 일관성이 중요한 환경에서 일반적으로 많이 사용된다.

---

## 46. soft

반대 개념:

```text
soft
```

는 일정 횟수 이상 실패 시 Application에 Error를 반환할 수 있다.

잘못 사용할 경우 Application이 I/O 실패를 경험하거나 Data 처리 문제가 생길 수 있으므로 용도에 따라 신중하게 사용해야 한다.

---

## 47. proto=tcp

실제:

```text
proto=tcp
```

가 확인되었다.

즉 NFS 통신에 TCP를 사용하고 있다.

---

## 48. rsize

실제 Mount Option:

```text
rsize=524288
```

NFS Client가 Server에서 Data를 읽을 때 사용하는 최대 Read Request 크기와 관련된다.

---

## 49. wsize

```text
wsize=524288
```

NFS Client가 Server로 Data를 쓸 때 사용하는 최대 Write Request 크기와 관련된다.

---

## 50. timeo

```text
timeo=600
```

NFS 요청 Timeout 처리와 관련된 값이다.

---

## 51. retrans

```text
retrans=2
```

요청 실패 시 재전송과 관련된 Option이다.

---

# 파일 공유

## 52. Server → Client

Server-A:

```text
/nfs/share/server-test.txt
```

Client-L:

```text
/mnt/nfs-share/server-test.txt
```

Client에서 동일한 파일 내용을 확인하였다.

---

## 53. Client → Server

Client-L:

```text
/mnt/nfs-share/client-test.txt
```

생성 후 Server-A:

```text
/nfs/share/client-test.txt
```

에서 동일한 파일을 확인하였다.

---

## 54. 파일 복사인가?

아니다.

NFS Mount는 Server File을 Client Local Disk에 단순 복사하는 방식이 아니다.

```text
Server File
        ↓
Network를 통해 접근
        ↓
Client에서 Local File처럼 보임
```

구조이다.

따라서 동일한 NFS 공유를 여러 Client가 Mount하면 같은 Server Data를 함께 사용할 수 있다.

---

# NFS의 장점

## 55. 중앙 집중 Storage

여러 Linux Client가 하나의 Server Storage를 공유할 수 있다.

예:

```text
             NFS Server
             /share
               |
       -----------------
       |       |       |
       v       v       v
    Client1 Client2 Client3
```

모든 Client가 동일한 Data에 접근할 수 있다.

---

## 56. 관리 편의성

Data를 Server에 중앙 저장하면 여러 Client에 파일을 각각 복사할 필요가 줄어든다.

---

## 57. Linux 환경에서 자연스러운 File Sharing

NFS는 Linux / UNIX 환경에서 Network File Sharing에 자주 사용된다.

Client에서는 일반 Directory처럼 Mount하여 사용할 수 있다.

---

# NFS의 주의점

## 58. Network 의존성

NFS는 Network File System이므로 Network 장애가 발생하면 File 접근에도 영향을 준다.

```text
Network 장애
        ↓
NFS Server 통신 실패
        ↓
File 접근 문제
```

---

## 59. Server 장애

NFS Server가 중단되면 Client가 공유 Storage를 사용할 수 없다.

---

## 60. Permission 문제

NFS는 Linux Permission과 UID/GID의 영향을 받는다.

문제가 있을 경우:

```bash
ls -ln
id 사용자명
```

등으로 숫자 UID/GID를 확인하는 것이 중요하다.

---

# 장애 확인 순서

## 61. Client에서 NFS Mount 실패

먼저 Network 확인:

```bash
ping -c 3 192.168.111.100
```

---

## 62. Export 확인

```bash
showmount -e 192.168.111.100
```

공유 Directory가 보여야 한다.

---

## 63. Server Service 확인

Server-A:

```bash
systemctl is-active nfs-server
```

---

## 64. Export 설정 확인

```bash
cat /etc/exports
exportfs -v
```

---

## 65. Port 확인

```bash
ss -lntup | grep -E ':2049|:111'
```

---

## 66. Firewall 확인

```bash
firewall-cmd --list-services
```

확인 대상:

```text
nfs
rpc-bind
mountd
```

---

## 67. Client Mount 확인

```bash
mount | grep nfs-share
```

---

## 68. File System 확인

```bash
df -hT /mnt/nfs-share
```

---

## 69. Permission 확인

Server:

```bash
ls -ld /nfs/share
ls -l /nfs/share
```

Client:

```bash
ls -ld /mnt/nfs-share
ls -l /mnt/nfs-share
```

---

## 70. UID/GID 확인

```bash
id guest
```

Server와 Client에서 UID/GID가 일치하는지 확인한다.

---

# 권한 문제 예시

## 71. 파일 생성 실패

Client에서:

```text
Permission denied
```

가 발생한다면 다음을 확인한다.

```text
Server Directory Permission
/etc/exports의 rw/ro
UID/GID
root_squash
SELinux
Mount Option
```

---

# SELinux

## 72. SELinux와 NFS

Rocky Linux는 SELinux가 활성화되어 있을 수 있다.

NFS Service 자체는 정상인데 특정 Application이나 Directory 접근에서 문제가 생기면 SELinux 정책도 확인해야 한다.

상태 확인:

```bash
getenforce
```

Context 확인:

```bash
ls -Zd /nfs/share
```

실무 Troubleshooting에서는 Permission과 Firewall만 확인하고 끝내지 않고 SELinux까지 확인하는 습관이 중요하다.

---

# NFS와 Local Storage 차이

## 73. Local Disk

```text
Application
        ↓
Local File System
        ↓
Local Disk
```

Data가 현재 Host의 Disk에 저장된다.

---

## 74. NFS

```text
Application
        ↓
NFS Mount Point
        ↓
Network
        ↓
NFS Server
        ↓
Server Disk
```

Data가 원격 NFS Server의 Disk에 저장된다.

---

# 이번 실습 구조

## 75. Server 설정

```text
Server-A
192.168.111.100
        |
        v
/nfs/share
        |
        v
/etc/exports

/nfs/share 192.168.111.0/24(rw,sync,root_squash)
```

---

## 76. Client 설정

```text
Client-L
192.168.111.150
        |
        v
/mnt/nfs-share
```

수동 Mount:

```bash
mount -t nfs 192.168.111.100:/nfs/share /mnt/nfs-share
```

영구 Mount:

```fstab
192.168.111.100:/nfs/share  /mnt/nfs-share  nfs  defaults,_netdev  0  0
```

---

# 최종 동작

## 77. 전체 흐름

```text
                         Server-A
                      192.168.111.100
                             |
                             v
                        /nfs/share
                             |
                             v
                       /etc/exports
                             |
             rw / sync / root_squash
                             |
                             v
                         NFS Server
                             |
                          TCP 2049
                             |
                             v
                         Network
                             |
                             v
                         Client-L
                      192.168.111.150
                             |
                             v
                     /mnt/nfs-share
                             |
                             v
                         NFSv4.2
                             |
                             v
                      Local Directory
                       처럼 사용
```

---

# 핵심 정리

## 78. NFS

```text
NFS
=
원격 Server의 Directory를
Network를 통해
Local Directory처럼 사용하는 File System
```

---

## 79. Server와 Client

```text
NFS Server
→ Directory 공유

NFS Client
→ 공유 Directory Mount
```

---

## 80. Export와 Mount

```text
Server
/etc/exports
→ 무엇을 공유할지 설정
```

```text
Client
mount
→ 공유된 File System 연결
```

---

## 81. 이번 실습 핵심 Directory

```text
Server-A
/nfs/share
```

```text
Client-L
/mnt/nfs-share
```

---

## 82. 중요한 Export Option

```text
rw
→ 읽기 / 쓰기

sync
→ Storage 반영 후 응답

root_squash
→ Client root 권한 제한
```

---

## 83. 중요한 Mount 정보

```text
nfs4
→ NFSv4

vers=4.2
→ NFS Version 4.2

rw
→ 읽기 / 쓰기

hard
→ Server 응답 실패 시 계속 재시도

proto=tcp
→ TCP 통신

sec=sys
→ UID/GID 기반 UNIX 인증
```

---

## 84. 자동 Mount

```text
/etc/fstab
        ↓
defaults,_netdev
        ↓
Network File System으로 인식
        ↓
부팅 후 자동 Mount
```

---

## 85. 실제 저장 위치

Client에서:

```text
/mnt/nfs-share/file.txt
```

을 생성하면 NFS Mount 상태에서는 실제 Data가:

```text
Server-A
/nfs/share/file.txt
```

에 저장된다.

즉:

```text
Client에서 Local Directory처럼 보임
        ↓
실제 Data
        ↓
NFS Server Storage
```

이다.

---

## 86. Troubleshooting 핵심 순서

```text
1. Network
   ↓
2. NFS Server Service
   ↓
3. /etc/exports
   ↓
4. exportfs
   ↓
5. Firewall
   ↓
6. Port 2049 / 111
   ↓
7. showmount
   ↓
8. Client Mount
   ↓
9. Permission
   ↓
10. UID/GID
   ↓
11. SELinux
```

---

## 87. 최종 이해

NFS는 Server의 Directory를 Client에 복사하는 기술이 아니라, 원격 File System을 Network를 통해 Client의 Directory에 Mount하여 사용하는 방식이다.

이번 실습에서는:

```text
Server-A
/nfs/share
        ↓
NFS Export
        ↓
Client-L
/mnt/nfs-share
```

구조를 구성하였다.

또한 Client에서 생성한 파일이 Server의 `/nfs/share`에 실제로 저장되는 것을 확인하여 NFS의 원격 File System 동작을 검증하였다.

마지막으로 `/etc/fstab`에 NFS Mount 정보를 등록하고 Client-L 재부팅 후에도 자동으로 `/mnt/nfs-share`가 Mount되는 것을 확인하여 Persistent NFS Mount까지 구성하였다.
