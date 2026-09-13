# Linux Disk Quota 실습

## 1. 실습 목표

Linux의 Disk Quota를 이용하여 사용자와 그룹별로 디스크 사용량을 제한하고 실제 제한 동작을 확인한다.

이번 실습에서는 다음 항목을 확인하였다.

- ext4 파일시스템에 Quota 기능 활성화
- User Block Quota 설정
- Soft Limit / Hard Limit / Grace Period 확인
- Group Block Quota 설정
- 여러 사용자의 디스크 사용량을 그룹 단위로 제한
- User Inode Quota 설정
- 파일 개수 제한 확인
- Hard Limit 초과 시 실제 쓰기 차단 확인

---

## 2. 실습 구성

Quota 실습용으로 `/dev/sdc` 10GB 디스크를 사용하였다.

```text
/dev/sdc 10G
├── /dev/sdc1 5G
│   └── ext4
│       └── /githome
│           └── User Quota
│
└── /dev/sdc2 5G
    └── ext4
        └── /winhome
            └── Group Quota
```

---

## 3. 파티션 생성

`fdisk`를 사용하여 `/dev/sdc`를 두 개의 5GB 파티션으로 구성하였다.

```bash
fdisk /dev/sdc
```

파티션 확인 결과:

```text
Disk /dev/sdc: 10 GiB, 10737418240 bytes, 20971520 sectors
Disk model: VMware Virtual S
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x88a23c89

Device     Boot    Start      End  Sectors Size Id Type
/dev/sdc1           2048 10487807 10485760   5G 83 Linux
/dev/sdc2       10487808 20971519 10483712   5G 83 Linux
```

구성:

```text
/dev/sdc1 → 5G
/dev/sdc2 → 5G
```

---

## 4. ext4 파일시스템 생성

두 파티션에 ext4 파일시스템을 생성하였다.

```bash
mkfs.ext4 /dev/sdc1
mkfs.ext4 /dev/sdc2
```

확인:

```bash
lsblk -f
```

결과:

```text
sdc
├─sdc1
│    ext4   1.0   7841e388-79a2-4ff2-b0aa-71dbcc1daf0c
└─sdc2
     ext4   1.0   e2f8d872-50c7-4286-9eda-dcc986108291
```

---

## 5. 마운트

마운트 디렉터리를 생성하였다.

```bash
mkdir -p /githome
mkdir -p /winhome
```

마운트:

```bash
mount /dev/sdc1 /githome
mount /dev/sdc2 /winhome
```

확인:

```bash
df -hT /githome /winhome
```

결과:

```text
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdc1      ext4  4.9G   24K  4.6G   1% /githome
/dev/sdc2      ext4  4.9G   24K  4.6G   1% /winhome
```

---

## 6. /etc/fstab Quota 설정

재부팅 후에도 자동으로 마운트되고 Quota 옵션이 적용되도록 `/etc/fstab`을 설정하였다.

```text
UUID=7841e388-79a2-4ff2-b0aa-71dbcc1daf0c  /githome  ext4  defaults,usrquota           0 2
UUID=e2f8d872-50c7-4286-9eda-dcc986108291  /winhome  ext4  defaults,usrquota,grpquota  0 2
```

설정 반영:

```bash
systemctl daemon-reload

mount -o remount /githome
mount -o remount /winhome
```

확인:

```bash
mount | grep -E '/githome|/winhome'
```

결과:

```text
/dev/sdc1 on /githome type ext4 (rw,relatime,seclabel,quota,usrquota)
/dev/sdc2 on /winhome type ext4 (rw,relatime,seclabel,quota,usrquota,grpquota)
```

---

## 7. Quota 명령어 확인

Quota 패키지가 설치되어 있는지 확인하였다.

```bash
rpm -q quota

which quota
which edquota
which repquota
which quotaon
```

결과:

```text
quota-4.09-4.el9.x86_64
/usr/bin/quota
/usr/sbin/edquota
/usr/sbin/repquota
/usr/sbin/quotaon
```

주요 명령어:

| 명령어 | 역할 |
|---|---|
| `quota` | 사용자 또는 그룹 Quota 상태 확인 |
| `edquota` | 사용자 또는 그룹 제한값 설정 |
| `repquota` | 파일시스템 전체 Quota 현황 확인 |
| `quotaon` | Quota 활성화 상태 확인 및 활성화 |

---

## 8. ext4 Quota Feature 활성화

파일시스템의 기존 Feature를 확인하였다.

