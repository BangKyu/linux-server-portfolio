# Linux RAID 정리

## 1. RAID란?

RAID(Redundant Array of Independent Disks)는 여러 개의 물리 디스크를 하나의 논리적인 저장장치처럼 구성하는 기술이다.

RAID를 사용하는 목적은 구성 방식에 따라 다르다.

- 여러 디스크의 용량을 하나로 사용
- 디스크 I/O 성능 향상
- 디스크 장애에 대한 가용성 확보
- 데이터 중복 저장 또는 Parity를 이용한 복구

RAID Level에 따라 성능, 사용 가능한 용량, 장애 허용 능력이 달라진다.

> RAID는 디스크 장애에 대비하기 위한 기술이며 백업을 대체하지 않는다.

예를 들어 RAID1이나 RAID5를 사용하더라도 사용자가 파일을 삭제하거나 데이터가 손상되면 RAID가 삭제된 데이터를 복구해 주는 것은 아니다.

---

# 2. RAID 종류

RAID는 크게 Hardware RAID와 Software RAID로 구성할 수 있다.

## Hardware RAID

별도의 RAID Controller가 RAID 처리를 담당하는 방식이다.

```text
Disk ─┐
Disk ─┼─ RAID Controller ─ OS
Disk ─┘
```

특징:

- RAID Controller가 RAID 연산 담당
- 운영체제와 독립적으로 RAID 구성 가능
- RAID 기능과 성능은 Controller에 따라 달라짐
- Hot Swap 등의 기능은 RAID Controller와 하드웨어가 지원해야 사용 가능

---

## Software RAID

운영체제가 RAID 처리를 담당하는 방식이다.

Linux에서는 대표적으로 `mdadm`을 사용한다.

```text
Disk ─┐
Disk ─┼─ Linux mdadm ─ /dev/md0
Disk ─┘
```

특징:

- 별도의 RAID Controller 없이 구성 가능
- Linux에서 `mdadm`을 이용하여 관리
- RAID 배열은 `/dev/md0`, `/dev/md1` 등의 장치로 생성 가능
- 일반적인 서버 및 실습 환경에서도 쉽게 구성 가능

이번 실습에서는 Linux Software RAID를 사용했다.

---

# 3. mdadm

`mdadm`은 Linux Software RAID를 생성하고 관리하는 명령어이다.

설치:

```bash
dnf install -y mdadm
```

RAID 생성:

```bash
mdadm --create /dev/md0 \
  --level=<RAID_LEVEL> \
  --raid-devices=<DISK_COUNT> \
  <DEVICE1> <DEVICE2>
```

예:

```bash
mdadm --create /dev/md0 \
  --level=5 \
  --raid-devices=3 \
  /dev/sdc1 /dev/sdd1 /dev/sde1
```

RAID 상태 확인:

```bash
cat /proc/mdstat
```

상세 정보 확인:

```bash
mdadm --detail /dev/md0
```

RAID 배열 정보 확인:

```bash
mdadm --detail --scan
```

---

# 4. Linear RAID

Linear RAID는 여러 디스크의 공간을 순서대로 연결하여 하나의 큰 논리 디스크처럼 사용하는 방식이다.

예:

```text
Disk A 10G
+
Disk B 10G

        ↓

Linear RAID 약 20G
```

데이터 저장 형태를 단순화하면 다음과 같다.

```text
Disk A
AAAA AAAA AAAA

Disk B
BBBB BBBB BBBB
```

먼저 Disk A의 공간을 사용하고 이후 Disk B의 공간을 사용하는 방식이다.

## 특징

- 최소 디스크: 2개
- 사용 가능 용량: 모든 디스크 용량의 합
- Striping 없음
- Mirroring 없음
- Parity 없음
- 장애 복구 기능 없음

용량:

```text
10G + 10G
= 약 20G
```

Linear RAID는 여러 디스크의 공간을 하나로 합칠 수 있지만 장애 허용 기능은 제공하지 않는다.

---

# 5. RAID0

RAID0은 데이터를 여러 디스크에 나누어 저장하는 Striping 방식이다.

```text
Data
A B C D E F

       ↓

Disk 1       Disk 2
A             B
C             D
E             F
```

