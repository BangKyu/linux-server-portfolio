# Linux Storage 정리

Linux에서 새로운 디스크를 사용하기 위해서는 단순히 디스크를 장착하는 것만으로 끝나지 않는다.

일반적으로 다음 과정을 거쳐 저장 공간을 사용할 수 있다.

```text
디스크 추가
    ↓
디스크 확인
    ↓
Partition 생성
    ↓
File System 생성
    ↓
Mount Point 생성
    ↓
Mount
    ↓
필요한 경우 /etc/fstab 등록
    ↓
재부팅 후 자동 Mount
```

---

# 1. Disk와 Partition

## 1-1. Disk

Disk는 데이터를 저장하는 물리적 또는 가상 저장 장치이다.

Linux에서는 저장 장치가 `/dev` 아래의 장치 파일로 표현된다.

예:

```text
/dev/sda
/dev/sdb
/dev/sdc
```

NVMe 장치는 다음과 같은 형태로 표시될 수 있다.

```text
/dev/nvme0n1
```

현재 디스크 및 파티션 상태는 다음 명령어로 확인할 수 있다.

```bash
lsblk
```

또는:

```bash
fdisk -l
```

---

# 2. Partition이란?

Partition은 하나의 디스크 저장 공간을 논리적으로 나누어 사용하는 영역이다.

예를 들어 하나의 100G 디스크를 다음과 같이 나눌 수 있다.

```text
100G Disk
   │
   ├── 30G
   ├── 20G
   ├── 20G
   ├── 20G
   └── 10G
```

각 Partition에는 서로 다른 File System을 생성하거나,
서로 다른 디렉터리에 Mount하여 사용할 수 있다.

예:

```text
/dev/sdb1 → XFS
/dev/sdb5 → ext4
/dev/sdb6 → ext4
```

---

# 3. Partition을 나누는 이유

## 3-1. 저장 공간 분리

디렉터리별로 사용할 수 있는 저장 공간을 분리할 수 있다.

예:

```text
/        → 운영체제
/home    → 사용자 데이터
/var     → 로그 및 서비스 데이터
/boot    → 부팅 관련 파일
```

예를 들어 `/var/log`의 로그가 계속 증가하는 경우
`/var`가 `/`와 같은 파일시스템을 사용하면 전체 루트 파일시스템의 공간까지 부족해질 수 있다.

별도의 Partition으로 분리하면 특정 영역의 공간 사용을 독립적으로 관리할 수 있다.

---

## 3-2. 독립적인 설정

Partition마다 서로 다른 설정을 적용할 수 있다.

예:

- File System 종류
- Mount Option
- Quota
- Backup 정책
- 보안 정책

---

## 3-3. 관리 편의성

각 저장 영역을 분리하면 특정 파일시스템을 별도로 점검하거나 관리하기 쉬워진다.

---

# 4. MBR Partition 구조

이번 실습에서는 DOS(MBR) Partition Table을 사용하였다.

MBR 환경에서 사용하는 주요 Partition 종류는 다음과 같다.

```text
Primary Partition
Extended Partition
Logical Partition
```

---

## 4-1. Primary Partition

Primary Partition은 일반적으로 직접 File System을 생성하여 사용할 수 있는 파티션이다.

MBR Partition Table에서는 Primary와 Extended를 포함하여
총 4개의 Partition Entry를 사용할 수 있다.

Partition 번호:

```text
1
2
3
4
```

예:

```text
/dev/sdb1
/dev/sdb2
/dev/sdb3
/dev/sdb4
```

---

# 5. Extended Partition

Extended Partition은 Logical Partition을 만들기 위한 영역이다.

Extended Partition 자체를 일반적인 데이터 저장용 파티션으로 사용하는 것이 아니라,
내부에 여러 개의 Logical Partition을 생성하기 위한 Container 역할을 한다.

예:

```text
/dev/sdb2  → Extended
```

구조:

```text
Extended Partition
       │
       ├── Logical Partition
       ├── Logical Partition
       └── Logical Partition
```

