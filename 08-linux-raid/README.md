# Linux RAID 실습

Rocky Linux 환경에서 `mdadm`을 이용하여 Software RAID를 구성하고  
Linear RAID, RAID0, RAID1, RAID5의 특징과 장애 복구 과정을 실습했습니다.

---

## 1. 실습 목표

- Linux Software RAID 구성 방법 확인
- Linear RAID, RAID0, RAID1, RAID5 구성
- RAID별 용량 및 특징 비교
- 파일시스템 생성 및 마운트
- RAID 디스크 장애 상황 확인
- 장애 디스크 제거 및 재추가
- RAID 복구 후 데이터 유지 확인
- `/etc/mdadm.conf`, `/etc/fstab`을 이용한 영구 설정
- 재부팅 후 RAID 자동 조립 및 자동 마운트 확인

---

## 2. 실습 환경

```text
OS      : Rocky Linux 9
RAID    : Linux Software RAID
Tool    : mdadm
Disk    : VMware Virtual Disk
FS      : ext4
```

RAID 실습에는 별도의 10G 디스크를 사용했습니다.

> `/dev/sdX` 장치명은 재부팅 후 변경될 수 있으므로  
> 실제 작업 전 `lsblk`, `blkid`, `mdadm --detail` 등을 이용하여 장치를 확인했습니다.

---

## 3. RAID 종류

| RAID | 구성 | 사용 가능 용량 | 장애 허용 |
|---|---|---:|---|
| Linear | 디스크 연결 | 전체 용량 합계 | 없음 |
| RAID0 | Striping | 전체 용량 합계 | 없음 |
| RAID1 | Mirroring | 디스크 1개 용량 | 1개 |
| RAID5 | Striping + Parity | `(N-1) × 최소 디스크 용량` | 1개 |

---

# 4. Linear RAID

두 개의 디스크를 하나의 연속된 논리 디스크처럼 사용하는 Linear RAID를 구성했습니다.

```bash
mdadm --create /dev/md0 \
  --level=linear \
  --raid-devices=2 \
  <RAID_MEMBER_1> <RAID_MEMBER_2>
```

RAID 상태 확인:

```bash
cat /proc/mdstat
mdadm --detail /dev/md0
mdadm --detail --scan
```

약 10G 디스크 2개를 연결하여 약 20G 공간으로 구성했습니다.

```text
RAID Level : linear
Array Size : 약 20G
```

ext4 파일시스템을 생성하고 `/linear`에 마운트했습니다.

```bash
mkfs.ext4 /dev/md0
mkdir /linear
mount /dev/md0 /linear
```

### 확인한 내용

- 여러 디스크의 공간을 순서대로 연결
- 전체 디스크 용량을 사용할 수 있음
- 데이터 중복 저장이나 패리티가 없음
- 디스크 장애에 대한 보호 기능 없음

---

# 5. RAID0

두 개의 10G 디스크를 이용하여 RAID0을 구성했습니다.

```bash
mdadm --create /dev/md0 \
  --level=0 \
  --raid-devices=2 \
  /dev/sdc1 /dev/sdd1
```

구성 결과:

```text
Raid Level : raid0
Array Size : 20951040 (19.98 GiB 21.45 GB)
Raid Devices : 2
Active Devices : 2
Working Devices : 2
Failed Devices : 0
Chunk Size : 512K
```

약 10G 디스크 2개를 RAID0으로 구성하여 약 20G를 사용할 수 있었습니다.

ext4 생성 및 마운트:

```bash
mkfs.ext4 /dev/md0
mkdir /raid0
mount /dev/md0 /raid0
```

마운트 결과:

```text
TARGET SOURCE   FSTYPE OPTIONS
/raid0 /dev/md0 ext4   rw,relatime,seclabel,stripe=256
```

```text
Filesystem Type Size Used Avail Use% Mounted on
/dev/md0   ext4  20G  24K   19G   1% /raid0
```

### 재부팅 후 확인

재부팅 후 RAID 멤버의 `/dev/sdX` 장치명이 변경되었지만 RAID 메타데이터를 이용하여 정상적으로 RAID0이 조립되었습니다.

```text
md0 : active raid0 ...
      20951040 blocks super 1.2 512k chunks
```

### 확인한 내용

- 데이터를 여러 디스크에 나누어 저장하는 Striping 방식
- 디스크 전체 용량 사용 가능
- RAID 자체적인 장애 복구 기능 없음
- RAID 구성 디스크 하나가 손상되면 전체 RAID에 문제가 발생할 수 있음

---

# 6. RAID1

두 개의 10G 디스크를 이용하여 RAID1을 구성했습니다.

```bash
mdadm --create /dev/md0 \
  --level=1 \
  --raid-devices=2 \
  /dev/sdd1 /dev/sde1
```

초기 동기화 완료 후 상태:

```text
md0 : active raid1 sde1[1] sdd1[0]
      10475520 blocks super 1.2 [2/2] [UU]
```