두 디스크에 데이터를 나누어 동시에 읽고 쓸 수 있다.

## 특징

- 최소 디스크: 2개
- 방식: Striping
- 사용 가능 용량: 모든 디스크 용량 합계
- 성능 향상 가능
- 장애 허용 기능 없음

예:

```text
10G + 10G
= 약 20G 사용
```

용량 공식:

```text
디스크 개수 × 가장 작은 디스크 용량
```

동일한 크기의 디스크를 사용하는 것이 일반적이다.

## 단점

RAID0에는 데이터 중복이나 Parity가 없다.

따라서 RAID 구성 디스크 중 하나가 손상되면 전체 데이터에 영향을 줄 수 있다.

```text
Disk 1 정상
Disk 2 장애

→ RAID0 전체 데이터 사용에 문제 발생
```

RAID0은 성능과 용량 효율은 좋지만 장애에 대한 보호 기능은 없다.

---

# 6. RAID1

RAID1은 동일한 데이터를 여러 디스크에 복제하는 Mirroring 방식이다.

```text
데이터 A

      ↓

Disk 1       Disk 2
A             A
B             B
C             C
```

## 특징

- 최소 디스크: 2개
- 방식: Mirroring
- 동일한 데이터를 각 디스크에 저장
- 디스크 1개 장애 시에도 서비스 지속 가능
- 사용 가능한 용량은 일반적인 2-disk RAID1에서 디스크 1개 크기

예:

```text
Disk 1 = 10G
Disk 2 = 10G

실제 사용 가능 용량 ≈ 10G
```

## 장애 발생

정상 상태:

```text
[UU]
```

의미:

```text
Disk 1 : U → 정상
Disk 2 : U → 정상
```

한 개의 디스크에 장애가 발생하면:

```text
[U_]
```

이 상태를 `degraded` 상태라고 한다.

```text
Disk 1 : U → 정상
Disk 2 : _ → 사용 불가
```

RAID1은 나머지 정상 디스크에 동일한 데이터가 존재하므로 계속 데이터에 접근할 수 있다.

---

# 7. RAID5

RAID5는 Striping과 Parity를 함께 사용하는 방식이다.

최소 3개의 디스크가 필요하다.

```text
Disk 1      Disk 2      Disk 3

Data A      Data B      Parity
Data C      Parity      Data D
Parity      Data E      Data F
```

Parity는 특정 디스크 하나에만 고정되는 것이 아니라 여러 디스크에 분산되어 저장된다.

## 특징

- 최소 디스크: 3개
- 방식: Striping + Distributed Parity
- 디스크 1개 장애 허용
- RAID0보다 장애 대응 능력이 있음
- RAID1보다 일반적으로 용량 효율이 높음

사용 가능 용량:

```text
(N - 1) × 가장 작은 디스크 용량
```

N은 RAID를 구성하는 디스크 개수이다.

예:

```text
10G × 3개

(3 - 1) × 10G
= 약 20G
```

실습 결과:

```text
Array Size : 19.98 GiB
```

---

# 8. RAID5 장애 원리

정상적인 3-disk RAID5 상태:

```text
[UUU]
```

디스크 하나가 장애 상태가 되면:

```text
[UU_]
```

RAID 상태는 다음과 같이 표시될 수 있다.

```text
State : clean, degraded
```

하지만 RAID5는 Data와 Parity 정보를 이용하여 하나의 디스크가 없는 상태에서도 데이터를 계산할 수 있다.

따라서 디스크 1개 장애 상태에서도 읽기와 쓰기가 가능하다.

이번 실습에서도 다음을 직접 확인했다.

```text
정상 RAID5 [UUU]
        ↓
디스크 1개 장애
        ↓
degraded [UU_]
        ↓
기존 파일 읽기 성공
        ↓
새 파일 쓰기 성공
```

단, 3-disk RAID5에서 이미 디스크 하나가 장애 난 상태에서 추가 디스크까지 장애가 발생하면 RAID5의 장애 허용 범위를 초과한다.

---

# 9. RAID6

RAID6는 RAID5와 비슷하지만 두 종류의 Parity 정보를 사용한다.

```text
Striping + Double Parity
```

## 특징

