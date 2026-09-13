# LVM 이론 정리

## 1. LVM이란?

LVM(Logical Volume Manager)은  
Linux에서 저장장치의 공간을 논리적으로 관리하기 위한 기능이다.

일반 파티션 방식에서는 디스크를 파티션으로 나누면 각 파티션의 크기가 고정된다.

```text
Disk
├── Partition 1
├── Partition 2
└── Partition 3
```

파티션의 공간이 부족해졌을 때 크기를 변경하거나  
다른 디스크의 공간을 추가하여 사용하는 과정이 불편할 수 있다.

LVM을 사용하면 물리적인 디스크나 파티션을 하나의 저장 공간으로 묶고  
그 공간에서 필요한 크기의 논리 볼륨을 생성하여 사용할 수 있다.

```text
Disk / Partition
       ↓
      PV
       ↓
      VG
       ↓
      LV
       ↓
Filesystem
       ↓
    Mount
```

LVM의 핵심 목적은 **저장 공간을 유연하게 구성하고 확장하는 것**이다.

---

# 2. LVM 기본 구조

LVM은 크게 다음 구조로 구성된다.

```text
Physical Disk
     ↓
Partition
     ↓
PV (Physical Volume)
     ↓
VG (Volume Group)
     ↓
LV (Logical Volume)
     ↓
Filesystem
     ↓
Mount Point
```

예:

```text
/dev/sdc1 ── PV ─┐
                 │
/dev/sdd1 ── PV ─┼── VG : SOLLVM
                 │
/dev/sde1 ── PV ─┘
                      │
                      ├── LV1 → ext4 → /CU
                      ├── LV2 → ext4 → /GS
                      └── LV3 → ext4 → /LG
```

---

# 3. PV (Physical Volume)

## 3-1. PV란?

PV(Physical Volume)는  
LVM에서 사용할 수 있도록 초기화된 물리 저장장치 또는 파티션이다.

예:

```text
/dev/sdc1
/dev/sdd1
/dev/sde1
```

일반 파티션을 바로 VG에 넣는 것이 아니라  
먼저 `pvcreate`를 이용하여 PV로 초기화한다.

```bash
pvcreate /dev/sdc1
```

여러 개를 동시에 생성할 수도 있다.

```bash
pvcreate /dev/sdc1 /dev/sdd1
```

---

## 3-2. PV 확인

간단한 상태 확인:

```bash
pvs
```

상세 정보 확인:

```bash
pvdisplay
```

주요 항목:

| 항목 | 의미 |
|---|---|
| PV | Physical Volume 장치 |
| VG | 해당 PV가 속한 Volume Group |
| PSize | PV 전체 크기 |
| PFree | 아직 할당되지 않은 공간 |
| PV UUID | PV의 고유 식별자 |

PV를 생성했지만 아직 VG에 포함하지 않은 경우  
`VG` 항목이 비어 있을 수 있다.

---

# 4. VG (Volume Group)

## 4-1. VG란?

VG(Volume Group)는 여러 PV를 하나의 논리적인 저장 공간으로 묶은 것이다.

예를 들어:

```text
/dev/sdc1 = 약 10GB
/dev/sdd1 = 약 10GB
```

두 PV를 하나의 VG로 묶으면:

```text
/dev/sdc1 ─┐
           ├── SOLLVM ≈ 20GB
/dev/sdd1 ─┘
```

처럼 사용할 수 있다.

VG 생성:

```bash
vgcreate SOLLVM /dev/sdc1 /dev/sdd1
```

---

## 4-2. VG 확인

간단한 상태 확인:

```bash
vgs
```

상세 정보 확인:

```bash
vgdisplay
```

`vgs` 주요 항목:

| 항목 | 의미 |
|---|---|
| VG | Volume Group 이름 |
| #PV | VG에 포함된 PV 개수 |
| #LV | VG에 생성된 LV 개수 |
| #SN | Snapshot 개수 |
| VSize | VG 전체 크기 |
| VFree | 아직 LV에 할당되지 않은 공간 |

예:

```text
VG      #PV  #LV  VSize    VFree
SOLLVM    3    3  <29.99g  <9.00g
```

의미:

```text
SOLLVM에 PV 3개 존재
LV 3개 존재
전체 약 30GB
약 9GB 여유 공간 존재
```