---

# 6. Logical Partition

Logical Partition은 Extended Partition 내부에 생성되는 Partition이다.

MBR 환경에서는 Logical Partition 번호가 일반적으로 5번부터 시작한다.

예:

```text
/dev/sdb5
/dev/sdb6
/dev/sdb7
/dev/sdb8
```

이번 실습의 구조:

```text
/dev/sdb 100G
│
├── /dev/sdb1  30G  Primary
│
└── /dev/sdb2  70G  Extended
     │
     ├── /dev/sdb5  20G  Logical
     ├── /dev/sdb6  20G  Logical
     ├── /dev/sdb7  20G  Logical
     └── /dev/sdb8  10G  Logical
```

정리:

```text
Primary  → 직접 사용할 수 있는 파티션
Extended → Logical Partition을 담는 영역
Logical  → Extended 내부에서 실제로 사용하는 파티션
```

---

# 7. fdisk

`fdisk`는 Disk Partition을 생성하거나 확인하는 데 사용하는 명령어이다.

현재 Partition 확인:

```bash
fdisk -l
```

특정 Disk의 Partition 정보 확인:

```bash
fdisk -l /dev/sdb
```

Partition 설정:

```bash
fdisk /dev/sdb
```

주요 명령:

| 명령 | 의미 |
|---|---|
| `n` | 새로운 Partition 생성 |
| `p` | 현재 Partition Table 확인 / Primary 선택 시 사용 |
| `e` | Extended Partition 선택 |
| `d` | Partition 삭제 |
| `w` | 변경 사항 저장 후 종료 |
| `q` | 저장하지 않고 종료 |

`fdisk`에서 설정한 내용은 `w`를 실행하기 전까지 실제 Partition Table에 최종 저장되지 않는다.

---

# 8. File System

File System은 운영체제가 저장 장치에 데이터를 저장하고 관리하는 방식이다.

File System은 다음과 같은 정보를 관리한다.

- 파일 이름
- 파일 크기
- 파일 위치
- 파일 권한
- 소유자
- 생성 및 수정 시간
- 디렉터리 구조

Partition만 생성했다고 바로 파일을 저장할 수 있는 것은 아니다.

```text
Partition 생성
     ↓
File System 생성
     ↓
파일 저장 가능
```

---

# 9. 주요 Linux File System

## 9-1. ext4

ext4는 Linux에서 널리 사용하는 범용 File System이다.

특징:

- 범용적인 Linux File System
- 안정적인 사용
- Journaling 지원
- 일반적인 서버 및 Linux 환경에서 사용 가능

생성:

```bash
mkfs.ext4 /dev/sdb5
```

또는:

```bash
mkfs -t ext4 /dev/sdb5
```

---

## 9-2. XFS

XFS는 확장성과 성능을 고려한 Linux File System이다.

Rocky Linux 계열 환경에서도 많이 사용된다.

특징:

- 대용량 데이터 처리에 적합
- 높은 확장성
- Journaling 지원
- 서버 환경에서 많이 사용

생성:

```bash
mkfs.xfs /dev/sdb1
```

또는:

```bash
mkfs -t xfs /dev/sdb1
```

---

# 10. SWAP

SWAP은 Disk의 일부 공간을 메모리 보조 공간으로 사용하는 영역이다.

실제 RAM과 동일한 성능을 제공하는 것은 아니며,
RAM보다 Disk의 접근 속도가 훨씬 느리다.

현재 SWAP 확인:

```bash
swapon --show
```

또는:

```bash
free -h
```

`lsblk -f`에서도 SWAP Partition을 확인할 수 있다.

예:

```text
sda1 swap [SWAP]
```

---

# 11. Journaling File System

Journaling은 파일시스템 변경 사항과 관련된 정보를 Journal(Log)에 기록하여
비정상적인 시스템 종료 이후 파일시스템을 복구하는 데 도움을 주는 기능이다.

개념:

```text
File System 변경
       ↓
Journal 기록
       ↓
실제 File System에 반영
```