- 최소 디스크: 4개
- 디스크 2개 장애 허용
- RAID5보다 장애 대응 능력이 높음
- Parity 계산과 저장 공간이 더 필요함

사용 가능 용량:

```text
(N - 2) × 가장 작은 디스크 용량
```

예:

```text
10G × 4개

(4 - 2) × 10G
= 약 20G
```

---

# 10. RAID10

RAID10은 RAID1과 RAID0을 조합한 방식이다.

```text
RAID1 + RAID0
```

일반적으로 최소 4개의 디스크가 필요하다.

구조 예:

```text
        RAID10

Disk1 ─┐
       ├─ Mirror ─┐
Disk2 ─┘          │
                  ├─ Striping
Disk3 ─┐          │
       ├─ Mirror ─┘
Disk4 ─┘
```

## 특징

- Mirroring + Striping
- 성능과 장애 대응 능력을 함께 고려할 수 있음
- 일반적인 구성에서 사용 가능 용량은 전체 용량의 약 50%

예:

```text
10G × 4개 = 40G

사용 가능 용량 ≈ 20G
```

실제 장애 허용 범위는 어떤 디스크 조합이 고장 나는지와 RAID10 layout에 따라 달라질 수 있다.

---

# 11. RAID 비교

| RAID | 최소 디스크 | 저장 방식 | 용량 효율 | 장애 허용 |
|---|---:|---|---|---|
| Linear | 2 | 순차 연결 | 높음 | 없음 |
| RAID0 | 2 | Striping | 높음 | 없음 |
| RAID1 | 2 | Mirroring | 낮음 | 일반적인 2-disk 구성에서 1개 |
| RAID5 | 3 | Striping + Parity | 비교적 높음 | 1개 |
| RAID6 | 4 | Striping + Double Parity | RAID5보다 낮음 | 2개 |
| RAID10 | 4 | Mirroring + Striping | 약 50% | 구성에 따라 다름 |

---

# 12. RAID 상태 확인

Linux Software RAID 상태를 빠르게 확인할 때:

```bash
cat /proc/mdstat
```

정상적인 RAID1:

```text
[2/2] [UU]
```

정상적인 RAID5:

```text
[3/3] [UUU]
```

RAID5 디스크 하나 장애:

```text
[3/2] [UU_]
```

각 문자의 의미:

```text
U = Up, 정상 동작 중
_ = 해당 RAID 위치의 정상 멤버가 없음
```

---

# 13. clean과 degraded

## clean

RAID 멤버가 정상적으로 구성되어 있는 상태이다.

```text
State : clean
```

예:

```text
Active Devices  : 3
Working Devices : 3
Failed Devices  : 0
```

---

## degraded

RAID 구성원 중 일부가 빠진 상태이다.

```text
State : clean, degraded
```

예:

```text
Active Devices  : 2
Working Devices : 2
Failed Devices  : 1
```

`degraded`라고 해서 RAID가 즉시 사용 불가능한 것은 아니다.

RAID Level의 장애 허용 범위 안이라면 서비스를 계속 제공할 수 있다.

하지만 정상 상태보다 추가 장애에 취약하므로 가능한 빨리 복구해야 한다.

---

# 14. RAID 장애 처리 명령어

특정 멤버를 장애 상태로 표시:

```bash
mdadm /dev/md0 --fail /dev/sde1
```

RAID 상태 확인:

```bash
cat /proc/mdstat
mdadm --detail /dev/md0
```

장애 디스크 제거:

```bash
mdadm /dev/md0 --remove /dev/sde1
```

디스크 다시 추가:

```bash
mdadm /dev/md0 --add /dev/sde1
```

추가한 후 RAID가 필요한 데이터를 다시 동기화한다.

---

# 15. Recovery와 Rebuild

RAID에 새로운 디스크를 추가하거나 장애 디스크를 다시 추가하면 기존 정상 디스크의 데이터를 이용하여 RAID를 복구한다.

이 과정을 Recovery 또는 Rebuild라고 한다.

진행 상태 확인:

```bash
cat /proc/mdstat
```

예:

```text
recovery = 40.1%
```

복구 완료:

```text
[UUU]
```

그리고:

```text
State : clean
```

