# Linux Disk Quota 사용자 및 그룹 용량 제한 실습

## 실습 개요

Rocky Linux에서 Disk Quota를 이용하여 사용자와 그룹의 디스크 사용량 및 파일 개수를 제한하는 방법을 실습하였다.

100GB 디스크를 50GB씩 분할하여 각각 `/githome`, `/winhome`에 마운트한 뒤 다음 항목을 확인하였다.

- User Quota를 이용한 사용자별 디스크 용량 제한
- Soft Limit / Hard Limit / Grace Period 동작
- Inode Quota를 이용한 파일 개수 제한
- Group Quota를 이용한 그룹 전체 용량 제한
- SetGID를 이용한 그룹 소유권 상속
- 여러 사용자의 디스크 사용량이 하나의 Group Quota로 합산되는 과정

---

## 실습 과정

### 1. 실습용 디스크 확인 및 quota 설치 확인

100GB 크기의 `/dev/sdd` 디스크를 사용하였다.

```bash
[root@Server-A ~]# lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sdd
NAME  SIZE TYPE FSTYPE MOUNTPOINTS
sdd   100G disk

[root@Server-A ~]# rpm -qa | grep quota
quota-nls-4.09-4.el9.noarch
quota-4.09-4.el9.x86_64
```

---

### 2. 50GB 파티션 2개 생성

`fdisk`를 이용하여 `/dev/sdd`를 50GB씩 두 개의 Primary Partition으로 구성하였다.

```text
/dev/sdd1 → 50GB
/dev/sdd2 → 나머지 약 50GB
```

확인:

```text
Disk /dev/sdd: 100 GiB, 107374182400 bytes, 209715200 sectors
Disk model: VMware Virtual S
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x748cca9b

Device     Boot     Start       End   Sectors Size Id Type
/dev/sdd1            2048 104859647 104857600  50G 83 Linux
/dev/sdd2       104859648 209715199 104855552  50G 83 Linux
```

---

### 3. ext4 파일시스템 생성

두 파티션 모두 ext4 파일시스템으로 구성하였다.

```bash
[root@Server-A ~]# mkfs.ext4 /dev/sdd1
[root@Server-A ~]# mkfs.ext4 /dev/sdd2
```

확인:

```bash
[root@Server-A ~]# lsblk -f /dev/sdd
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdd
├─sdd1 ext4   1.0         a476bbf1-836a-4c8a-acb0-7c2f6542b066
└─sdd2 ext4   1.0         29dc3825-6c18-4806-8daf-7464a58a7d44
```

---

### 4. `/githome`, `/winhome` 마운트

마운트 포인트를 생성하였다.

```bash
[root@Server-A ~]# mkdir /githome
[root@Server-A ~]# mkdir /winhome
```

각 파티션을 마운트하였다.

```bash
[root@Server-A ~]# mount /dev/sdd1 /githome
[root@Server-A ~]# mount /dev/sdd2 /winhome
```

`/etc/fstab`에 UUID를 이용하여 영구 마운트를 설정하였다.

```text
UUID=a476bbf1-836a-4c8a-acb0-7c2f6542b066 /githome ext4 defaults 0 0
UUID=29dc3825-6c18-4806-8daf-7464a58a7d44 /winhome ext4 defaults 0 0
```

설정 검증:

```bash
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /githome
[root@Server-A ~]# umount /winhome
[root@Server-A ~]# mount -a
```

확인:

```bash
[root@Server-A ~]# mount | grep -E '/githome|/winhome'
/dev/sdd1 on /githome type ext4 (rw,relatime,seclabel)
/dev/sdd2 on /winhome type ext4 (rw,relatime,seclabel)
```

```bash
[root@Server-A ~]# lsblk -f /dev/sdd
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdd
├─sdd1 ext4   1.0         a476bbf1-836a-4c8a-acb0-7c2f6542b066   46.4G     0% /githome
└─sdd2 ext4   1.0         29dc3825-6c18-4806-8daf-7464a58a7d44   46.4G     0% /winhome
```

---

### 5. User Quota 테스트 계정 생성

`/githome`을 홈 디렉터리로 사용하는 사용자 세 명을 생성하였다.