상세 상태:

```text
Raid Level : raid1
Array Size : 10475520 (9.99 GiB 10.73 GB)
Raid Devices : 2
Active Devices : 2
Working Devices : 2
Failed Devices : 0
State : clean
```

RAID1은 약 10G 디스크 두 개를 사용했지만 동일한 데이터를 복제하기 때문에 사용 가능한 용량은 약 10G였습니다.

---

## RAID1 장애 테스트

`/dev/sde1`을 장애 상태로 변경했습니다.

```bash
mdadm /dev/md0 --fail /dev/sde1
```

장애 발생 후:

```text
md0 : active raid1 sde1[1](F) sdd1[0]
      10475520 blocks super 1.2 [2/1] [U_]
```

```text
State : clean, degraded
Active Devices : 1
Working Devices : 1
Failed Devices : 1
```

RAID가 degraded 상태에서도 기존 데이터가 정상적으로 읽히는 것을 확인했습니다.

장애 디스크 제거:

```bash
mdadm /dev/md0 --remove /dev/sde1
```

다시 RAID에 추가:

```bash
mdadm /dev/md0 --add /dev/sde1
```

복구 완료 후:

```text
[2/2] [UU]

State : clean
Active Devices : 2
Working Devices : 2
Failed Devices : 0
```

### 확인한 내용

- RAID1은 동일한 데이터를 두 디스크에 복제
- 디스크 1개 장애 발생 시에도 서비스 가능
- 장애 디스크 제거 후 새로운 디스크를 추가하여 복구 가능
- 사용 가능한 용량은 디스크 한 개 크기

---

# 7. RAID5

10G 디스크 3개를 이용하여 RAID5를 구성했습니다.

```bash
mdadm --create /dev/md0 \
  --level=5 \
  --raid-devices=3 \
  /dev/sdc1 /dev/sdd1 /dev/sde1
```

초기 RAID 생성 과정에서는 recovery가 진행되었습니다.

```text
State : clean, degraded, recovering
Active Devices : 2
Working Devices : 3
Failed Devices : 0
Spare Devices : 1
```

복구 완료 후:

```text
md0 : active raid5 sde1[3] sdd1[1] sdc1[0]
      20951040 blocks super 1.2 level 5, 512k chunk,
      algorithm 2 [3/3] [UUU]
```

상세 상태:

```text
Raid Level : raid5
Array Size : 20951040 (19.98 GiB 21.45 GB)
Used Dev Size : 10475520 (9.99 GiB 10.73 GB)
Raid Devices : 3
Active Devices : 3
Working Devices : 3
Failed Devices : 0
State : clean
Chunk Size : 512K
```

10G 디스크 3개를 사용하여 약 20G의 사용 가능한 공간을 확보했습니다.

```text
RAID5 용량

(N - 1) × 최소 디스크 용량

(3 - 1) × 10G
= 약 20G
```

---

## RAID5 파일시스템 및 마운트

```bash
mkfs.ext4 /dev/md0

mkdir -p /raid5
mount /dev/md0 /raid5
```

확인:

```text
TARGET SOURCE   FSTYPE OPTIONS
/raid5 /dev/md0 ext4   rw,relatime,seclabel,stripe=256
```

```text
Filesystem Type Size Used Avail Use% Mounted on
/dev/md0   ext4  20G  28K   19G   1% /raid5
```

테스트 파일 생성:

```bash
echo "RAID5 정상 동작 테스트" > /raid5/test.txt
cat /raid5/test.txt
```

결과:

```text
RAID5 정상 동작 테스트
```

---

# 8. RAID5 장애 테스트

`/dev/sde1`을 장애 상태로 변경했습니다.

```bash
mdadm /dev/md0 --fail /dev/sde1
```

장애 발생 후:

```text
md0 : active raid5 sde1[3](F) sdd1[1] sdc1[0]
      20951040 blocks super 1.2 level 5,
      512k chunk, algorithm 2 [3/2] [UU_]
```

상세 상태:

```text
State : clean, degraded
Active Devices : 2
Working Devices : 2
Failed Devices : 1
```

RAID5가 degraded 상태임에도 기존 파일을 읽을 수 있었습니다.

```bash
cat /raid5/test.txt
```

```text
RAID5 정상 동작 테스트
```

---

## degraded 상태 쓰기 테스트

장애가 발생한 상태에서 새로운 파일을 생성했습니다.

```bash
echo "RAID5 degraded 상태에서도 저장 성공" \
  > /raid5/degraded-test.txt
```

확인:

```text
RAID5 degraded 상태에서도 저장 성공
```

파일 목록:

```text
-rw-r--r--. 1 root root 45 degraded-test.txt
-rw-r--r--. 1 root root 30 test.txt
```

따라서 RAID5는 디스크 1개가 장애 난 상태에서도 데이터 읽기와 쓰기가 가능한 것을 확인했습니다.

---

# 9. RAID5 장애 복구