으로 돌아온다.

---

# 16. Write-intent Bitmap

RAID1과 RAID5를 생성하면서 Internal Bitmap을 사용했다.

확인:

```text
Intent Bitmap : Internal
```

또는:

```text
bitmap: 0/1 pages
```

Write-intent Bitmap은 RAID가 정상 상태가 아닐 때 변경된 영역을 추적하는 데 사용할 수 있다.

이를 통해 RAID 멤버가 다시 추가되었을 때 전체 디스크를 무조건 다시 동기화하는 대신 필요한 변경 영역을 중심으로 복구할 수 있다.

이번 실습에서는 데이터 변경량이 매우 적었기 때문에 장애 디스크를 다시 추가했을 때 복구가 매우 빠르게 끝났다.

---

# 17. Chunk Size

RAID0이나 RAID5에서 데이터는 일정 크기의 단위로 나누어 여러 디스크에 저장된다.

이 단위를 Chunk라고 한다.

실습 RAID5:

```text
Chunk Size : 512K
```

즉 RAID가 데이터를 분배할 때 기본 Chunk 크기가 512K로 설정되었다.

Chunk Size는 RAID의 I/O 특성에 영향을 줄 수 있다.

---

# 18. ext4의 stripe 값

RAID5를 ext4로 마운트했을 때 다음과 같은 결과를 확인했다.

```text
/raid5 /dev/md0 ext4 rw,relatime,seclabel,stripe=256
```

실습 환경:

```text
RAID Chunk Size = 512K
ext4 Block Size = 4K
RAID5 Data Disk = 2개
```

계산:

```text
512K ÷ 4K
= 128 blocks

128 × 2 Data Disks
= 256 blocks
```

따라서 ext4에서:

```text
stripe=256
```

으로 확인되었다.

---

# 19. RAID Superblock

Software RAID는 RAID 구성 정보를 멤버 디스크에 metadata로 저장한다.

실습에서는:

```text
Version : 1.2
```

를 사용했다.

RAID 멤버를 확인하면:

```bash
lsblk -f
```

다음과 같이 표시될 수 있다.

```text
linux_raid_member
```

또한 RAID metadata를 자세히 확인하려면:

```bash
mdadm --examine /dev/sdc1
```

을 사용할 수 있다.

---

# 20. RAID UUID

RAID 배열에는 고유한 UUID가 있다.

확인:

```bash
mdadm --detail /dev/md0
```

또는:

```bash
mdadm --detail --scan
```

실습 RAID5:

```text
UUID : 980bd44b:85927ed8:da33bb9b:fe75266b
```

`mdadm.conf`에 RAID UUID 정보를 저장하면 RAID 배열을 식별하고 조립하는 데 사용할 수 있다.

---

# 21. 파일시스템 UUID와 RAID UUID 차이

RAID에는 두 종류의 UUID를 구분해서 이해하는 것이 중요하다.

## RAID UUID

RAID 배열 자체를 식별한다.

```text
980bd44b:85927ed8:da33bb9b:fe75266b
```

사용 위치:

```text
/etc/mdadm.conf
```

예:

```text
ARRAY /dev/md0 metadata=1.2 UUID=980bd44b:85927ed8:da33bb9b:fe75266b
```

---

## 파일시스템 UUID

RAID 위에 생성한 ext4 파일시스템을 식별한다.

```text
49fe1783-e3f1-418a-b63a-ace33b94938e
```

확인:

```bash
blkid /dev/md0
```

사용 위치:

```text
/etc/fstab
```

예:

```text
UUID=49fe1783-e3f1-418a-b63a-ace33b94938e  /raid5  ext4  defaults  0 2
```

즉:

```text
RAID UUID
→ RAID 배열 식별

Filesystem UUID
→ RAID 위의 파일시스템 식별
```

이다.

---

# 22. /etc/mdadm.conf

`/etc/mdadm.conf`는 Software RAID 배열 정보를 저장하는 데 사용할 수 있다.

현재 RAID 정보를 생성:

```bash
mdadm --detail --scan
```

파일에 저장:

```bash
mdadm --detail --scan > /etc/mdadm.conf
```

실습 RAID5:

```text
ARRAY /dev/md0 metadata=1.2 UUID=980bd44b:85927ed8:da33bb9b:fe75266b
```

---

# 23. /etc/fstab

RAID를 재부팅 후 자동으로 마운트하기 위해 RAID 위에 생성된 파일시스템 UUID를 `/etc/fstab`에 등록했다.

```text
UUID=49fe1783-e3f1-418a-b63a-ace33b94938e  /raid5  ext4  defaults  0 2
```

`fstab` 수정 후:

```bash
systemctl daemon-reload
```

그리고 재부팅 전에:

```bash
umount /raid5
mount -a
```

를 이용하여 설정에 오류가 없는지 확인했다.

`mount -a`가 정상적으로 수행되는지 먼저 확인하면 잘못된 `fstab` 설정을 재부팅 전에 발견할 수 있다.

---

# 24. dracut

RAID 영구 설정 후 다음 명령을 사용했다.

```bash
dracut -f
```

`dracut`은 initramfs를 생성하거나 갱신하는 도구이다.

필요한 RAID 관련 설정 및 모듈이 부팅 초기 단계에서 사용될 수 있도록 initramfs를 갱신할 때 사용할 수 있다.

그 후:

```bash
reboot
```

하여 RAID 자동 조립과 자동 마운트를 확인했다.

---

# 25. /dev/sdX 장치명 주의

Linux에서 다음과 같은 디스크 이름은:

```text
/dev/sdc
/dev/sdd
/dev/sde
```

항상 같은 물리 디스크를 의미한다고 가정하면 안 된다.

실습에서도 재부팅 후 RAID 멤버의 `/dev/sdX` 이름이 변경된 경우가 있었다.

따라서 실제 서버에서 디스크 작업을 할 때는 명령을 실행하기 전에 반드시 확인해야 한다.

```bash
lsblk -f
blkid
fdisk -l
mdadm --detail /dev/md0
mdadm --examine <DEVICE>
```

특히 다음과 같은 명령은 데이터를 삭제하거나 변경할 수 있으므로 장치 확인이 중요하다.

```bash
fdisk
mkfs
wipefs
mdadm --zero-superblock
```

---

# 26. RAID 삭제 및 초기화

RAID 실습을 정리할 때 먼저 마운트를 해제한다.

```bash
umount /raid5
```

RAID 중지:

```bash
mdadm --stop /dev/md0
```

RAID metadata 제거:

```bash
mdadm --zero-superblock /dev/sdc1
mdadm --zero-superblock /dev/sdd1
mdadm --zero-superblock /dev/sde1
```

확인:

```bash
mdadm --examine /dev/sdc1
```

metadata가 정상적으로 제거되었다면 다음과 같은 메시지를 볼 수 있다.

```text
No md superblock detected
```

> `--zero-superblock`, `mkfs`, `fdisk` 등은 기존 데이터를 손상시킬 수 있으므로 대상 디스크를 반드시 확인한 후 실행해야 한다.

---

# 27. RAID와 Backup의 차이

RAID와 Backup은 목적이 다르다.

## RAID

```text
목적
→ 디스크 장애에 대한 가용성 확보
→ RAID Level에 따라 성능 향상
```

## Backup

```text
목적
→ 삭제된 파일 복구
→ 과거 데이터 복원
→ 시스템 장애 및 데이터 손상 대비
```

예를 들어 RAID1에서:

```text
rm important.txt
```

를 실행하면 파일 삭제 내용도 두 디스크에 동일하게 반영될 수 있다.

따라서 RAID1이라고 해서 삭제된 파일을 복구할 수 있는 것은 아니다.

```text
RAID ≠ Backup
```

RAID와 별도로 Backup 정책이 필요하다.

---

# 28. RAID Level 선택 기준

RAID Level은 무조건 하나가 가장 좋은 것이 아니라 목적에 따라 선택해야 한다.

## 용량과 성능 중심

```text
RAID0
```

- 전체 용량 활용
- Striping
- 장애 허용 없음

---

## 단순한 장애 대응 중심

```text
RAID1
```

- Mirroring
- 구조 이해와 관리가 비교적 단순
- 용량 효율이 낮음

---

## 용량 효율 + 1개 디스크 장애 대응