```bash
[root@Server-A ~]# useradd -md /githome/quser1 quser1
[root@Server-A ~]# useradd -md /githome/quser2 quser2
[root@Server-A ~]# useradd -md /githome/quser3 quser3
```

확인:

```bash
[root@Server-A ~]# ls -l /githome
합계 28
drwx------. 2 root   root   16384  9월 17 11:35 lost+found
drwx------. 3 quser1 quser1  4096  9월 17 11:48 quser1
drwx------. 3 quser2 quser2  4096  9월 17 11:48 quser2
drwx------. 3 quser3 quser3  4096  9월 17 11:48 quser3
```

```bash
[root@Server-A ~]# grep quser /etc/passwd
quser1:x:1006:1006::/githome/quser1:/bin/bash
quser2:x:1007:1007::/githome/quser2:/bin/bash
quser3:x:1008:1008::/githome/quser3:/bin/bash
```

---

### 6. `/githome` User Quota 활성화

`/etc/fstab`의 `/githome`에 `usrquota` 내용을 추가하였다.

```text
UUID=a476bbf1-836a-4c8a-acb0-7c2f6542b066 /githome ext4 defaults,usrquota 0 0
```

```bash
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# mount -o remount /dev/sdd1
```

확인:

```bash
[root@Server-A ~]# mount | grep sdd1
/dev/sdd1 on /githome type ext4 (rw,relatime,seclabel,quota,usrquota)
```

---

### 7. User Quota 관리 파일 생성

`quotacheck`를 이용하여 사용자 Quota 관리 파일을 생성하였다.

```bash
[root@Server-A ~]# quotacheck -cu /githome
```

확인:

```bash
[root@Server-A ~]# ls -l /githome
합계 36
-rw-------. 1 root   root    7168  9월 17 11:51 aquota.user
drwx------. 2 root   root   16384  9월 17 11:35 lost+found
drwx------. 3 quser1 quser1  4096  9월 17 11:48 quser1
drwx------. 3 quser2 quser2  4096  9월 17 11:48 quser2
drwx------. 3 quser3 quser3  4096  9월 17 11:48 quser3
```

`aquota.user`가 생성된 것을 확인하였다.

---

### 8. `quser1` Block Quota 설정

`quser1`의 디스크 용량을 다음과 같이 제한하였다.

```text
Block Soft Limit → 10MB
Block Hard Limit → 15MB
Inode Limit      → 제한 없음
```

```bash
[root@Server-A ~]# quotaoff /githome
[root@Server-A ~]# edquota quser1
```

설정값:

```text
Filesystem     blocks   soft    hard   inodes   soft   hard
/dev/sdd1          28   10240   15360       7      0      0
```

Quota 활성화 후 확인:

```bash
[root@Server-A ~]# quotaon /githome

[root@Server-A ~]# quota -u quser1
Disk quotas for user quser1 (uid 1006):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdd1      28   10240   15360               7       0       0
```

---

### 9. `quser1` Soft / Hard Limit 테스트

`quser1`으로 전환한 뒤 `/etc`의 파일들을 복사하여 디스크 사용량을 증가시켰다.

```bash
[quser1@Server-A ~]$ cp -r /etc/* ./
```

Soft Limit을 초과하자 경고 메세지가 출력되었다.

```text
sdd1: warning, user block quota exceeded.
```

Hard Limit에 도달하자 추가 쓰기가 차단되었다.

```text
sdd1: write failed, user block limit reached.
cp: './ssh/moduli'에 쓰는 도중 오류 발생: 디스크 할당량이 초과됨
```

Quota 상태 확인:

```bash
[root@Server-A ~]# repquota /githome
*** Report for user quotas on device /dev/sdd1
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      --      20       0       0              2     0     0
quser1    +-   15360   10240   15360  6days    1898     0     0
quser2    --      28       0       0              7     0     0
quser3    --      28       0       0              7     0     0
```

`quser1`의 상태가 `+-`로 표시되었으며 Block Soft Limit을 초과하여 Grace Period가 적용된 것을 확인하였다.

---

### 10. 사용자별 Quota 조건 설정

`quser2`에는 파일 개수 제한만 설정하였다.

```bash
[root@Server-A ~]# edquota quser2
```

