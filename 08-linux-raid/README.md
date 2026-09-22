# Linux Software RAID 구성 및 장애 복구 실습

## 실습 개요

Rocky Linux에서 `mdadm`을 이용하여 Linear RAID, RAID 0, RAID 10, RAID 5, RAID 6을 구성하였다.

각 RAID에 ext4 파일시스템을 생성하고 원하는 디렉터리에 마운트한 후 UUID를 이용하여 `/etc/fstab`에 영구 마운트를 설정하였다.

또한 디스크 장애를 발생시켜 RAID별 장애 허용 범위를 확인하고 RAID 10의 Rebuild, RAID 5의 Hot Spare 자동 복구, RAID 6의 다중 디스크 장애 상황을 실습하였다.

---

## 실습 과정

### 1. 디스크 상태 확인

현재 시스템의 디스크와 파일시스템 확인

```
[root@Server-A ~]# lsblk -f
NAME FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1
│    swap   1           a4e70e42-b4f9-468e-a9db-e0f60f7c217b                [SWAP]
└─sda2
     xfs                0d31b54e-e1d8-4b91-8a86-9e5bb76b8950   88.7G     8% /
sdb
sdc
sdd
sde
sdf
sr0  iso966 Jolie Rocky-9-8-x86_64-dvd
                      2026-05-25-17-10-45-00
```

`/dev/sda`는 운영체제가 설치된 100G 디스크이므로 RAID 실습 대상에서 제외하였다.

`/dev/sdb~sdf` 100G 디스크를 RAID 실습용으로 사용하였다.

---

### 2. RAID용 파티션 생성

`fdisk`를 이용하여 RAID에서 사용할 파티션 생성

```
[root@Server-A ~]# fdisk /dev/sdb

Command (m for help): n
Select (default p):
Partition number (1-4, default 1):
First sector:
Last sector:

Command (m for help): t
Hex code or alias (type L to list all): fd

Command (m for help): p

Device     Boot Start       End   Sectors  Size Id Type
/dev/sdb1        2048 209715199 209713152  100G fd Linux raid autodetect

Command (m for help): w
```

`/dev/sdc`, `/dev/sdd`, `/dev/sde`, `/dev/sdf`도 동일한 방법으로 RAID용 파티션을 생성하였다.

파티션 Type은 `fd Linux raid autodetect`로 설정하였다.
```
[root@Server-A ~]# lsblk -f
sdb
└─sdb1
sdc
└─sdc1
sdd
└─sdd1
sde
└─sde1
sdf
└─sdf1
```

---

### 3. Linear RAID 구성

`/dev/sdb1`, `/dev/sdc1`을 이용하여 Linear RAID 생성

```
[root@Server-A ~]# mdadm --create /dev/md0 --level=linear --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

RAID 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
        Raid Level : linear
        Array Size : 209580032 (199.87 GiB 214.61 GB)
      Raid Devices : 2
     Active Devices : 2
    Working Devices : 2
     Failed Devices : 0
              State : clean
```

100G 디스크 2개의 공간이 연결되어 약 200G의 Linear RAID가 생성되는 것을 확인하였다.

---

### 4. Linear RAID 파일시스템 생성 및 /linear 마운트

`/dev/md0`에 ext4 파일시스템 생성

```bash
[root@Server-A ~]# mkfs.ext4 /dev/md0
```

마운트 디렉터리 생성 후 마운트

```bash
[root@Server-A ~]# mkdir /linear
[root@Server-A ~]# mount /dev/md0 /linear
```

확인

```bash
[root@Server-A ~]# lsblk -f /dev/md0
NAME FSTYPE FSVER UUID                                  FSAVAIL FSUSE% MOUNTPOINTS
md0  ext4   1.0   f089dd92-4044-4ffa-aad2-10c4bacd3116  185.7G     0% /linear
```

`/dev/md0`이 `/linear`에 마운트되었다.

---

### 5. Linear RAID /etc/fstab 영구 마운트 설정

`/etc/fstab`에 다음 내용을 추가하였다.