---

# 5. LV (Logical Volume)

## 5-1. LV란?

LV(Logical Volume)는 VG의 공간을 필요한 크기로 나누어 만든 논리 볼륨이다.

일반 파티션과 비슷하게 파일시스템을 생성하고 마운트하여 사용할 수 있다.

예:

```text
SOLLVM
├── 8G_LV1
├── 6G_LV2
└── 6G_LV3
```

LV 생성:

```bash
lvcreate -L 8G -n 8G_LV1 SOLLVM
```

옵션:

```text
-L
```

생성할 LV의 크기를 지정한다.

```text
-n
```

LV의 이름을 지정한다.

---

## 5-2. 남은 공간 전체 사용

VG에 남아 있는 공간을 전부 LV에 할당하려면:

```bash
lvcreate -l 100%FREE -n 6G_LV3 SOLLVM
```

여기서:

```text
-l 100%FREE
```

는 VG의 남은 Extent를 모두 사용한다는 의미이다.

---

## 5-3. LV 확인

간단한 상태 확인:

```bash
lvs
```

상세 정보 확인:

```bash
lvdisplay
```

예:

```text
LV      VG      LSize
6G_LV2  SOLLVM  6.00g
6G_LV3  SOLLVM  5.99g
8G_LV1  SOLLVM  9.00g
```

LV 이름과 실제 용량은 반드시 같을 필요가 없다.

예를 들어:

```text
LV 이름     : 8G_LV1
실제 LV 크기 : 9G
```

처럼 LV를 나중에 확장하여도 이름은 자동으로 변경되지 않는다.

---

# 6. PE (Physical Extent)

PE(Physical Extent)는 LVM에서 공간을 할당하는 기본 단위이다.

VG는 내부적으로 공간을 일정한 크기의 PE로 나누어 관리한다.

이번 실습 환경에서는 PE 크기가 다음과 같이 확인되었다.

```text
PE Size = 4.00 MiB
```

즉 약 10GB의 PV는 여러 개의 4MiB PE로 나뉘어 관리된다.

```text
PV
├── PE
├── PE
├── PE
├── PE
├── ...
└── PE
```

LV를 생성하면 이러한 Extent들이 LV에 할당된다.

### 참고

4MiB는 일반적으로 사용하는 기본 PE 크기이며  
PE 크기는 VG 생성 시 설정에 따라 달라질 수 있다.

---

# 7. PV → VG → LV 관계

가장 중요한 구조는 다음과 같다.

```text
물리 Disk
   ↓
Partition
   ↓
PV
   ↓
VG
   ↓
LV
   ↓
Filesystem
   ↓
Mount Point
```

예:

```text
/dev/sdc
   ↓
/dev/sdc1
   ↓
PV
   │
   ├──────────────┐
   │              │
/dev/sdd1(PV)     │
   │              │
   └──── SOLLVM ──┘
             │
             ├── 8G_LV1
             ├── 6G_LV2
             └── 6G_LV3
```

정리하면:

```text
PV = 실제 저장 공간을 LVM에서 사용할 수 있도록 만든 단위

VG = 여러 PV를 합쳐 만든 저장 공간 Pool

LV = VG에서 필요한 만큼 나누어 만든 논리 볼륨
```

---

# 8. LVM 구성 순서

일반적인 LVM 구성 과정:

```text
1. 디스크 확인
        ↓
2. 파티션 생성
        ↓
3. PV 생성
        ↓
4. VG 생성
        ↓
5. LV 생성
        ↓
6. 파일시스템 생성
        ↓
7. 마운트
        ↓
8. /etc/fstab 등록
```

명령어로 표현하면:

```bash
lsblk

fdisk /dev/sdc

pvcreate /dev/sdc1

vgcreate SOLLVM /dev/sdc1

lvcreate -L 8G -n 8G_LV1 SOLLVM

mkfs.ext4 /dev/SOLLVM/8G_LV1

mkdir /CU

mount /dev/SOLLVM/8G_LV1 /CU
```

---

# 9. LVM 파티션 타입

이번 실습에서는 DOS/MBR 파티션 테이블을 사용하였다.

LVM용 파티션 타입:

```text
8e Linux LVM
```

예:

```bash
fdisk /dev/sdc
```