장애 디스크를 RAID에서 제거했습니다.

```bash
mdadm /dev/md0 --remove /dev/sde1
```

다시 RAID 멤버로 추가했습니다.

```bash
mdadm /dev/md0 --add /dev/sde1
```

복구 완료 후:

```text
[3/3] [UUU]
```

상세 상태:

```text
State : clean
Active Devices : 3
Working Devices : 3
Failed Devices : 0
Spare Devices : 0
```

복구 후 데이터 확인:

```bash
cat /raid5/test.txt
cat /raid5/degraded-test.txt
```

결과:

```text
RAID5 정상 동작 테스트
RAID5 degraded 상태에서도 저장 성공
```

장애 디스크 복구 후에도 기존 데이터와 degraded 상태에서 생성한 데이터가 정상적으로 유지되었습니다.

---

# 10. RAID 영구 설정

RAID 정보를 `/etc/mdadm.conf`에 등록했습니다.

```bash
mdadm --detail --scan > /etc/mdadm.conf
```

설정 내용:

```text
ARRAY /dev/md0 metadata=1.2 UUID=980bd44b:85927ed8:da33bb9b:fe75266b
```

RAID5 ext4 파일시스템 UUID:

```text
49fe1783-e3f1-418a-b63a-ace33b94938e
```

`/etc/fstab`에 다음 항목을 추가했습니다.

```text
UUID=49fe1783-e3f1-418a-b63a-ace33b94938e  /raid5  ext4  defaults  0 2
```

재부팅 전 설정 확인:

```bash
systemctl daemon-reload

umount /raid5
mount -a

findmnt /raid5
```

결과:

```text
TARGET SOURCE   FSTYPE OPTIONS
/raid5 /dev/md0 ext4   rw,relatime,seclabel,stripe=256
```

initramfs 갱신:

```bash
dracut -f
```

이후 시스템을 재부팅했습니다.

```bash
reboot
```

---

# 11. 재부팅 후 RAID5 확인

재부팅 후 RAID가 자동으로 조립된 것을 확인했습니다.

```bash
cat /proc/mdstat
```

```text
md0 : active raid5 sdc1[0] sde1[3] sdd1[1]
      20951040 blocks super 1.2 level 5,
      512k chunk, algorithm 2 [3/3] [UUU]
```

상세 상태:

```text
State : clean
Active Devices : 3
Working Devices : 3
Failed Devices : 0
```

`/raid5`도 자동으로 마운트되었습니다.

```text
TARGET SOURCE   FSTYPE OPTIONS
/raid5 /dev/md0 ext4   rw,relatime,seclabel,stripe=256
```

```text
Filesystem Type Size Used Avail Use% Mounted on
/dev/md0   ext4  20G  32K   19G   1% /raid5
```

재부팅 후에도 테스트 데이터가 정상적으로 유지되었습니다.

```text
RAID5 정상 동작 테스트
RAID5 degraded 상태에서도 저장 성공
```

---

# 12. 주요 명령어 정리

```bash
# RAID 생성
mdadm --create /dev/md0 --level=<LEVEL> \
  --raid-devices=<COUNT> <DEVICE...>

# RAID 상태 확인
cat /proc/mdstat

# RAID 상세 확인
mdadm --detail /dev/md0

# RAID 정보 확인
mdadm --detail --scan

# 디스크 장애 처리
mdadm /dev/md0 --fail <DEVICE>

# 장애 디스크 제거
mdadm /dev/md0 --remove <DEVICE>

# 디스크 추가
mdadm /dev/md0 --add <DEVICE>

# RAID 중지
mdadm --stop /dev/md0

# RAID 메타데이터 제거
mdadm --zero-superblock <DEVICE>
```

---

# 13. 실습 결과

이번 실습을 통해 Linux Software RAID의 생성부터 장애 및 복구까지 직접 확인했습니다.

특히 RAID1과 RAID5에서 실제 디스크 장애를 발생시켜 degraded 상태를 확인하고, 장애 상태에서도 데이터 접근이 가능한지 검증했습니다.

RAID5에서는 다음 과정을 직접 확인했습니다.

```text
정상 RAID5 [UUU]
        ↓
디스크 1개 장애
        ↓
degraded [UU_]
        ↓
기존 데이터 읽기 성공
        ↓
새로운 데이터 쓰기 성공
        ↓
장애 디스크 제거 및 재추가
        ↓
RAID 복구
        ↓
정상 RAID5 [UUU]
        ↓
재부팅
        ↓
RAID 자동 조립 및 자동 마운트 확인
```

또한 재부팅 후 `/dev/sdX` 장치명이 변경될 수 있다는 점을 확인했고, RAID 메타데이터와 UUID 기반 설정을 이용하여 안정적으로 RAID를 구성할 수 있음을 확인했습니다.

> RAID는 디스크 장애에 대한 가용성을 높이기 위한 기술이며 백업을 대체하지 않습니다.