```
UUID=f089dd92-4044-4ffa-aad2-10c4bacd3116 /linear ext4 defaults 0 0
```

설정 반영 및 재마운트 테스트

```
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /linear
[root@Server-A ~]# mount -a
```

확인

```
[root@Server-A ~]# df -hT /linear
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md0       ext4  196G   28K  186G   1% /linear
```

UUID 기반 영구 마운트 설정이 정상적으로 적용되는 것을 확인하였다.

---

### 6. RAID 0 구성

기존 Linear RAID를 제거한 후 `/dev/sdb1`, `/dev/sdc1`을 RAID 0으로 구성하였다.

```
[root@Server-A ~]# mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

RAID 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md0
/dev/md0:
        Raid Level : raid0
        Array Size : 209580032 (199.87 GiB 214.61 GB)
      Raid Devices : 2
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
        Chunk Size : 512K
```

RAID 0은 Striping 방식으로 100G 디스크 2개의 전체 공간을 사용하여 약 200G의 저장 공간이 생성되었다.

RAID 0은 성능은 높지만 디스크 장애 허용 기능은 제공하지 않는다.

---

### 7. RAID 0 파일시스템 및 /RAID0 마운트

ext4 파일시스템 생성

```
[root@Server-A ~]# mkfs.ext4 /dev/md0
```


마운트

```
[root@Server-A ~]# mkdir /RAID0
[root@Server-A ~]# mount /dev/md0 /RAID0
```

확인
```
[root@Server-A ~]# lsblk -f /dev/md0
NAME FSTYPE FSVER LABEL UUID                                  FSAVAIL FSUSE% MOUNTPOINTS
md0  ext4   1.0         fe4c96db-a780-488d-8587-57b5351bf2fe  185.7G     0% /RAID0


```

`/etc/fstab`에 다음 내용을 추가하였다.

```
UUID=fe4c96db-a780-488d-8587-57b5351bf2fe /RAID0 ext4 defaults 0 0
```

설정 반영 및 재마운트

```
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /RAID0
[root@Server-A ~]# mount -a
```

확인

```
[root@Server-A ~]# df -hT /RAID0
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md0       ext4  196G   28K  186G   1% /RAID0
```

RAID 0 영구 마운트 설정이 정상적으로 동작하는 것을 확인하였다.

---

### 8. RAID 10 구성

`RAID 0`을 제거한 후 `/dev/sdb1`, `/dev/sdc1`, `/dev/sdd1`, `/dev/sde1`을 이용하여 RAID 10 생성

```
[root@Server-A ~]# mdadm --create /dev/md10 --level=10 --raid-devices=4 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1

mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md10 started.
```

RAID 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md10
/dev/md10:
           Version : 1.2
        Raid Level : raid10
        Array Size : 209580032 (199.87 GiB 214.61 GB)
     Used Dev Size : 104790016 (99.94 GiB 107.30 GB)
      Raid Devices : 4
     Total Devices : 4
      Intent Bitmap : Internal
              State : clean, resyncing
     Active Devices : 4
    Working Devices : 4
     Failed Devices : 0
             Layout : near=2
         Chunk Size : 512K
```

RAID 10은 Mirroring과 Striping을 결합한 구조로 4개의 100G 디스크에서 약 200G를 사용할 수 있는 것을 확인하였다.

---

### 9. RAID 10 파일시스템 및 /RAID10 마운트

ext4 파일시스템 생성

```
[root@Server-A ~]# mkfs.ext4 /dev/md10
```

마운트

```
[root@Server-A ~]# mkdir /RAID10
[root@Server-A ~]# mount /dev/md10 /RAID10
```

확인

```
[root@Server-A ~]# lsblk -f /dev/md10
NAME FSTYPE FSVER UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
md10 ext4   1.0   fdd668de-0de0-4d65-87b9-2e246a79e909 185.7G     0% /RAID10
```

`/etc/fstab`에 다음 내용을 추가하였다.

```
UUID=fdd668de-0de0-4d65-87b9-2e246a79e909 /RAID10 ext4 defaults 0 0
```

재마운트 테스트

```
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /RAID10
[root@Server-A ~]# mount -a