```bash
tune2fs -l /dev/sdc1 | grep -E 'Filesystem features|User quota inode|Group quota inode'
tune2fs -l /dev/sdc2 | grep -E 'Filesystem features|User quota inode|Group quota inode'
```

초기에는 `quota` Feature가 존재하지 않았다.

파일시스템을 마운트 해제하였다.

```bash
umount /githome
umount /winhome
```

ext4 자체 Quota Feature를 활성화하였다.

```bash
tune2fs -O quota /dev/sdc1
tune2fs -O quota /dev/sdc2
```

확인:

```bash
tune2fs -l /dev/sdc1 | grep -E 'Filesystem features|User quota inode|Group quota inode'
tune2fs -l /dev/sdc2 | grep -E 'Filesystem features|User quota inode|Group quota inode'
```

결과:

```text
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize quota metadata_csum
User quota inode:         3
Group quota inode:        4
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize quota metadata_csum
User quota inode:         3
Group quota inode:        4
```

두 파일시스템 모두 ext4 Quota Feature가 활성화되었다.

다시 마운트:

```bash
mount /githome
mount /winhome
```

Quota 활성화 상태 확인:

```bash
quotaon -p /githome
quotaon -p /winhome
```

결과:

```text
group quota on /githome (/dev/sdc1) is on
user quota on /githome (/dev/sdc1) is on
project quota on /githome (/dev/sdc1) is off

group quota on /winhome (/dev/sdc2) is on
user quota on /winhome (/dev/sdc2) is on
project quota on /winhome (/dev/sdc2) is off
```

---

# 9. User Block Quota 실습

## 9-1. 테스트 사용자 생성

`quser1`의 홈 디렉터리를 `/githome`에 생성하였다.

```bash
useradd -d /githome/quser1 -m quser1
```

---

## 9-2. 사용자 용량 제한 설정

```bash
edquota -u quser1
```

`/dev/sdc1`에 다음 제한을 설정하였다.

```text
Soft Limit = 10240 KB
Hard Limit = 15360 KB
```

즉:

```text
Soft Limit ≈ 10 MiB
Hard Limit ≈ 15 MiB
```

확인:

```bash
quota -u quser1
```

결과:

```text
Disk quotas for user quser1 (uid 1004):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdc1      28   10240   15360               7       0       0
```

여기서:

```text
quota = Soft Limit
limit = Hard Limit
```

이다.

---

## 9-3. User Hard Limit 테스트

먼저 `quser1` 권한으로 파일을 생성하였다.

```bash
runuser -u quser1 -- dd if=/dev/zero of=/githome/quser1/file1 bs=1M count=8
runuser -u quser1 -- dd if=/dev/zero of=/githome/quser1/file2 bs=1M count=5
```

이후 추가로 5MiB 파일 생성을 시도하였다.

```bash
runuser -u quser1 -- dd if=/dev/zero of=/githome/quser1/file3 bs=1M count=5
```

결과:

```text
sdc1: write failed, user block limit reached.
dd: '/githome/quser1/file3'에 쓰는 도중 오류 발생: 디스크 할당량이 초과됨
2+0 records in
1+0 records out
2068480 bytes (2.1 MB, 2.0 MiB) copied, 0.00153153 s, 1.4 GB/s
```

Hard Limit으로 인해 요청한 5MiB 전체가 기록되지 않고 약 2MiB까지만 기록되었다.

Quota 상태 확인:

```bash
quota -u quser1
```

결과:

```text
Disk quotas for user quser1 (uid 1004):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdc1   15360*  10240   15360   7days      10       0       0
```

파일 확인:

```bash
ls -lh /githome/quser1/
```

결과:

```text
합계 15M
-rw-r--r--. 1 quser1 quser1 8.0M  9월 13 15:38 file1
-rw-r--r--. 1 quser1 quser1 5.0M  9월 13 15:38 file2
-rw-r--r--. 1 quser1 quser1 2.0M  9월 13 15:39 file3
```

최종적으로 약 15MiB에서 추가 쓰기가 차단되었다.

```text
Soft Limit = 10 MiB
Hard Limit = 15 MiB

사용량 15 MiB
→ Hard Limit 도달
→ 추가 쓰기 차단
```

`*` 표시는 Soft Limit을 초과한 상태이며 `7days`는 Grace Period를 의미한다.

---

# 10. Group Block Quota 실습

## 10-1. 그룹 및 사용자 생성

그룹 생성:

```bash
groupadd winteam
```

세 사용자의 주 그룹을 `winteam`으로 지정하였다.