```text
n
t
8e
w
```

## 주의

`8e`는 **DOS/MBR 파티션 테이블에서 사용하는 Linux LVM 파티션 타입**이다.

GPT 파티션 테이블에서는 MBR의 `8e` 방식과 동일하게 생각하면 안 되며  
GPT용 Linux LVM 파티션 타입을 사용한다.

또한 LVM은 반드시 파티션만 사용할 수 있는 것은 아니며  
환경에 따라 전체 블록 장치를 PV로 사용할 수도 있다.

실습에서는 구조를 명확하게 확인하기 위해:

```text
Disk → Partition → PV
```

방식을 사용하였다.

---

# 10. LV와 파일시스템은 서로 다른 계층

LVM에서 매우 중요한 개념이다.

```text
LV
↓
Filesystem
↓
Mount
```

LV는 저장 공간을 제공하는 논리 블록 장치이고  
ext4, XFS 등은 그 위에 생성되는 파일시스템이다.

따라서:

```bash
lvextend
```

로 LV를 확장했다고 해서 항상 파일시스템까지 자동으로 커지는 것은 아니다.

예:

```text
LV
8GB → 9GB

Filesystem
기존 크기 유지
```

따라서 파일시스템도 별도로 확장해야 한다.

---

# 11. ext4 파일시스템 생성

LV 생성 후 ext4 파일시스템을 생성할 수 있다.

```bash
mkfs.ext4 /dev/SOLLVM/8G_LV1
```

다른 LV도 동일하다.

```bash
mkfs.ext4 /dev/SOLLVM/6G_LV2
mkfs.ext4 /dev/SOLLVM/6G_LV3
```

## 주의

`mkfs`는 새로운 파일시스템을 생성하는 명령이다.

기존 데이터가 존재하는 장치에 잘못 실행하면  
데이터가 손상될 수 있으므로 대상 장치를 반드시 확인해야 한다.

---

# 12. LVM 장치 경로

LVM의 LV는 다음과 같은 형태로 접근할 수 있다.

```text
/dev/<VG>/<LV>
```

예:

```text
/dev/SOLLVM/8G_LV1
/dev/SOLLVM/6G_LV2
/dev/SOLLVM/6G_LV3
```

Device Mapper 경로로는 다음처럼 보일 수 있다.

```text
/dev/mapper/SOLLVM-8G_LV1
/dev/mapper/SOLLVM-6G_LV2
/dev/mapper/SOLLVM-6G_LV3
```

따라서:

```bash
lsblk
```

또는:

```bash
df -hT
```

에서는 `/dev/mapper/...` 형태로 나타날 수 있다.

---

# 13. 마운트

파일시스템을 생성한 LV를 디렉터리에 연결한다.

마운트 디렉터리 생성:

```bash
mkdir -p /CU
mkdir -p /GS
mkdir -p /LG
```

마운트:

```bash
mount /dev/SOLLVM/8G_LV1 /CU
mount /dev/SOLLVM/6G_LV2 /GS
mount /dev/SOLLVM/6G_LV3 /LG
```

확인:

```bash
findmnt /CU
findmnt /GS
findmnt /LG
```

또는:

```bash
df -hT
```

---

# 14. LVM과 `/etc/fstab`

수동 `mount` 명령으로 마운트한 상태는  
재부팅 후 자동으로 다시 마운트된다는 보장이 없다.

부팅할 때 자동으로 마운트하려면 `/etc/fstab`에 등록한다.

UUID 확인:

```bash
lsblk -f
```

또는:

```bash
blkid
```

예:

```text
UUID=<LV의 파일시스템 UUID>  /CU  ext4  defaults  0 2
```

설정 후:

```bash
systemctl daemon-reload
mount -a
```

오류가 없는지 확인하고 재부팅 후 다시 확인한다.

```bash
findmnt /CU
```

## 주의

`/etc/fstab`을 잘못 작성하면 부팅 과정에 문제가 발생할 수 있으므로  
수정 후 바로 재부팅하지 말고 먼저:

```bash
mount -a
```

등으로 설정을 확인하는 것이 안전하다.

---

# 15. VG 용량 확장

기존 VG의 여유 공간이 부족하면  
새로운 디스크 또는 파티션을 PV로 만든 뒤 기존 VG에 추가할 수 있다.