[root@Server-A ~]# df -hT /RAID10
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md10      ext4  196G   28K  186G   1% /RAID10
```

RAID 10 영구 마운트 설정이 정상적으로 동작하는 것을 확인하였다.

---

### 10. RAID 10 데이터 저장 및 디스크 장애 테스트

장애 테스트를 위해 `/etc`에서 `f`로 시작하는 파일과 디렉터리를 `/RAID10`에 복사하였다.

```
[root@Server-A ~]# cp -r /etc/f* /RAID10

[root@Server-A ~]# ls -l /RAID10
합계 56
lrwxrwxrwx. 1 root root    56  9월 18 10:52 favicon.png -> /usr/share/icons/hicolor/16x16/apps/fedora-logo-icon.png
-rw-r--r--. 1 root root    66  9월 18 10:52 filesystems
drwxr-xr-x. 3 root root  4096  9월 18 10:52 firefox
drwxr-x---. 8 root root  4096  9월 18 10:52 firewalld
drwxr-xr-x. 3 root root  4096  9월 18 10:52 flatpak
drwxr-xr-x. 3 root root  4096  9월 18 10:52 fonts
drwxr-xr-x. 3 root root  4096  9월 18 10:52 foomatic
-rw-r--r--. 1 root root    20  9월 18 10:52 fprintd.conf
-rw-r--r--. 1 root root   589  9월 18 10:52 fstab
-rw-r--r--. 1 root root    38  9월 18 10:52 fuse.conf
drwxr-xr-x. 4 root root  4096  9월 18 10:52 fwupd
```

RAID 구성 디스크 1개를 제거한 후 상태 확인

```
[root@Server-A ~]# shutdown now
VMware에서 하드디스크 하나 제거
```

```
[root@Server-A ~]# mdadm --detail /dev/md10
/dev/md10:
              State : clean, degraded
       Raid Devices : 4
      Total Devices : 3
     Active Devices : 3
    Working Devices : 3
     Failed Devices : 0
```

장치 목록에서 RAID 구성 디스크 하나가 `removed` 상태로 표시되었다.

```
Number   Major   Minor   RaidDevice State
   0       8       17        0      active sync set-A   /dev/sdb1
   1       8       33        1      active sync set-B   /dev/sdc1
   -       0        0        2      removed
   3       8       49        3      active sync set-B   /dev/sdd1
```

디스크 1개가 제거된 상태에서도 `/RAID10`의 데이터가 정상적으로 유지되는 것을 확인하였다.

---

### 11. RAID 10 장애 디스크 복구

새 디스크 `/dev/sde1`을 RAID 10에 추가하였다.

```
[root@Server-A ~]# mdadm /dev/md10 --add /dev/sde1
mdadm: added /dev/sde1
```

RAID 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md10
/dev/md10:
              State : clean, degraded, recovering
     Active Devices : 3
    Working Devices : 4
     Failed Devices : 0
      Spare Devices : 1

     Rebuild Status : 4% complete
```

새로 추가된 `/dev/sde1`이 다음과 같이 표시되었다.

```
spare rebuilding   /dev/sde1
```

장애 디스크를 교체하면 RAID 10이 자동으로 데이터를 새 디스크에 Rebuild하는 것을 확인하였다.

---

### 12. RAID 5 구성

기존 RAID 10을 제거 후 `/dev/sdb1`, `/dev/sdc1`, `/dev/sdd1`, `/dev/sde1`을 이용하여 RAID 5 생성

```
[root@Server-A ~]# mdadm --create /dev/md5 --level=5 --raid-devices=4 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1

mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md5 started.
```

확인

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
        Raid Level : raid5
        Array Size : 314370048 (299.81 GiB 321.91 GB)
     Used Dev Size : 104790016 (99.94 GiB 107.30 GB)
      Raid Devices : 4
     Total Devices : 4
             Layout : left-symmetric
         Chunk Size : 512K
