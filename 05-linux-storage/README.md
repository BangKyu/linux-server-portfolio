# Linux 디스크 파티션 및 마운트 실습

Rocky Linux 환경에서 추가 디스크를 파티셔닝하고 파일시스템을 생성한 뒤,
수동 마운트와 `/etc/fstab`을 이용한 자동 마운트를 구성하고 검증하였다.

---

## 1. 실습 환경 구성

- 시스템 디스크: `/dev/sda` 100G
- 실습용 추가 디스크: `/dev/sdb` 100G

추가 디스크 확인:

```bash
lsblk
```

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  100G  0 disk
├─sda1   8:1    0    4G  0 part [SWAP]
└─sda2   8:2    0   96G  0 part /
sdb      8:16   0  100G  0 disk
sr0     11:0    1 14.2G  0 rom
```

`/dev/sda`는 운영체제가 설치된 시스템 디스크이므로 변경하지 않고,
새로 추가한 `/dev/sdb`를 실습에 사용

---

## 2. 파티션 구성

`fdisk`를 이용하여 `/dev/sdb`에 DOS(MBR) 파티션 테이블을 구성

```bash
fdisk /dev/sdb
```

구성 결과:

```text
Device     Boot     Start       End   Sectors Size Id Type
/dev/sdb1            2048  62916607  62914560  30G 83 Linux
/dev/sdb2        62916608 209715199 146798592  70G  5 Extended
/dev/sdb5        62918656 104861695  41943040  20G 83 Linux
/dev/sdb6       104863744 146806783  41943040  20G 83 Linux
/dev/sdb7       146808832 188751871  41943040  20G 83 Linux
/dev/sdb8       188753920 209715199  20961280  10G 83 Linux
```

파티션 구성:

| 파티션 | 크기 | 종류 |
|---|---:|---|
| `/dev/sdb1` | 30G | Primary |
| `/dev/sdb2` | 70G | Extended |
| `/dev/sdb5` | 20G | Logical |
| `/dev/sdb6` | 20G | Logical |
| `/dev/sdb7` | 20G | Logical |
| `/dev/sdb8` | 10G | Logical |

---

## 3. 파일시스템 생성

`/dev/sdb1`에는 XFS 파일시스템을 생성하고,
Logical 파티션에는 ext4 파일시스템을 생성

```bash
mkfs.xfs /dev/sdb1

mkfs.ext4 /dev/sdb5
mkfs.ext4 /dev/sdb6
mkfs.ext4 /dev/sdb7
mkfs.ext4 /dev/sdb8
```

확인:

```bash
lsblk -f
```

```text
sdb
├─sdb1 xfs
├─sdb2
├─sdb5 ext4
├─sdb6 ext4
├─sdb7 ext4
└─sdb8 ext4
```

---

## 4. 마운트 포인트 생성

```bash
mkdir -p /GIT
mkdir -p /homeSK

mkdir -p /homeLG/user1
mkdir -p /homeLG/user2
mkdir -p /homeLG/user3
```

---

## 5. 수동 마운트

각 파티션을 지정한 디렉터리에 마운트

```bash
mount /dev/sdb1 /GIT
mount /dev/sdb5 /homeSK
mount /dev/sdb6 /homeLG/user1
mount /dev/sdb7 /homeLG/user2
mount /dev/sdb8 /homeLG/user3
```

마운트 구성:

| 장치 | 파일시스템 | 마운트 포인트 |
|---|---|---|
| `/dev/sdb1` | XFS | `/GIT` |
| `/dev/sdb5` | ext4 | `/homeSK` |
| `/dev/sdb6` | ext4 | `/homeLG/user1` |
| `/dev/sdb7` | ext4 | `/homeLG/user2` |
| `/dev/sdb8` | ext4 | `/homeLG/user3` |

확인:

```bash
df -hT
```

```text
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      xfs    30G  247M   30G   1% /GIT
/dev/sdb5      ext4   20G   24K   19G   1% /homeSK
/dev/sdb6      ext4   20G   24K   19G   1% /homeLG/user1
/dev/sdb7      ext4   20G   24K   19G   1% /homeLG/user2
/dev/sdb8      ext4  9.8G   24K  9.3G   1% /homeLG/user3
```

---

## 6. 수동 마운트 재부팅 테스트

수동 마운트 상태에서 시스템 재부팅

```bash
reboot
```

재부팅 후 확인:

```bash
lsblk -f
df -hT
```

`/dev/sdb1`, `/dev/sdb5~8`의 마운트 포인트가 사라진 것을 확인

이를 통해 `mount` 명령으로 수행한 수동 마운트는
재부팅 후 자동으로 다시 마운트되지 않는 것을 확인

---

## 7. /etc/fstab 자동 마운트 설정

먼저 기존 설정 파일을 백업

```bash
cp -p /etc/fstab /etc/fstab.bak
```

각 파일시스템의 UUID를 확인

```bash
lsblk -f
```

확인된 UUID를 이용하여 `/etc/fstab`에 다음 내용을 추가

```text
UUID=b94248cd-9d51-4e99-a08f-627c647b9268  /GIT          xfs   defaults  0 0
UUID=6ff8a392-405a-4eea-941c-a0d91abe2e1f  /homeSK       ext4  defaults  0 2
UUID=72c12690-7c93-4cf3-9073-f77d8456775e  /homeLG/user1 ext4  defaults  0 2
UUID=d9463d4b-0069-40b6-a77c-77ec70589df5  /homeLG/user2 ext4  defaults  0 2
UUID=262d8af6-961b-4817-945a-865dd9c6f312  /homeLG/user3 ext4  defaults  0 2
```

설정 변경 후 systemd 설정을 다시 읽고,
`mount -a`를 이용하여 `/etc/fstab` 설정을 테스트

```bash
systemctl daemon-reload
mount -a
```

오류 없이 실행되었으며 모든 파일시스템이 정상적으로 마운트 확인

---

## 8. 자동 마운트 최종 검증

`/etc/fstab` 설정 후 시스템을 다시 재부팅

```bash
reboot
```

재부팅 직후 별도의 `mount` 명령 없이 상태를 확인

```bash
lsblk -f
```

```text
sdb
├─sdb1 xfs  → /GIT
├─sdb2
├─sdb5 ext4 → /homeSK
├─sdb6 ext4 → /homeLG/user1
├─sdb7 ext4 → /homeLG/user2
└─sdb8 ext4 → /homeLG/user3
```

`df -hT`에서도 모든 파일시스템이 정상적으로 자동 마운트된 것을 확인

```text
/dev/sdb1  xfs   30G   247M  30G   1% /GIT
/dev/sdb7  ext4  20G    24K  19G   1% /homeLG/user2
/dev/sdb6  ext4  20G    24K  19G   1% /homeLG/user1
/dev/sdb5  ext4  20G    24K  19G   1% /homeSK
/dev/sdb8  ext4  9.8G   24K  9.3G  1% /homeLG/user3
```

---

## 9. 실습 결과

이번 실습을 통해 다음 과정을 직접 수행하고 확인

- 추가 디스크 확인
- `fdisk`를 이용한 MBR 파티션 구성
- Primary / Extended / Logical 파티션 생성
- XFS 및 ext4 파일시스템 생성
- 마운트 포인트 생성
- `mount`를 이용한 수동 마운트
- `lsblk -f`, `df -hT`를 이용한 마운트 상태 확인
- 재부팅을 통한 수동 마운트 해제 확인
- UUID 기반 `/etc/fstab` 자동 마운트 설정
- `mount -a`를 이용한 설정 검증
- 재부팅 후 자동 마운트 최종 확인