예:

```text
기존

/dev/sdc1 ─┐
           ├── SOLLVM ≈ 20GB
/dev/sdd1 ─┘

추가 디스크

/dev/sde1 ≈ 10GB
```

먼저 새 장치를 PV로 생성한다.

```bash
pvcreate /dev/sde1
```

그다음 기존 VG에 추가한다.

```bash
vgextend SOLLVM /dev/sde1
```

결과:

```text
/dev/sdc1 ─┐
/dev/sdd1 ─┼── SOLLVM ≈ 30GB
/dev/sde1 ─┘
```

확인:

```bash
pvs
vgs
```

---

# 16. LV 용량 확장

VG에 여유 공간이 존재하면 기존 LV에 공간을 추가할 수 있다.

예:

```text
8G_LV1

8GB → 9GB
```

1GB 추가:

```bash
lvextend -L +1G /dev/SOLLVM/8G_LV1
```

확인:

```bash
lvs
vgs
```

`VG`의 여유 공간에서 1GB가 LV에 할당되므로:

```text
VFree 약 10GB
       ↓
VFree 약 9GB
```

처럼 감소한다.

---

# 17. ext4 파일시스템 확장

LV의 크기를 늘린 후에는  
그 위에 있는 ext4 파일시스템의 크기도 확장해야 한다.

```bash
resize2fs /dev/SOLLVM/8G_LV1
```

ext4는 확장 작업을 온라인 상태에서 수행할 수 있으므로  
일반적인 확장 상황에서는 마운트된 상태에서도 증가시킬 수 있다.

확인:

```bash
df -hT /CU
```

전체 흐름:

```text
VG 여유 공간
     ↓
lvextend
     ↓
LV 크기 증가
     ↓
resize2fs
     ↓
ext4 파일시스템 크기 증가
     ↓
df로 확인
```

---

# 18. 파일시스템에 따른 확장 명령 차이

LV를 확장한 뒤 어떤 명령을 사용하는지는  
LV 위에 생성된 파일시스템 종류에 따라 달라진다.

## ext4

```bash
lvextend -L +1G /dev/VG/LV
resize2fs /dev/VG/LV
```

## XFS

LV를 먼저 확장한 뒤 XFS 파일시스템을 확장한다.

```bash
lvextend -L +1G /dev/VG/LV
xfs_growfs <마운트포인트>
```

예:

```bash
xfs_growfs /DATA
```

따라서 파일시스템 종류를 먼저 확인해야 한다.

```bash
lsblk -f
df -hT
```

---

# 19. `lvextend -r`

LV와 파일시스템을 한 번에 확장하는 방법도 있다.

예:

```bash
lvextend -r -L +1G /dev/SOLLVM/8G_LV1
```

`-r` 옵션을 사용하면 지원되는 파일시스템에 대해  
LV 확장 후 파일시스템 크기 조정도 함께 수행한다.

하지만 이번 실습에서는 구조를 이해하기 위해:

```text
lvextend
    ↓
resize2fs
```

를 각각 따로 수행하였다.

이를 통해:

```text
LV의 크기
```

와

```text
파일시스템의 크기
```

가 서로 다른 계층이라는 것을 확인할 수 있다.

---

# 20. LV 축소

LV 확장은 비교적 일반적으로 사용하지만  
**LV 축소는 훨씬 더 주의해야 한다.**

축소 순서를 잘못 처리하면 파일시스템의 데이터 영역보다  
LV 자체를 먼저 작게 만들어 데이터가 손상될 수 있다.

ext4의 일반적인 축소 흐름:

```text
백업
 ↓
umount
 ↓
e2fsck
 ↓
resize2fs로 파일시스템 축소
 ↓
lvreduce로 LV 축소
 ↓
mount
 ↓
검증
```

예:

```bash
umount /GS
```

파일시스템 검사:

```bash
e2fsck -f /dev/SOLLVM/6G_LV2
```

파일시스템을 먼저 축소:

```bash
resize2fs /dev/SOLLVM/6G_LV2 5G
```

그다음 LV 축소:

```bash
lvreduce -L 5G /dev/SOLLVM/6G_LV2
```

마운트:

```bash
mount /dev/SOLLVM/6G_LV2 /GS
```