```

AID 5는 4개의 100G 디스크에서 1개 디스크 분량의 용량을 Parity에 사용하기 때문에 약 300G의 공간을 사용할 수 있는 것을 확인하였다.

---

### 13. RAID 5 파일시스템 생성 및 /RAID5 마운트

ext4 파일시스템 생성

```
[root@Server-A ~]# mkfs.ext4 /dev/md5
```

마운트 디렉터리 생성 후 마운트

```
[root@Server-A ~]# mkdir /RAID5
[root@Server-A ~]# mount /dev/md5 /RAID5
```

확인
```
[root@Server-A ~]# mount | grep RAID5
/dev/md5 on /RAID5 type ext4 (rw,relatime,seclabel,stripe=384)
```

용량 확인

```
[root@Server-A ~]# df -hT /RAID5
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md5       ext4  295G   28K  280G   1% /RAID5
```

---

### 14. RAID 5 UUID 기반 영구 마운트 설정

`/etc/fstab`에 다음 내용을 추가하였다.

```
UUID=7a029c15-a537-4c1e-8125-5f26a2179a13 /RAID5 ext4 defaults 0 0
```

설정 반영 및 재마운트

```
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /RAID5
[root@Server-A ~]# mount -a
```

확인

```
[root@Server-A ~]# df -hT /RAID5
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md5       ext4  295G   28K  280G   1% /RAID5
```

`/etc/fstab`을 이용하여 RAID 5가 정상적으로 다시 마운트되는 것을 확인하였다.

---

### 15. RAID 5 데이터 저장

장애 테스트를 위해 `/etc`에서 `c`로 시작하는 모든 파일과 디렉터리를 `/RAID5`에 복사하였다.

```
[root@Server-A ~]# cp -r /etc/c* /RAID5

[root@Server-A ~]# ls -l /RAID5
합계 80
drwxr-xr-x. 3 root root  4096  9월 18 11:16 chromium
-rw-r--r--. 1 root root  1370  9월 18 11:16 chrony.conf
-rw-r-----. 1 root root   540  9월 18 11:16 chrony.keys
drwxr-xr-x. 2 root root  4096  9월 18 11:16 cifs-utils
drwxr-xr-x. 4 root root  4096  9월 18 11:16 cockpit
drwxr-xr-x. 2 root root  4096  9월 18 11:16 cron.d
drwxr-xr-x. 2 root root  4096  9월 18 11:16 cron.daily
-rw-r--r--. 1 root root     0  9월 18 11:16 cron.deny
drwxr-xr-x. 6 root root  4096  9월 18 11:16 crypto-policies
-rw-------. 1 root root     0  9월 18 11:16 crypttab
drwxr-xr-x. 4 root root  4096  9월 18 11:16 cups
```

RAID 5 장애 테스트용 데이터가 정상적으로 저장되는 것을 확인하였다.

---

### 16. RAID 5 디스크 장애 테스트

RAID 5가 정상 상태인지 확인하였다.

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
              State : clean
     Active Devices : 4
    Working Devices : 4
     Failed Devices : 0
      Spare Devices : 0
```

`/dev/sdc1`에 논리적인 장애 발생

```
[root@Server-A ~]# mdadm --fail /dev/md5 /dev/sdc1
```

확인

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
              State : clean, degraded
     Active Devices : 3
    Working Devices : 3
     Failed Devices : 1
      Spare Devices : 0