```bash
useradd -d /winhome/guser1 -m -g winteam guser1
useradd -d /winhome/guser2 -m -g winteam guser2
useradd -d /winhome/guser3 -m -g winteam guser3
```

확인:

```bash
getent group winteam

id guser1
id guser2
id guser3
```

결과:

```text
winteam:x:1006:

uid=1005(guser1) gid=1006(winteam) groups=1006(winteam)
uid=1006(guser2) gid=1006(winteam) groups=1006(winteam)
uid=1007(guser3) gid=1006(winteam) groups=1006(winteam)
```

홈 디렉터리:

```text
drwx------. 3 guser1 winteam 4096  9월 13 15:42 /winhome/guser1
drwx------. 3 guser2 winteam 4096  9월 13 15:42 /winhome/guser2
drwx------. 3 guser3 winteam 4096  9월 13 15:42 /winhome/guser3
```

---

## 10-2. Group Quota 설정

```bash
edquota -g winteam
```

`/dev/sdc2`에 다음 값을 설정하였다.

```text
Soft Limit = 20480 KB
Hard Limit = 40960 KB
```

즉:

```text
Soft Limit ≈ 20 MiB
Hard Limit ≈ 40 MiB
```

확인:

```bash
quota -g winteam
```

결과:

```text
Disk quotas for group winteam (gid 1006):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdc2      84   20480   40960              21       0       0
```

---

## 10-3. Group Hard Limit 테스트

`guser1`이 15MiB를 생성하였다.

```bash
runuser -u guser1 -- dd if=/dev/zero of=/winhome/guser1/file1 bs=1M count=15
```

`guser2`도 15MiB를 생성하였다.

```bash
runuser -u guser2 -- dd if=/dev/zero of=/winhome/guser2/file2 bs=1M count=15
```

이후 `guser3`이 추가로 15MiB를 생성하도록 시도하였다.

```bash
runuser -u guser3 -- dd if=/dev/zero of=/winhome/guser3/file3 bs=1M count=15
```

결과:

```text
sdc2: write failed, group block limit reached.
dd: '/winhome/guser3/file3'에 쓰는 도중 오류 발생: 디스크 할당량이 초과됨
10+0 records in
9+0 records out
10399744 bytes (10 MB, 9.9 MiB) copied, 0.005537 s, 1.9 GB/s
```

그룹 Hard Limit으로 인해 `guser3`의 파일은 약 10MiB까지만 생성되었다.

확인:

```bash
quota -g winteam
```

결과:

```text
Disk quotas for group winteam (gid 1006):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdc2   40960*  20480   40960   7days      24       0       0
```

파일 확인:

```bash
du -sh /winhome/guser1 /winhome/guser2 /winhome/guser3

ls -lh /winhome/guser1/ /winhome/guser2/ /winhome/guser3/
```

결과:

```text
16M     /winhome/guser1
16M     /winhome/guser2
10M     /winhome/guser3
```

```text
/winhome/guser1/:
합계 15M
-rw-r--r--. 1 guser1 winteam 15M  9월 13 15:43 file1

/winhome/guser2/:
합계 15M
-rw-r--r--. 1 guser2 winteam 15M  9월 13 15:43 file2

/winhome/guser3/:
합계 10M
-rw-r--r--. 1 guser3 winteam 10M  9월 13 15:44 file3
```

그룹 전체 사용량이 Hard Limit인 약 40MiB에 도달하면서 추가 기록이 차단되었다.

```text
guser1 15 MiB
       +
guser2 15 MiB
       +
guser3 약 10 MiB
       ↓
winteam 약 40 MiB
       ↓
Group Hard Limit 도달
       ↓
추가 쓰기 차단
```

---

# 11. User Inode Quota 실습

Block Quota는 디스크 용량을 제한하고, Inode Quota는 생성 가능한 파일 및 디렉터리 개수를 제한한다.

## 11-1. 사용자 생성

```bash
useradd -d /githome/quser2 -m quser2
```

---

## 11-2. Inode Limit 설정

```bash
edquota -u quser2
```

`/dev/sdc1`에 다음 값을 설정하였다.

```text
Block Soft = 0
Block Hard = 0

Inode Soft = 20
Inode Hard = 30
```

확인:

```bash
quota -u quser2
```

결과:

```text
Disk quotas for user quser2 (uid 1008):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdc1      28       0       0               7      20      30
```

홈 디렉터리에 기본 파일이 존재하므로 이미 inode 7개를 사용하고 있었다.

---

## 11-3. Inode Hard Limit 테스트

30개의 파일 생성을 시도하였다.