## 매우 중요

```text
파일시스템 축소
        ↓
LV 축소
```

순서를 지켜야 한다.

반대로 LV부터 지나치게 작게 줄이면  
파일시스템 데이터가 잘릴 수 있다.

따라서 중요한 데이터가 있는 LV를 축소하기 전에는  
반드시 백업하고 절차를 확인해야 한다.

이번 포트폴리오 실습에서는 안전을 위해 실제 LV 축소는 진행하지 않았다.

---

# 21. XFS 축소 주의

XFS는 파일시스템 확장은 가능하지만  
일반적인 방법으로 파일시스템 크기를 축소할 수 없다.

따라서:

```text
ext4
확장 가능
축소 가능(오프라인 절차 필요)

XFS
확장 가능
축소 불가
```

라고 구분해서 기억한다.

LVM LV 자체가 축소 가능한 것과  
그 위의 파일시스템이 축소 가능한 것은 별개의 문제이다.

즉:

```text
LVM이 가능하다고 해서
모든 파일시스템이 동일하게 가능한 것은 아니다.
```

---

# 22. LVM과 RAID 차이

LVM과 RAID는 디스크를 묶어서 사용할 수 있다는 점에서는 비슷해 보이지만  
주요 목적이 다르다.

| 구분 | LVM | RAID |
|---|---|---|
| 주요 목적 | 유연한 저장 공간 관리 | 성능 또는 장애 허용 |
| 여러 디스크 사용 | 가능 | 가능 |
| 용량 확장 | 주요 기능 중 하나 | RAID 구성에 따라 다름 |
| LV 생성 | 가능 | 해당 개념 없음 |
| 기본적인 데이터 중복 저장 | 없음 | RAID Level에 따라 가능 |
| 장애 허용 | LVM 단독으로 보장하지 않음 | RAID Level에 따라 가능 |

예:

```text
LVM
디스크 공간 관리 및 확장에 초점

RAID
여러 디스크를 이용한 성능/장애 허용 등에 초점
```

LVM과 RAID는 서로 배타적인 기술이 아니며  
환경에 따라 함께 사용할 수도 있다.

## 주의

LVM 자체를 사용한다고 해서  
디스크 장애로부터 데이터가 자동으로 보호되는 것은 아니다.

또한 RAID 역시 **백업을 대신하는 기술은 아니다.**

---

# 23. LVM의 장점

### 1) 저장 공간 관리가 유연함

여러 PV를 하나의 VG로 구성하여 관리할 수 있다.

```text
Disk A ─┐
Disk B ─┼── VG
Disk C ─┘
```

---

### 2) 필요에 따라 LV 생성 가능

VG에서 필요한 크기만큼 LV를 나누어 사용할 수 있다.

```text
VG
├── LV1
├── LV2
└── LV3
```

---

### 3) 새로운 저장장치를 기존 VG에 추가 가능

```bash
pvcreate /dev/sde1
vgextend SOLLVM /dev/sde1
```

기존 VG에 새로운 용량을 추가할 수 있다.

---

### 4) 기존 LV 확장 가능

```bash
lvextend -L +1G /dev/SOLLVM/8G_LV1
```

기존 LV를 삭제하고 새로 만들지 않고도  
VG의 여유 공간을 이용해 용량을 확장할 수 있다.

---

# 24. LVM 사용 시 주의할 점

LVM은 용량 관리 기능이지  
그 자체가 데이터 백업 기능은 아니다.

따라서:

```text
LVM = Backup
```

이 아니다.

또한 여러 디스크의 공간을 하나의 LV가 사용하고 있는 경우  
해당 LV의 데이터가 여러 PV에 걸쳐 존재할 수 있다.

따라서 실제 서버에서는:

- 디스크 장애 대책
- RAID 구성 여부
- 백업 정책
- 파일시스템 종류
- 복구 계획

등을 함께 고려해야 한다.

---

# 25. 하나의 LV가 여러 PV를 사용할 수 있음

LVM에서는 LV가 반드시 하나의 물리 디스크 안에만 존재하는 것은 아니다.

예:

```text
PV1 (/dev/sdc1)
        │
        ├──── LV1 일부
        │
PV2 (/dev/sdd1)
        │
        └──── 다른 LV

PV3 (/dev/sde1)
        │
        └──── LV1 확장 공간
```