```

장애 디스크

```
faulty   /dev/sdc1
```

데이터 확인

```
[root@Server-A ~]# ls -l /RAID5
합계 80
drwxr-xr-x. 3 root root  4096  9월 18 11:16 chromium
-rw-r--r--. 1 root root  1370  9월 18 11:16 chrony.conf
-rw-r-----. 1 root root   540  9월 18 11:16 chrony.keys
drwxr-xr-x. 2 root root  4096  9월 18 11:16 cifs-utils
drwxr-xr-x. 4 root root  4096  9월 18 11:16 cockpit
drwxr-xr-x. 2 root root  4096  9월 18 11:16 cron.d
drwxr-xr-x. 2 root root  4096  9월 18 11:16 cron.daily
-rw-r--r--. 1 root root     0  9월 18 11:16 cron.deny
drwxr-xr-x. 6 root root  4096  9월 18 11:16 crypto-policies
-rw-------. 1 root root     0  9월 18 11:16 crypttab
drwxr-xr-x. 4 root root  4096  9월 18 11:16 cups
```

디스크 1개가 장애 상태임에도 기존 데이터가 정상적으로 유지되는 것을 확인하였다.

---

### 17. RAID 5 장애 디스크 재추가

Faulty 상태의 `/dev/sdc1`을 바로 다시 추가하였다.

```
[root@Server-A ~]# mdadm --add /dev/md5 /dev/sdc1
mdadm: Cannot open /dev/sdc1: Device or resource busy
```

`/dev/sdc1`이 아직 Faulty 장치로 RAID에 등록되어 있기 때문에 바로 추가할 수 없었다.

RAID에서 장애 디스크 제거

```
[root@Server-A ~]# mdadm --remove /dev/md5 /dev/sdc1
mdadm: hot removed /dev/sdc1 from /dev/md5
```

다시 추가

```
[root@Server-A ~]# mdadm --add /dev/md5 /dev/sdc1
mdadm: re-added /dev/sdc1
```

확인

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
              State : clean, degraded, recovering
     Active Devices : 3
    Working Devices : 4
      Spare Devices : 1

     Rebuild Status : 4% complete

spare rebuilding   /dev/sdc1
```

장애 디스크를 제거한 후 다시 추가하면 RAID가 자동으로 Rebuild되는 것을 확인하였다.

---

### 18. RAID 5 Hot Spare 구성

RAID 5의 정상 Active Disk 4개 외에 `/dev/sdd1`을 Hot Spare로 추가하였다.

```
[root@Server-A ~]# mdadm --add /dev/md5 /dev/sdd1
mdadm: added /dev/sdd1
```

확인

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
      Raid Devices : 4
     Total Devices : 5
              State : clean
     Active Devices : 4
    Working Devices : 5
     Failed Devices : 0
      Spare Devices : 1
```

현재 RAID 구성

```
/dev/sdb1  active sync
/dev/sdc1  active sync
/dev/sde1  active sync
/dev/sdf1  active sync

/dev/sdd1  spare
```

4개의 디스크는 RAID 5의 실제 구성 디스크로 사용되고 `/dev/sdd1`은 장애 발생에 대비한 Hot Spare로 대기하는 것을 확인하였다.

---

### 19. RAID 5 Hot Spare 자동 복구 테스트

Active Disk인 `/dev/sdb1`에 장애 발생

```
[root@Server-A ~]# mdadm --fail /dev/md5 /dev/sdb1
```

확인

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
              State : clean, degraded, recovering
     Active Devices : 3
    Working Devices : 4
     Failed Devices : 1
      Spare Devices : 1

     Rebuild Status : 2% complete
```

Hot Spare였던 `/dev/sdd1`의 상태

```
spare rebuilding   /dev/sdd1
```

`/dev/sdb1`에 장애가 발생하자 Hot Spare인 `/dev/sdd1`이 자동으로 RAID에 투입되어 Rebuild를 시작하는 것을 확인하였다.

---

### 20. RAID 5 Hot Spare 복구 완료 확인

Rebuild 완료 후 상태 확인

```
[root@Server-A ~]# cat /proc/mdstat
Personalities : [raid4] [raid5] [raid6]
md5 : active raid5 sdd1[5] sdf1[4] sdc1[1] sde1[2] sdb1[0](F)
      314370048 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]

unused devices: <none>
```

상세 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
              State : clean
     Active Devices : 4
    Working Devices : 4
     Failed Devices : 1
      Spare Devices : 0