```text
Block Limit → 제한 없음
Inode Soft  → 10000
Inode Hard  → 15000
```

`quser3`에는 디스크 용량과 파일 개수 제한을 모두 설정하였다.

```bash
[root@Server-A ~]# edquota quser3
```

```text
Block Soft  → 61440KB = 60MB
Block Hard  → 92160KB = 90MB
Inode Soft  → 10000
Inode Hard  → 15000
```

전체 상태 확인:

```bash
[root@Server-A ~]# repquota /githome
*** Report for user quotas on device /dev/sdd1
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      --      20       0       0              2     0     0
quser1    +-   15360   10240   15360  6days    1898     0     0
quser2    --      28       0       0              7 10000 15000
quser3    --      28   61440   92160              7 10000 15000
```

---

### 11. `quser2` Inode Quota 테스트

`quser2`에 설정한 Inode Soft 10000 / Hard 15000의 실제 동작을 확인하였다.

```bash
[root@Server-A ~]# su - quser2
```

반복문을 이용하여 빈 파일을 생성하였다.

```bash
[quser2@Server-A ~]$ i=1
while [ $i -le 16000 ]
do
    touch inode_$i || {
        echo "생성 실패: inode_$i"
        break
    }
    i=$((i+1))
done
```

결과:

```text
sdd1: warning, user file quota exceeded.
sdd1: write failed, user file limit reached.
touch: cannot touch 'inode_14993': 디스크 할당량이 초과됨
생성 실패: inode_14993
```

Quota 확인:

```bash
[quser2@Server-A ~]$ quota -u quser2
Disk quotas for user quser2 (uid 1007):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdd1     492       0       0           15000*  10000   15000   7days
```

직접 생성된 테스트 파일 개수:

```bash
[quser2@Server-A ~]$ find . -maxdepth 1 -name 'inode_*' | wc -l
14992
```

기존에 `quser2`가 소유하고 있던 홈 디렉터리와 기본 파일들의 inode까지 포함하여 총 inode 사용량이 15000에 도달하자 추가 파일 생성이 차단되는 것을 확인하였다.

---

### 12. `/winhome` User / Group Quota 활성화

`/etc/fstab`의 `/winhome`에 User Quota와 Group Quota 내용을 추가하였다.

```text
UUID=29dc3825-6c18-4806-8daf-7464a58a7d44 /winhome ext4 defaults,usrquota,grpquota 0 0
```

설정 반영:

```bash
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# mount -o remount /winhome
```

확인:

```bash
[root@Server-A ~]# mount | grep /winhome
/dev/sdd2 on /winhome type ext4 (rw,relatime,seclabel,quota,usrquota,grpquota)
```

User / Group Quota 관리 파일 생성:

```bash
[root@Server-A ~]# quotacheck -cu /winhome
[root@Server-A ~]# quotacheck -cg /winhome
```

확인:

```bash
[root@Server-A ~]# ls -l /winhome
합계 32
-rw-------. 1 root root  6144  9월 17 12:04 aquota.group
-rw-------. 1 root root  6144  9월 17 12:04 aquota.user
drwx------. 2 root root 16384  9월 17 11:35 lost+found
```

---

### 13. Group Quota 테스트 사용자 및 그룹 생성

`/winhome`을 홈 디렉터리로 사용하는 사용자 세 명을 생성하였다.

```bash
[root@Server-A ~]# useradd -md /winhome/guser1 guser1
[root@Server-A ~]# useradd -md /winhome/guser2 guser2
[root@Server-A ~]# useradd -md /winhome/guser3 guser3
```

확인:

```text
guser1:x:1009:1009::/winhome/guser1:/bin/bash
guser2:x:1010:1010::/winhome/guser2:/bin/bash
guser3:x:1011:1011::/winhome/guser3:/bin/bash
```

`winteam` 그룹을 생성하고 세 사용자를 보조 그룹으로 추가하였다.

```bash
[root@Server-A ~]# groupadd winteam

[root@Server-A ~]# usermod -aG winteam guser1
[root@Server-A ~]# usermod -aG winteam guser2
[root@Server-A ~]# usermod -aG winteam guser3
```

확인:

```bash
[root@Server-A ~]# id guser1
uid=1009(guser1) gid=1009(guser1) groups=1009(guser1),1012(winteam)

[root@Server-A ~]# id guser2
uid=1010(guser2) gid=1010(guser2) groups=1010(guser2),1012(winteam)

[root@Server-A ~]# id guser3
uid=1011(guser3) gid=1011(guser3) groups=1011(guser3),1012(winteam)
```

---

### 14. `winteam` Group Quota 설정

사용자 홈 디렉터리의 그룹 소유권을 `winteam`으로 변경하였다.

```bash
[root@Server-A ~]# chgrp -R winteam /winhome/guser1
[root@Server-A ~]# chgrp -R winteam /winhome/guser2
[root@Server-A ~]# chgrp -R winteam /winhome/guser3
```

확인:

```bash
[root@Server-A ~]# ls -l /winhome
합계 44
-rw-------. 1 root   root     7168  9월 17 12:10 aquota.group
-rw-------. 1 root   root     6144  9월 17 12:04 aquota.user
drwx------. 3 guser1 winteam  4096  9월 17 12:08 guser1
drwx------. 3 guser2 winteam  4096  9월 17 12:08 guser2
drwx------. 3 guser3 winteam  4096  9월 17 12:08 guser3
drwx------. 2 root   root    16384  9월 17 11:35 lost+found
```

그룹 소유권 변경 내용을 Group Quota 관리 정보에 반영하기 위해 Group Quota 정보를 다시 검사하였다.

```bash
[root@Server-A ~]# quotacheck -cg /winhome
```

`winteam`에 Group Quota를 설정하였다.

```bash
[root@Server-A ~]# edquota -g winteam
```

```text
Soft Limit → 20480KB = 20MB
Hard Limit → 40690KB ≈ 40MB
Inode      → 제한 없음
```

설정 후 User / Group Quota를 활성화하였다.

```bash
[root@Server-A ~]# quotaon -ug /winhome
```

Group Quota 활성화 상태를 확인하였다.

```bash
[root@Server-A ~]# quotaon -p /winhome
group quota on /winhome (/dev/sdd2) is on
user quota on /winhome (/dev/sdd2) is on
project quota on /winhome (/dev/sdd2) is off
```

---

### 15. `/winhome` SetGID 설정

`/winhome`에서 생성되는 파일과 디렉터리가 `winteam` 그룹을 상속하도록 SetGID를 설정하였다.

```bash
[root@Server-A ~]# chmod 2777 /winhome
[root@Server-A ~]# chown :winteam /winhome
```

확인:

```bash
[root@Server-A ~]# ls -ld /winhome
drwxrwsrwx. 6 root winteam 4096  9월 17 12:12 /winhome
```

`guser1`이 `/winhome`에 파일을 생성하였다.

```bash
[guser1@Server-A ~]$ dd if=/dev/zero of=/winhome/bigfile.img bs=1M count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 0.00698471 s, 1.5 GB/s
```

파일 그룹 확인:

```bash
[guser1@Server-A ~]$ ls -lh /winhome/bigfile.img
-rw-r--r--. 1 guser1 winteam 10M  9월 17 12:14 /winhome/bigfile.img
```

생성된 파일의 그룹이 자동으로 `winteam`이 된 것을 확인하였다.

SetGID 동작 확인 후 Group Quota 용량 테스트를 위해 테스트 파일을 정리하였다.