즉 기존 `LV1`이 `/dev/sdc1`의 공간을 사용하고 있다가  
새로운 `/dev/sde1`의 공간을 추가로 할당받을 수 있다.

이것이 LVM의 유연한 용량 관리 방식 중 하나이다.

---

# 26. `lsblk`에서 LV가 여러 곳에 표시되는 이유

하나의 LV가 여러 PV에 걸쳐 구성되어 있다면:

```bash
lsblk -f
```

결과에서 같은 LV가 둘 이상의 PV 아래에 표시될 수 있다.

예:

```text
sdc1
└─SOLLVM-8G_LV1

sde1
└─SOLLVM-8G_LV1
```

이는 LV가 두 개 존재한다는 의미가 아니다.

하나의 LV가:

```text
/dev/sdc1
```

과

```text
/dev/sde1
```

두 PV의 공간을 사용하고 있다는 의미이다.

---

# 27. `pvs`, `vgs`, `lvs`

LVM 상태를 확인할 때 가장 자주 사용하는 명령이다.

## PV 확인

```bash
pvs
```

## VG 확인

```bash
vgs
```

## LV 확인

```bash
lvs
```

간단하게 기억하면:

```text
pvs → 물리 볼륨
vgs → 볼륨 그룹
lvs → 논리 볼륨
```

---

# 28. `pvdisplay`, `vgdisplay`, `lvdisplay`

`pvs`, `vgs`, `lvs`보다 자세한 정보를 확인한다.

```bash
pvdisplay
vgdisplay
lvdisplay
```

차이:

| 간단 확인 | 상세 확인 |
|---|---|
| `pvs` | `pvdisplay` |
| `vgs` | `vgdisplay` |
| `lvs` | `lvdisplay` |

일상적인 상태 확인에는:

```bash
pvs
vgs
lvs
```

가 편하고,

세부 정보가 필요할 때:

```bash
pvdisplay
vgdisplay
lvdisplay
```

를 사용한다.

---

# 29. `lsblk -f`와 `df -hT`

LVM 상태 확인 명령과 파일시스템 확인 명령의 목적은 다르다.

## `lsblk -f`

```bash
lsblk -f
```

확인 가능:

- 물리 디스크
- 파티션
- LVM 구조
- 파일시스템 종류
- UUID
- 마운트 위치

---

## `df -hT`

```bash
df -hT
```

확인 가능:

- 현재 마운트된 파일시스템
- 파일시스템 타입
- 전체 용량
- 사용량
- 남은 공간
- 마운트 위치

즉:

```text
pvs/vgs/lvs
→ LVM 관점

lsblk
→ Block Device 구조 관점

df
→ 마운트된 파일시스템 관점
```

으로 구분하면 이해하기 쉽다.

---

# 30. LVM 제거 순서

LVM 구성을 제거할 때는 일반적으로 상위 사용 구조부터 역순으로 제거한다.

```text
LV
 ↓
VG
 ↓
PV
```

관련 명령:

```bash
lvremove
vgremove
pvremove
```

예:

```bash
lvremove /dev/VG/LV
vgremove VG
pvremove /dev/sdX1
```

## 주의

이 명령들은 실제 LVM 구성을 제거하는 작업이므로  
데이터가 필요한 환경에서 연습 목적으로 실행하면 안 된다.

삭제 전에는:

```bash
lsblk
pvs
vgs
lvs
findmnt
```

등으로 대상 장치를 반드시 확인해야 한다.

---

# 31. 이번 실습 구조

이번 실습에서는 처음 10GB 디스크 2개를 이용하여 VG를 구성하였다.

```text
/dev/sdc1 약 10GB ─┐
                   ├── SOLLVM 약 20GB
/dev/sdd1 약 10GB ─┘
```

그 안에:

```text
8G_LV1 → /CU
6G_LV2 → /GS
6G_LV3 → /LG
```

를 생성하였다.

초기 VG의 모든 공간을 사용하여:

```text
VFree = 0
```

상태가 되었다.

이후 10GB 디스크를 새로 추가하였다.

```text
/dev/sde1
```

PV 생성:

```bash
pvcreate /dev/sde1
```

VG 확장:

```bash
vgextend SOLLVM /dev/sde1
```