```

기존 Hot Spare였던 `/dev/sdd1`이 다음과 같이 Active Disk로 변경되었다.

장애 발생 후 Hot Spare가 자동으로 장애 디스크를 대체하여 RAID 5가 다시 정상 상태가 되는 것을 확인하였다.

---

### 21. RAID 5 장애 허용 한계 확인

추가 디스크에 장애를 발생시킨 후 RAID 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md5
/dev/md5:
              State : clean, FAILED
     Active Devices : 2
    Working Devices : 2
     Failed Devices : 3
      Spare Devices : 0
```

장치 상태

```
/dev/sdc1  faulty
/dev/sde1  active sync
/dev/sdf1  active sync

/dev/sdb1  faulty
/dev/sdd1  faulty
```

RAID 5는 현재 RAID를 구성하는 디스크 중 1개의 장애까지 허용할 수 있지만 장애 허용 범위를 초과하면 `FAILED` 상태가 되는 것을 확인하였다.

Hot Spare는 장애 허용 개수를 증가시키는 것이 아니라 장애 발생 시 자동으로 교체 디스크를 투입하여 복구 시간을 줄이기 위한 기능이다.

---

### 22. RAID 6 구성

기존 RAID 5를 정리하고 `/dev/sdb1~sdf1` 5개의 디스크를 이용하여 RAID 6을 구성하였다.

```
[root@Server-A ~]# mdadm --create /dev/md6 --level=6 --raid-devices=5 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 /dev/sdf1
```

RAID 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md6
/dev/md6:
           Version : 1.2
        Raid Level : raid6
        Array Size : 314370048 (299.81 GiB 321.91 GB)
     Used Dev Size : 104790016 (99.94 GiB 107.30 GB)
      Raid Devices : 5
     Total Devices : 5
              State : clean, resyncing
     Active Devices : 5
    Working Devices : 5
     Failed Devices : 0
      Spare Devices : 0
             Layout : left-symmetric
         Chunk Size : 512K

     Resync Status : 1% complete
```

5개의 100G 디스크를 RAID 6으로 구성하여 약 300G의 사용 가능한 공간이 생성되는 것을 확인하였다.

RAID 6은 2개 디스크 분량의 용량을 Parity에 사용한다.

---

### 23. RAID 6 파일시스템 생성 및 /RAID6 마운트

ext4 파일시스템 생성

```
[root@Server-A ~]# mkfs.ext4 /dev/md6

Filesystem UUID: b324e18e-7a67-4bb9-a697-210d484f2e17
```

마운트 디렉터리 생성 및 마운트

```
[root@Server-A ~]# mkdir /RAID6
[root@Server-A ~]# mount /dev/md6 /RAID6
```

확인

```
[root@Server-A ~]# df -hT /RAID6
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md6       ext4  295G   28K  280G   1% /RAID6
```

`/dev/md6`이 ext4 파일시스템으로 생성되어 `/RAID6`에 정상적으로 마운트되는 것을 확인하였다.

---

### 24. RAID 6 UUID 기반 /etc/fstab 영구 마운트 설정

`/etc/fstab`에 다음 내용을 추가하였다.

```
UUID=b324e18e-7a67-4bb9-a697-210d484f2e17 /RAID6 ext4 defaults 0 0
```

설정 반영 및 재마운트

```
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /RAID6
[root@Server-A ~]# mount -a
```

확인

```
[root@Server-A ~]# df -hT /RAID6
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md6       ext4  295G   28K  280G   1% /RAID6
```

RAID 6 영구 마운트 설정이 정상적으로 적용되는 것을 확인하였다.

---

### 25. RAID 6 장애 테스트용 데이터 저장

`/etc`에서 `d`로 시작하는 파일과 디렉터리를 `/RAID6`에 복사하였다.

```
[root@Server-A ~]# cp -r /etc/d* /RAID6
```

확인

```
[root@Server-A ~]# ls -l /RAID6
합계 84
drwxr-xr-x. 4 root root  4096  9월 18 12:36 dbus-1
drwxr-xr-x. 4 root root  4096  9월 18 12:36 dconf
drwxr-xr-x. 2 root root  4096  9월 18 12:36 debuginfod
drwxr-xr-x. 2 root root  4096  9월 18 12:36 default
drwxr-xr-x. 2 root root  4096  9월 18 12:36 depmod.d
drwxr-x---. 3 root root  4096  9월 18 12:36 dhcp
drwxr-xr-x. 9 root root  4096  9월 18 12:36 dnf
-rw-r--r--. 1 root root 27839  9월 18 12:36 dnsmasq.conf
drwxr-xr-x. 2 root root  4096  9월 18 12:36 dnsmasq.d
-rw-r--r--. 1 root root   117  9월 18 12:36 dracut.conf
drwxr-xr-x. 2 root root  4096  9월 18 12:36 dracut.conf.d
```

장애 테스트에 사용할 데이터가 정상적으로 저장되는 것을 확인하였다.

---

### 26. RAID 6 첫 번째 디스크 장애 테스트

RAID 6 정상 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md6
/dev/md6:
              State : clean
     Active Devices : 5
    Working Devices : 5
     Failed Devices : 0
      Spare Devices : 0
```