갑작스러운 전원 차단이나 시스템 장애가 발생했을 때
Journal 정보를 이용하여 파일시스템의 일관성을 복구하는 데 도움을 줄 수 있다.

ext4와 XFS 모두 Journaling 기능을 지원한다.

---

# 12. mkfs

`mkfs`는 Partition 또는 Disk에 File System을 생성하는 명령어이다.

형식:

```bash
mkfs -t [파일시스템] [장치]
```

예:

```bash
mkfs -t xfs /dev/sdb1
mkfs -t ext4 /dev/sdb5
```

전용 명령을 사용할 수도 있다.

```bash
mkfs.xfs /dev/sdb1
mkfs.ext4 /dev/sdb5
```

주의:

```text
mkfs는 기존 파일시스템과 데이터를 손상시킬 수 있으므로
대상 장치를 반드시 확인한 후 실행해야 한다.
```

특히 운영체제가 설치된 System Disk에 실수로 실행하지 않도록 주의한다.

---

# 13. lsblk

`lsblk`는 Linux의 Block Device를 확인하는 명령어이다.

Disk와 Partition 구조를 확인할 때 사용한다.

```bash
lsblk
```

예:

```text
sdb
├─sdb1
├─sdb2
├─sdb5
├─sdb6
├─sdb7
└─sdb8
```

File System과 UUID까지 확인:

```bash
lsblk -f
```

주요 확인 항목:

```text
NAME
FSTYPE
UUID
FSAVAIL
FSUSE%
MOUNTPOINTS
```

---

# 14. df

`df`는 현재 Mount되어 사용 중인 File System의 용량 상태를 확인하는 명령어이다.

기본 확인:

```bash
df
```

사람이 읽기 쉬운 단위로 확인:

```bash
df -h
```

File System 종류 포함:

```bash
df -T
```

File System 종류 + 읽기 쉬운 용량:

```bash
df -hT
```

---

# 15. lsblk와 df의 차이

두 명령어는 확인 목적이 조금 다르다.

## lsblk

Block Device 관점에서 확인한다.

```bash
lsblk
```

Mount되지 않은 Partition도 확인할 수 있다.

```text
Disk
 └── Partition
```

---

## df

현재 Mount된 File System 관점에서 확인한다.

```bash
df -hT
```

즉:

```text
lsblk
→ 디스크 및 파티션 구조 확인

df
→ 현재 사용 중인 Mount된 파일시스템 용량 확인
```

---

# 16. Mount

Mount는 File System을 Linux의 특정 Directory에 연결하는 작업이다.

Linux에서는 Disk에 File System을 생성했더라도
Directory에 Mount하지 않으면 일반적인 경로를 통해 사용할 수 없다.

구조:

```text
/dev/sdb1
   │
   │ Mount
   ▼
 /GIT
```

이후 `/GIT`에 저장하는 데이터는 `/dev/sdb1`의 File System에 저장된다.

---

# 17. Mount Point

Mount Point는 File System을 연결할 Directory이다.

먼저 Directory를 생성한다.

```bash
mkdir -p /GIT
```

그다음 File System을 연결한다.

```bash
mount /dev/sdb1 /GIT
```

형식:

```bash
mount [장치] [마운트할 디렉터리]
```

---

# 18. umount

Mount된 File System의 연결을 해제할 때 `umount`를 사용한다.

장치명을 이용:

```bash
umount /dev/sdb1
```

Mount Point를 이용:

```bash
umount /GIT
```

주의:

명령어 이름은 `unmount`가 아니라 다음과 같다.

```text
umount
```

---

# 19. 수동 Mount

다음과 같이 직접 `mount` 명령을 실행하는 것을 수동 Mount라고 할 수 있다.

```bash
mount /dev/sdb1 /GIT
```

수동 Mount만 한 상태에서는 시스템을 재부팅했을 때
해당 장치가 자동으로 다시 Mount되지 않는다.

따라서 부팅할 때 자동으로 Mount되도록 설정하려면
`/etc/fstab`을 사용할 수 있다.