결과:

```text
SOLLVM
약 20GB → 약 30GB
```

그 후:

```bash
lvextend -L +1G /dev/SOLLVM/8G_LV1
```

을 이용하여 LV를:

```text
8GB → 9GB
```

로 확장하였다.

ext4 파일시스템 역시:

```bash
resize2fs /dev/SOLLVM/8G_LV1
```

를 이용하여 확장하였다.

최종 구조:

```text
/dev/sdc1 약 10GB ─┐
                   │
/dev/sdd1 약 10GB ─┼── SOLLVM 약 30GB
                   │
/dev/sde1 약 10GB ─┘
                        │
                        ├── 8G_LV1 → 9GB    → ext4 → /CU
                        ├── 6G_LV2 → 6GB    → ext4 → /GS
                        └── 6G_LV3 → 5.99GB → ext4 → /LG

VG 남은 공간 ≈ 9GB
```

---

# 32. 주요 명령어 정리

| 명령어 | 기능 |
|---|---|
| `pvcreate` | PV 생성 |
| `pvs` | PV 요약 확인 |
| `pvdisplay` | PV 상세 확인 |
| `pvremove` | PV 제거 |
| `vgcreate` | VG 생성 |
| `vgextend` | 기존 VG에 PV 추가 |
| `vgs` | VG 요약 확인 |
| `vgdisplay` | VG 상세 확인 |
| `vgremove` | VG 제거 |
| `lvcreate` | LV 생성 |
| `lvextend` | LV 확장 |
| `lvreduce` | LV 축소 |
| `lvs` | LV 요약 확인 |
| `lvdisplay` | LV 상세 확인 |
| `lvremove` | LV 제거 |
| `mkfs.ext4` | ext4 파일시스템 생성 |
| `resize2fs` | ext 계열 파일시스템 크기 조정 |
| `xfs_growfs` | XFS 파일시스템 확장 |
| `lsblk -f` | 저장장치/LVM/파일시스템 구조 확인 |
| `df -hT` | 파일시스템 사용량 및 타입 확인 |
| `findmnt` | 마운트 정보 확인 |
| `blkid` | UUID 및 파일시스템 정보 확인 |
| `mount` | 파일시스템 마운트 |
| `umount` | 파일시스템 마운트 해제 |

---

# 33. 핵심 암기 흐름

LVM 생성:

```text
Disk
 ↓
Partition
 ↓
PV
 ↓
VG
 ↓
LV
 ↓
Filesystem
 ↓
Mount
```

명령어:

```text
fdisk
 ↓
pvcreate
 ↓
vgcreate
 ↓
lvcreate
 ↓
mkfs
 ↓
mount
```

용량 확장:

```text
새 Disk
 ↓
Partition
 ↓
pvcreate
 ↓
vgextend
 ↓
lvextend
 ↓
Filesystem 확장
```

ext4의 경우:

```text
pvcreate
 ↓
vgextend
 ↓
lvextend
 ↓
resize2fs
```

---

# 34. 핵심 정리

- LVM은 Linux의 논리 볼륨 관리 기능
- PV는 LVM에서 사용할 물리 저장 공간
- VG는 여러 PV를 묶은 저장 공간 Pool
- LV는 VG에서 필요한 만큼 할당한 논리 볼륨
- PE는 LVM에서 공간을 할당하는 기본 단위
- LV 위에 ext4, XFS 등의 파일시스템을 생성
- LV는 일반 파티션처럼 마운트하여 사용
- 새로운 PV를 `vgextend`로 기존 VG에 추가 가능
- VG의 여유 공간을 이용하여 기존 LV 확장 가능
- LV 확장과 파일시스템 확장은 서로 다른 작업
- ext4 확장에는 `resize2fs` 사용 가능
- XFS 확장에는 `xfs_growfs` 사용
- XFS는 일반적인 축소를 지원하지 않음
- ext4 축소는 언마운트 및 파일시스템 검사가 필요하므로 주의
- 하나의 LV가 여러 PV의 공간을 사용할 수 있음
- LVM 자체는 RAID 또는 백업을 대신하지 않음
- `/etc/fstab`을 이용하면 부팅 시 자동 마운트 가능
- LVM 삭제는 일반적으로 LV → VG → PV 순서로 진행