`/dev/sdb1` 장애 발생

```
[root@Server-A ~]# mdadm --fail /dev/md6 /dev/sdb1
```

확인

```
[root@Server-A ~]# mdadm --detail /dev/md6
/dev/md6:
              State : clean, degraded
     Active Devices : 4
    Working Devices : 4
     Failed Devices : 1
      Spare Devices : 0
```

장애 디스크

```
/dev/sdb1  faulty
```

데이터 확인

```
[root@Server-A ~]# ls -l /RAID6
합계 84
drwxr-xr-x. 4 root root  4096  9월 18 12:36 dbus-1
drwxr-xr-x. 4 root root  4096  9월 18 12:36 dconf
drwxr-xr-x. 2 root root  4096  9월 18 12:36 debuginfod
drwxr-xr-x. 2 root root  4096  9월 18 12:36 default
drwxr-xr-x. 2 root root  4096  9월 18 12:36 depmod.d
drwxr-x---. 3 root root  4096  9월 18 12:36 dhcp
drwxr-xr-x. 9 root root  4096  9월 18 12:36 dnf
-rw-r--r--. 1 root root 27839  9월 18 12:36 dnsmasq.conf
drwxr-xr-x. 2 root root  4096  9월 18 12:36 dnsmasq.d
-rw-r--r--. 1 root root   117  9월 18 12:36 dracut.conf
drwxr-xr-x. 2 root root  4096  9월 18 12:36 dracut.conf.d
```

디스크 1개가 장애 상태여도 데이터가 정상적으로 유지되는 것을 확인하였다.

---

### 27. RAID 6 두 번째 디스크 장애 테스트

두 번째 디스크 `/dev/sdc1`에 장애 발생

```
[root@Server-A ~]# mdadm --fail /dev/md6 /dev/sdc1
```

처음 RAID Device가 아닌 Mount Point를 입력하였다.

```
[root@Server-A ~]# mdadm --detail /RAID6
mdadm: /RAID6 does not appear to be an md device
```

`/RAID6`는 Mount Point이고 RAID Device는 `/dev/md6`이므로 다음과 같이 다시 확인하였다.

```
[root@Server-A ~]# mdadm --detail /dev/md6
/dev/md6:
              State : clean, degraded
     Active Devices : 3
    Working Devices : 3
     Failed Devices : 2
      Spare Devices : 0
```

장치 상태

```
/dev/sdb1  faulty
/dev/sdc1  faulty

/dev/sdd1  active sync
/dev/sde1  active sync
/dev/sdf1  active sync
```

RAID 6은 디스크 2개가 동시에 장애 상태임에도 계속 동작하는 것을 확인하였다.

---

### 28. RAID 6 세 번째 디스크 장애 테스트

세 번째 디스크 `/dev/sdd1`에 장애 발생

```
[root@Server-A ~]# mdadm --fail /dev/md6 /dev/sdd1
```