---

# 20. 자동 Mount

부팅 과정에서 File System을 자동으로 Mount하려면
`/etc/fstab`에 Mount 정보를 등록한다.

파일:

```text
/etc/fstab
```

설정 확인:

```bash
cat /etc/fstab
```

편집:

```bash
vi /etc/fstab
```

---

# 21. UUID

UUID는 File System을 식별하기 위한 고유 식별값이다.

확인 방법:

```bash
lsblk -f
```

또는:

```bash
blkid
```

예:

```text
/dev/sdb1
UUID=b94248cd-9d51-4e99-a08f-627c647b9268
TYPE=xfs
```

`/etc/fstab`에는 다음과 같이 장치명을 직접 사용할 수도 있다.

```text
/dev/sdb1 /GIT xfs defaults 0 0
```

하지만 UUID를 이용하여 File System을 지정하는 방식도 사용할 수 있다.

```text
UUID=b94248cd-9d51-4e99-a08f-627c647b9268 /GIT xfs defaults 0 0
```

이번 실습에서는 UUID 방식으로 설정하였다.

---

# 22. /etc/fstab 구조

기본 형식:

```text
장치   마운트포인트   파일시스템   옵션   dump   fsck
```

예:

```text
UUID=b94248cd-9d51-4e99-a08f-627c647b9268 /GIT xfs defaults 0 0
```

각 필드:

```text
UUID=...     /GIT      xfs       defaults       0        0
   │           │        │            │          │        │
   │           │        │            │          │        └─ fsck 검사 순서
   │           │        │            │          └────────── dump 설정
   │           │        │            └───────────────────── Mount Option
   │           │        └────────────────────────────────── File System
   │           └─────────────────────────────────────────── Mount Point
   └─────────────────────────────────────────────────────── 장치/File System 식별
```

---

# 23. /etc/fstab 1번째 필드

Mount할 장치 또는 File System을 지정한다.

장치명을 사용하는 방법:

```text
/dev/sdb1
```

UUID를 사용하는 방법:

```text
UUID=b94248cd-9d51-4e99-a08f-627c647b9268
```

---

# 24. /etc/fstab 2번째 필드

Mount Point를 지정한다.

예:

```text
/GIT
/homeSK
/homeLG/user1
```

해당 Directory가 미리 존재해야 한다.

---

# 25. /etc/fstab 3번째 필드

File System 종류를 지정한다.

예:

```text
xfs
ext4
swap
```

---

# 26. /etc/fstab 4번째 필드

Mount Option을 설정한다.

일반적인 기본값:

```text
defaults
```

주요 Mount Option:

| 옵션 | 의미 |
|---|---|
| `defaults` | 일반적인 기본 Mount 옵션 사용 |
| `rw` | 읽기/쓰기 허용 |
| `ro` | 읽기 전용 |
| `exec` | 실행 파일 실행 허용 |
| `noexec` | 실행 파일 실행 제한 |
| `user` | 일반 사용자의 Mount 허용 |
| `nouser` | 일반 사용자의 Mount 제한 |
| `nosuid` | SUID/SGID 효과 제한 |

---

# 27. /etc/fstab 5번째 필드

`dump` 백업과 관련된 설정이다.

```text
0 → dump 대상에서 제외
1 → dump 대상에 포함
```

일반적인 설정에서 `0`을 많이 확인할 수 있다.

---

# 28. /etc/fstab 6번째 필드

부팅 과정에서 File System을 점검할 때 사용하는 `fsck` 검사 순서와 관련된 필드이다.

```text
0 → 자동 fsck 검사 대상에서 제외
1 → 우선 검사
2 → 그 이후 검사
```

일반적으로 `/`와 다른 File System의 값이 다르게 구성될 수 있다.

File System 종류와 시스템 구성에 따라 적절하게 설정해야 한다.

---

# 29. /etc/fstab 수정 후 확인

`/etc/fstab`은 부팅 과정에 영향을 주는 중요한 설정 파일이므로
수정하기 전에 백업하는 것이 안전하다.