```bash
runuser -u quser2 -- bash -c 'for i in {1..30}; do touch /githome/quser2/test$i || break; done'
```

결과:

```text
sdc1: warning, user file quota exceeded.
sdc1: write failed, user file limit reached.
touch: cannot touch '/githome/quser2/test24': 디스크 할당량이 초과됨
```

Quota 확인:

```bash
quota -u quser2
```

결과:

```text
Disk quotas for user quser2 (uid 1008):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdc1      28       0       0              30*     20      30   7days
```

생성된 테스트 파일 수 확인:

```bash
ls -l /githome/quser2/test* 2>/dev/null | wc -l
```

결과:

```text
23
```

기존 inode 7개와 새로 생성된 파일 23개를 합하면:

```text
기존 inode = 7
테스트 파일 = 23
────────────────
총 inode = 30
```

Hard Limit인 30개에 도달하여 `test24` 생성부터 차단되었다.

---

# 12. Soft Limit / Hard Limit / Grace Period

## Soft Limit

Soft Limit은 경고 기준이다.

Soft Limit을 초과해도 일정 시간 동안 추가 사용이 가능하다.

```text
Soft Limit 초과
        ↓
Grace Period 시작
        ↓
일정 기간 동안 사용 가능
```

이번 실습에서는 Grace Period가 다음과 같이 확인되었다.

```text
7days
```

---

## Hard Limit

Hard Limit은 절대 상한선이다.

Hard Limit에 도달하면 Grace Period와 관계없이 추가 사용이 제한된다.

실제 실습 결과:

```text
user block limit reached
group block limit reached
user file limit reached
디스크 할당량이 초과됨
```

을 확인하였다.

---

# 13. Block Quota와 Inode Quota

| 구분 | 제한 대상 |
|---|---|
| Block Quota | 사용 가능한 디스크 용량 |
| Inode Quota | 생성 가능한 파일 및 디렉터리 개수 |

예:

```text
Block Quota

사용자 최대 15 MiB
→ 파일 크기 합계가 제한에 도달하면 쓰기 차단
```

```text
Inode Quota

사용자 최대 30개
→ 파일/디렉터리 개수가 제한에 도달하면 생성 차단
```

---

# 14. User Quota와 Group Quota

## User Quota

특정 사용자 한 명의 사용량을 제한한다.

```text
quser1
└── Hard Limit 15 MiB
```

이번 실습에서는 `quser1`이 약 15MiB에 도달하자 추가 쓰기가 차단되었다.

---

## Group Quota

특정 그룹에 속한 파일들의 사용량을 그룹 단위로 제한한다.

```text
winteam
├── guser1
├── guser2
└── guser3
```

세 사용자의 사용량을 합산하여 약 40MiB에 도달하자 추가 쓰기가 차단되었다.

---

# 15. 주요 명령어

| 명령어 | 설명 |
|---|---|
| `quota -u USER` | 사용자 Quota 확인 |
| `quota -g GROUP` | 그룹 Quota 확인 |
| `edquota -u USER` | 사용자 Quota 설정 |
| `edquota -g GROUP` | 그룹 Quota 설정 |
| `repquota` | 파일시스템 Quota 현황 확인 |
| `quotaon` | Quota 활성화 |
| `quotaon -p` | Quota 활성화 상태 확인 |
| `tune2fs -O quota` | ext4 Quota Feature 활성화 |
| `findmnt` | 파일시스템 마운트 상태 및 옵션 확인 |
| `df -hT` | 파일시스템 용량 및 타입 확인 |

---

# 16. 실습 결과

이번 실습을 통해 다음 내용을 확인하였다.

```text
1. ext4 파일시스템에 Quota 기능 활성화

2. User Block Quota
   quser1
   Soft 10 MiB
   Hard 15 MiB
   → Hard Limit에서 실제 쓰기 차단 확인

3. Group Block Quota
   winteam
   Soft 20 MiB
   Hard 40 MiB
   → 여러 사용자의 그룹 사용량 합계가
     Hard Limit에 도달하자 쓰기 차단 확인

4. User Inode Quota
   quser2
   Soft 20개
   Hard 30개
   → inode 30개 도달 후 새로운 파일 생성 차단 확인

5. Soft Limit 초과 시 Grace Period 적용 확인

6. Hard Limit 도달 시
   Disk quota exceeded 동작 확인
```

Disk Quota를 이용하면 다중 사용자 환경에서 특정 사용자 또는 그룹이 디스크 공간이나 inode를 과도하게 사용하는 것을 제한할 수 있다.