```text
RAID5
```

- Striping + Parity
- 디스크 1개 장애 허용
- 최소 3개 디스크 필요

---

## 2개 디스크 장애 대응

```text
RAID6
```

- Double Parity
- 디스크 2개 장애 허용
- 최소 4개 디스크 필요

---

## 성능 + Mirroring

```text
RAID10
```

- RAID1 + RAID0
- 일반적으로 최소 4개 디스크
- 용량의 상당 부분을 Mirroring에 사용

---

# 29. 실습에서 확인한 핵심 흐름

## RAID 생성

```text
물리 디스크
    ↓
파티션 생성
    ↓
Linux RAID 파티션 준비
    ↓
mdadm --create
    ↓
/dev/md0 생성
```

## 파일시스템 사용

```text
/dev/md0
   ↓
mkfs.ext4
   ↓
mount
   ↓
파일 저장
```

## 장애 및 복구

```text
정상 RAID
   ↓
--fail
   ↓
degraded
   ↓
데이터 접근 확인
   ↓
--remove
   ↓
--add
   ↓
recovery / rebuild
   ↓
정상 RAID
```

## 영구 설정

```text
RAID UUID
   ↓
/etc/mdadm.conf

Filesystem UUID
   ↓
/etc/fstab

   ↓
mount -a
   ↓
dracut -f
   ↓
reboot
   ↓
자동 조립 / 자동 마운트 확인
```

---

# 30. 주요 명령어 정리

## 디스크 확인

```bash
lsblk
lsblk -f
blkid
fdisk -l
```

## RAID 생성

```bash
mdadm --create /dev/md0 \
  --level=<LEVEL> \
  --raid-devices=<COUNT> \
  <DEVICE...>
```

## RAID 상태

```bash
cat /proc/mdstat
mdadm --detail /dev/md0
mdadm --detail --scan
```

## RAID metadata 확인

```bash
mdadm --examine <DEVICE>
```

## 장애 처리

```bash
mdadm /dev/md0 --fail <DEVICE>
```

## 디스크 제거

```bash
mdadm /dev/md0 --remove <DEVICE>
```

## 디스크 추가

```bash
mdadm /dev/md0 --add <DEVICE>
```

## RAID 중지

```bash
mdadm --stop /dev/md0
```

## RAID metadata 제거

```bash
mdadm --zero-superblock <DEVICE>
```

## 파일시스템 생성

```bash
mkfs.ext4 /dev/md0
```

## 마운트

```bash
mount /dev/md0 /raid5
findmnt /raid5
df -hT /raid5
```

## 영구 설정

```bash
mdadm --detail --scan
blkid /dev/md0

systemctl daemon-reload
mount -a

dracut -f
```

---

# 31. 정리

RAID는 여러 디스크를 하나의 논리 저장장치로 구성하는 기술이다.

RAID Level에 따라 목적이 다르다.

```text
Linear
→ 디스크 공간 연결

RAID0
→ Striping
→ 성능 및 전체 용량 활용
→ 장애 허용 없음

RAID1
→ Mirroring
→ 동일 데이터 복제
→ 일반적인 2-disk 구성에서 1개 디스크 장애 허용

RAID5
→ Striping + Distributed Parity
→ 최소 3개 디스크
→ 1개 디스크 장애 허용

RAID6
→ Double Parity
→ 최소 4개 디스크
→ 2개 디스크 장애 허용

RAID10
→ Mirroring + Striping
→ 성능과 장애 대응을 함께 고려
```

Linux Software RAID에서는 `mdadm`을 사용하여 RAID를 생성하고 관리할 수 있다.

RAID 관리 시 중요한 점:

```text
1. 작업 전 실제 디스크 확인
2. /dev/sdX 이름만 믿지 않기
3. RAID 상태는 /proc/mdstat와 mdadm --detail로 확인
4. degraded 상태에서는 추가 장애에 주의
5. 장애 디스크 제거 후 새 디스크 추가 및 복구
6. RAID UUID와 Filesystem UUID 구분
7. mdadm.conf와 fstab을 이용한 영구 설정
8. 재부팅 후 실제 자동 조립 및 마운트 확인
9. RAID를 Backup으로 생각하지 않기
```