RAID 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md6
/dev/md6:
              State : clean, FAILED
     Active Devices : 2
    Working Devices : 2
     Failed Devices : 3
      Spare Devices : 0
```

장치 상태

```
/dev/sdb1  faulty
/dev/sdc1  faulty
/dev/sdd1  faulty

/dev/sde1  active sync
/dev/sdf1  active sync
```

RAID 6은 디스크 2개 장애까지 허용하지만 3개의 디스크에 장애가 발생하면 `FAILED` 상태가 되는 것을 확인하였다.

---

### 29. RAID 상태 확인 명령어

현재 동작 중인 Software RAID의 간단한 상태 확인

```
[root@Server-A ~]# cat /proc/mdstat
```

RAID 상세 상태 확인

```
[root@Server-A ~]# mdadm --detail /dev/md6
```

주요 RAID 상태

`clean`

정상적으로 모든 RAID 구성원이 동작하는 상태

`clean, degraded`

RAID 구성 디스크가 부족하지만 RAID가 아직 정상적으로 서비스를 제공할 수 있는 상태

`clean, degraded, recovering`

장애 디스크를 교체하거나 새로운 디스크를 추가하여 Rebuild가 진행 중인 상태

`clean, FAILED`

RAID Level이 허용할 수 있는 장애 디스크 개수를 초과하여 RAID가 정상적으로 동작할 수 없는 상태

---

### 30. RAID Level별 장애 허용 확인

실습을 통해 RAID Level별 장애 허용 범위를 확인하였다.

Linear RAID

```
장애 허용 없음
```

RAID 0

```
장애 허용 없음
```

RAID 10

```
Mirroring을 이용하여 디스크 장애 발생 시에도 데이터 유지 가능
```

RAID 5

```
디스크 1개 장애까지 허용
```

RAID 5 + Hot Spare

```
디스크 장애 발생 시 Spare Disk가 자동으로 투입되어 Rebuild 수행
```

RAID 6

```
디스크 2개 장애까지 허용
```

RAID 5의 Hot Spare는 RAID 자체의 장애 허용 개수를 늘리는 기능이 아니라 장애 발생 시 복구를 빠르게 시작하기 위한 예비 디스크라는 것을 확인하였다.

---

## 실습 결과

* `fdisk`를 이용하여 Linux RAID용 파티션 생성
* `mdadm`을 이용하여 Linear RAID 구성
* RAID 0 Striping 구성 및 약 200G 저장 공간 확인
* RAID 10 Mirroring + Striping 구성 및 약 200G 저장 공간 확인
* RAID 5 구성 및 Parity를 제외한 약 300G 저장 공간 확인
* RAID 6 구성 및 Double Parity를 제외한 약 300G 저장 공간 확인
* `mkfs.ext4`를 이용하여 RAID 장치에 ext4 파일시스템 생성
* `/linear`, `/RAID0`, `/RAID10`, `/RAID5`, `/RAID6` 마운트 구성
* UUID를 이용하여 `/etc/fstab`에 영구 마운트 설정
* `systemctl daemon-reload`, `mount -a`를 이용한 영구 마운트 동작 확인
* `mdadm --fail`을 이용한 RAID 디스크 장애 상황 구성
* RAID 10 장애 디스크 교체 및 Rebuild 과정 확인
* RAID 5에서 디스크 1개 장애 발생 후 데이터 유지 확인
* RAID 5 장애 디스크 `remove` 및 `add`를 통한 Rebuild 확인
* RAID 5 Hot Spare 구성 및 장애 발생 시 Spare Disk 자동 투입 확인
* RAID 5 장애 허용 범위를 초과하면 `FAILED` 상태가 되는 것을 확인
* RAID 6에서 디스크 1개와 2개 장애 상태에서도 RAID 동작 확인
* RAID 6에서 세 번째 디스크 장애 발생 시 `FAILED` 상태가 되는 것을 확인
* `cat /proc/mdstat`, `mdadm --detail`을 이용하여 RAID 상태 확인