예:

```bash
cp -p /etc/fstab /etc/fstab.bak
```

수정:

```bash
vi /etc/fstab
```

Rocky Linux의 `/etc/fstab` 안내에 따라 설정 변경 후:

```bash
systemctl daemon-reload
```

설정된 File System Mount 테스트:

```bash
mount -a
```

`mount -a`는 `/etc/fstab`을 기준으로 Mount 가능한 항목을 Mount한다.

오류가 출력된다면 재부팅하기 전에 `/etc/fstab` 설정을 다시 확인한다.

---

# 30. 자동 Mount 확인 방법

설정 후 다음 명령으로 확인한다.

```bash
lsblk -f
```

또는:

```bash
df -hT
```

최종적으로 재부팅하여 확인할 수 있다.

```bash
reboot
```

재부팅 후 별도의 `mount` 명령 없이:

```bash
lsblk -f
df -hT
```

를 실행한다.

Mount Point가 정상적으로 표시된다면
`/etc/fstab`을 이용한 자동 Mount가 정상적으로 적용된 것이다.

---

# 31. Mount 시 주의할 점

이미 파일이 존재하는 Directory 위에 다른 File System을 Mount하면
기존 Directory의 파일이 삭제되는 것은 아니지만 Mount 상태에서는 보이지 않게 된다.

예:

```text
/home/user1
  ├── .bashrc
  └── data.txt
```

여기에 다른 File System을 Mount하면:

```bash
mount /dev/sdb6 /home/user1
```

Mount된 File System의 내용이 `/home/user1`에서 보이게 되므로
기존 파일은 Mount가 유지되는 동안 가려진다.

따라서 기존 데이터가 존재하는 Directory를 Mount Point로 사용할 때는 주의해야 한다.

---

# 32. Disk를 사용하는 전체 과정

새로운 Disk를 추가했을 때 전체 흐름:

```text
1. Disk 추가
      ↓
2. lsblk / fdisk -l
   장치 확인
      ↓
3. fdisk
   Partition 생성
      ↓
4. mkfs
   File System 생성
      ↓
5. mkdir
   Mount Point 생성
      ↓
6. mount
   File System 연결
      ↓
7. lsblk -f / df -hT
   상태 확인
      ↓
8. /etc/fstab
   자동 Mount 설정
      ↓
9. mount -a
   설정 테스트
      ↓
10. reboot
    자동 Mount 최종 확인
```

---

# 33. 주요 명령어 정리

| 명령어 | 용도 |
|---|---|
| `lsblk` | Disk / Partition 구조 확인 |
| `lsblk -f` | File System, UUID, Mount Point 확인 |
| `fdisk -l` | Partition Table 확인 |
| `fdisk /dev/sdX` | Partition 설정 |
| `mkfs.xfs` | XFS File System 생성 |
| `mkfs.ext4` | ext4 File System 생성 |
| `mount` | File System 연결 |
| `umount` | Mount 해제 |
| `df -hT` | Mount된 File System의 용량 및 종류 확인 |
| `blkid` | UUID 및 File System 정보 확인 |
| `mount -a` | `/etc/fstab` 설정에 따라 Mount |
| `systemctl daemon-reload` | systemd 설정 다시 읽기 |

---

# 34. 핵심 정리

```text
Disk
↓
실제 저장 장치

Partition
↓
Disk의 저장 공간을 논리적으로 분할

File System
↓
Partition에 데이터를 저장하고 관리하는 방식
예: XFS, ext4

Mount
↓
File System을 Linux Directory와 연결

Mount Point
↓
File System이 연결되는 Directory

UUID
↓
File System을 식별하는 고유값

/etc/fstab
↓
부팅 시 File System을 자동으로 Mount하기 위한 설정 파일
```

가장 중요한 흐름:

```text
Disk 추가
→ Partition
→ File System
→ Mount Point
→ Mount
→ /etc/fstab
→ 재부팅 후 자동 Mount 확인
```

---