```bash
[root@Server-A ~]# rm -f /winhome/bigfile.img

---

### 16. Group Soft / Hard Limit 테스트

`guser1`이 `/winhome`에 10MB 파일을 연속으로 생성하였다.

```bash
[guser1@Server-A ~]$ dd if=/dev/zero of=/winhome/bigfile.img bs=1M count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 0.00545904 s, 1.9 GB/s
```

두 번째 파일을 생성하면서 Group Soft Limit을 초과하였다.

```text
sdd2: warning, group block quota exceeded.
```

세 번째 파일까지는 Grace Period에 의해 생성할 수 있었다.

```text
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 0.00757879 s, 1.4 GB/s
```

네 번째 파일 생성 중 Group Hard Limit에 도달하였다.

```text
sdd2: write failed, group block limit reached.
dd: '/winhome/bigfile4.img'에 쓰는 도중 오류 발생: 디스크 할당량이 초과됨
10+0 records in
9+0 records out
10117120 bytes (10 MB, 9.6 MiB) copied, 0.0206872 s, 489 MB/s
```

최종 Group Quota 상태:

```bash
[root@Server-A ~]# repquota -g /winhome
*** Report for group quotas on device /dev/sdd2
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
Group           used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      --      16       0       0              1     0     0
guser1    --       8       0       0              2     0     0
winteam   +-   40688   20480   40690  6days      26     0     0
```

파일 확인:

```bash
[root@Server-A ~]# ls -lh /winhome
합계 40M
-rw-------. 1 root   root    7.0K  9월 17 12:12 aquota.group
-rw-------. 1 root   root    7.0K  9월 17 12:04 aquota.user
-rw-r--r--. 1 guser1 winteam  10M  9월 17 12:26 bigfile.img
-rw-r--r--. 1 guser1 winteam  10M  9월 17 12:26 bigfile2.img
-rw-r--r--. 1 guser1 winteam  10M  9월 17 12:26 bigfile3.img
-rw-r--r--. 1 guser1 winteam 9.7M  9월 17 12:26 bigfile4.img
```

Group Hard Limit에 도달하면서 마지막 파일이 정상 크기인 10MB를 모두 기록하지 못한 것을 확인하였다.

---

### 17. 여러 사용자의 Group Quota 사용량 합산 확인

기존 Group Quota 테스트 파일을 정리하였다.

```bash
[root@Server-A ~]# rm -f /winhome/bigfile*.img
```

여러 사용자가 같은 `winteam` 그룹 공간을 사용하도록 하였다.

`guser1`이 10MB 파일을 생성하였다.

```bash
[guser1@Server-A ~]$ dd if=/dev/zero of=/winhome/guser1.img bs=1M count=10
```

`guser2`가 15MB 파일을 생성하였다.

```bash
[guser2@Server-A ~]$ dd if=/dev/zero of=/winhome/guser2.img bs=1M count=15
```

동기화 후 Group Quota를 확인하였다.

```bash
[root@Server-A ~]# quota -g winteam
Disk quotas for group winteam (gid 1012):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/sdd2   25688*  20480   40690   7days      24       0       0
```

최종 상태:

```bash
[root@Server-A ~]# repquota -g /winhome
*** Report for group quotas on device /dev/sdd2
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
Group           used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      --      16       0       0              1     0     0
guser1    --       8       0       0              2     0     0
guser2    --       8       0       0              2     0     0
winteam   +-   25688   20480   40690  6days      24     0     0
```

`guser1`과 `guser2`가 각각 파일을 생성했지만 두 사용자의 디스크 사용량이 `winteam`의 하나의 Group Quota로 합산되는 것을 확인하였다.

약 25MB가 사용되어 Group Soft Limit인 20MB를 초과하였고 Grace Period가 시작되었다.

---


## 실습 결과

- 100GB 디스크를 50GB 파티션 2개로 구성
- 두 파티션을 ext4 파일시스템으로 생성
- `/githome`, `/winhome` 영구 마운트 구성
- `/githome`에 User Quota 적용
- `quser1`에 Block Soft 10MB / Hard 15MB 설정
- User Soft Limit 초과 시 Grace Period 적용 확인
- User Hard Limit 도달 시 추가 쓰기 차단 확인
- `quser2`에 Inode Soft 10000 / Hard 15000 설정
- Inode Hard Limit 도달 시 추가 파일 생성 차단 확인
- `quser3`에 Block Quota와 Inode Quota 동시 설정
- `/winhome`에 User Quota와 Group Quota 적용
- `winteam` 그룹에 Soft 20MB / Hard 약 40MB 설정
- `/winhome`에 SetGID를 적용하여 생성 파일의 그룹을 `winteam`으로 상속
- Group Soft Limit 초과 시 Grace Period 적용 확인
- Group Hard Limit 도달 시 추가 쓰기 차단 확인
- `guser1`, `guser2`가 사용한 디스크 공간이 `winteam` Group Quota에 합산되는 것을 확인
- `quota`, `repquota`, `edquota`, `quotacheck`, `quotaon`, `quotaoff` 명령을 이용한 Disk Quota 관리 방법 확인

