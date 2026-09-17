374182400 bytes, 209715200 sectors
Disk model: VMware Virtual S
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

`/dev/sdb`가 100GiB 크기의 빈 디스크인 것을 확인하였다.

---

### 3. Primary Partition 생성

`fdisk`를 이용하여 `/dev/sdb1` 30G 파티션 생성

```bash
[root@Server-A ~]# fdisk /dev/sdb
```

생성 후 파티션 정보 확인

```text
Disk /dev/sdb: 100 GiB, 107374182400 bytes, 209715200 sectors
Disklabel type: dos
Disk identifier: 0xff9719bd

Device     Boot Start      End  Sectors Size Id Type
/dev/sdb1        2048 62916607 62914560  30G 83 Linux
```

파티션 정보를 저장한 후 확인

```bash
[root@Server-A ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  100G  0 disk
├─sda1   8:1    0    4G  0 part [SWAP]
└─sda2   8:2    0   96G  0 part /
sdb      8:16   0  100G  0 disk
└─sdb1   8:17   0   30G  0 part
sdc      8:32   0  100G  0 disk
sdd      8:48   0  100G  0 disk
sde      8:64   0  100G  0 disk
sr0     11:0    1 14.2G  0 rom
```

---

### 4. XFS 파일시스템 생성 및 /GIT 마운트

`/dev/sdb1`에 XFS 파일시스템 생성

```bash
[root@Server-A ~]# mkfs.xfs /dev/sdb1
```

마운트 디렉터리 생성 후 마운트

```bash
[root@Server-A ~]# mkdir -p /GIT
[root@Server-A ~]# mount /dev/sdb1 /GIT
```

확인

```bash
[root@Server-A ~]# df -hT /GIT
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      xfs    30G  247M   30G   1% /GIT
```

`/dev/sdb1`이 XFS 파일시스템으로 생성되어 `/GIT`에 정상적으로 마운트된 것을 확인하였다.

---

### 5. UUID를 이용한 /GIT 영구 마운트

`/dev/sdb1`의 UUID 확인

```bash
[root@Server-A ~]# lsblk -f /dev/sdb
NAME FSTYPE FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdb
└─sdb1
     xfs                      3e51ac3a-fd57-4a6e-aada-31c4eeae01f3   29.7G     1% /GIT
```

`/etc/fstab`에 다음 내용을 추가하였다.

```text
UUID=3e51ac3a-fd57-4a6e-aada-31c4eeae01f3 /GIT xfs defaults 0 0
```

마운트를 해제한 후 `/etc/fstab` 설정을 이용하여 다시 마운트

```bash
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /GIT
[root@Server-A ~]# mount -a
```

확인

```bash
[root@Server-A ~]# df -hT /GIT
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      xfs    30G  247M   30G   1% /GIT
```

UUID 기반 영구 마운트 설정이 정상적으로 적용되는 것을 확인하였다.

---

### 6. Extended 및 Logical Partition 생성

MBR 구조에서 남은 공간을 Extended Partition으로 생성하고 내부에 Logical Partition을 생성하였다.

첫 번째 Logical Partition 생성 후 확인

```text
Device     Boot    Start       End   Sectors Size Id Type
/dev/sdb1           2048  62916607  62914560  30G 83 Linux
/dev/sdb2       62916608 209715199 146798592  70G  5 Extended
/dev/sdb5       62918656 104861695  41943040  20G 83 Linux
```

- `/dev/sdb1` → Primary Partition
- `/dev/sdb2` → Extended Partition
- `/dev/sdb5` → Logical Partition

MBR에서 Logical Partition 번호는 `5`부터 시작하는 것을 확인하였다.

---

### 7. /dev/sdb5 ext4 파일시스템 및 /homeSK 마운트

`/dev/sdb5`에 ext4 파일시스템 생성

```bash
[root@Server-A ~]# mkfs.ext4 /dev/sdb5
[root@Server-A ~]# mkdir -p /homeSK
[root@Server-A ~]# mount /dev/sdb5 /homeSK
```

확인

```bash
[root@Server-A ~]# df -hT /homeSK
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb5      ext4   20G   24K   19G   1% /homeSK
```

파일시스템 및 UUID 확인

```bash
[root@Server-A ~]# lsblk -f /dev/sdb
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdb
├─sdb1 xfs                3e51ac3a-fd57-4a6e-aada-31c4eeae01f3   29.7G     1% /GIT
├─sdb2
└─sdb5 ext4   1.0         801720ee-1dc2-4509-8d7d-b4c5b231297a   18.5G     0% /homeSK
```

`/etc/fstab`에 추가

```text
UUID=801720ee-1dc2-4509-8d7d-b4c5b231297a /homeSK ext4 defaults 0 0
```

---

### 8. 추가 Logical Partition 생성

Extended Partition 내부의 남은 공간에 Logical Partition 3개를 추가 생성하였다.

```bash
[root@Server-A ~]# fdisk -l /dev/sdb
Device     Boot     Start       End   Sectors Size Id Type
/dev/sdb1            2048  62916607  62914560  30G 83 Linux
/dev/sdb2        62916608 209715199 146798592  70G  5 Extended
/dev/sdb5        62918656 104861695  41943040  20G 83 Linux
/dev/sdb6       104863744 146806783  41943040  20G 83 Linux
/dev/sdb7       146808832 188751871  41943040  20G 83 Linux
/dev/sdb8       188753920 209715199  20961280  10G 83 Linux
```

최종 파티션 구조:

```text
/dev/sdb 100G
├─ /dev/sdb1  30G  Primary
└─ /dev/sdb2  70G  Extended
   ├─ /dev/sdb5  20G  Logical
   ├─ /dev/sdb6  20G  Logical
   ├─ /dev/sdb7  20G  Logical
   └─ /dev/sdb8  10G  Logical
```

---

### 9. 추가 Logical Partition 파일시스템 생성

`/dev/sdb6`, `/dev/sdb7`, `/dev/sdb8`에 ext4 파일시스템 생성

```bash
[root@Server-A ~]# mkfs.ext4 /dev/sdb6
[root@Server-A ~]# mkfs.ext4 /dev/sdb7
[root@Server-A ~]# mkfs.ext4 /dev/sdb8
```

마운트 디렉터리 생성

```bash
[root@Server-A ~]# mkdir -p /homeLG/user1
[root@Server-A ~]# mkdir -p /homeLG/user2
[root@Server-A ~]# mkdir -p /homeLG/user3
```

각 파티션 마운트

```bash
[root@Server-A ~]# mount /dev/sdb6 /homeLG/user1
[root@Server-A ~]# mount /dev/sdb7 /homeLG/user2
[root@Server-A ~]# mount /dev/sdb8 /homeLG/user3
```

확인

```bash
[root@Server-A ~]# df -hT /homeLG/user1 /homeLG/user2 /homeLG/user3
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb6      ext4   20G   24K   19G   1% /homeLG/user1
/dev/sdb7      ext4   20G   24K   19G   1% /homeLG/user2
/dev/sdb8      ext4  9.8G   24K  9.3G   1% /homeLG/user3
```

---

### 10. UUID 기반 영구 마운트 설정

Logical Partition의 UUID 확인

```text
[root@Server-A ~]# lsblk -f /dev/sdb
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdb
├─sdb1 xfs                3e51ac3a-fd57-4a6e-aada-31c4eeae01f3   29.7G     1% /GIT
├─sdb2
├─sdb5 ext4   1.0         801720ee-1dc2-4509-8d7d-b4c5b231297a   18.5G     0% /homeSK
├─sdb6 ext4   1.0         7f7a0751-cbbf-427a-8439-661d6f13e55e   18.5G     0% /homeLG/user1
├─sdb7 ext4   1.0         c1898f8a-e125-48e5-b4a8-6ef7a82a79da   18.5G     0% /homeLG/user2
└─sdb8 ext4   1.0         c1f7bd62-050e-4fe3-8410-9cc732a7ca68    9.2G     0% /homeLG/user3

```

`/etc/fstab`에 등록

```text
UUID=7f7a0751-cbbf-427a-8439-661d6f13e55e /homeLG/user1 ext4 defaults 0 0
UUID=c1898f8a-e125-48e5-b4a8-6ef7a82a79da /homeLG/user2 ext4 defaults 0 0
UUID=c1f7bd62-050e-4fe3-8410-9cc732a7ca68 /homeLG/user3 ext4 defaults 0 0
```

설정 반영 및 재마운트 테스트

```bash
[root@Server-A ~]# systemctl daemon-reload

[root@Server-A ~]# umount /homeLG/user1
[root@Server-A ~]# umount /homeLG/user2
[root@Server-A ~]# umount /homeLG/user3

[root@Server-A ~]# mount -a
```

---

### 11. 최종 마운트 상태 확인

```bash
[root@Server-A ~]# df -hT /GIT /homeSK /homeLG/user1 /homeLG/user2 /homeLG/user3
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      xfs    30G  247M   30G   1% /GIT
/dev/sdb5      ext4   20G   24K   19G   1% /homeSK
/dev/sdb6      ext4   20G   24K   19G   1% /homeLG/user1
/dev/sdb7      ext4   20G   24K   19G   1% /homeLG/user2
/dev/sdb8      ext4  9.8G   24K  9.3G   1% /homeLG/user3
```

```bash
[root@Server-A ~]# lsblk -f /dev/sdb
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdb
├─sdb1 xfs                3e51ac3a-fd57-4a6e-aada-31c4eeae01f3   29.7G     1% /GIT
├─sdb2
├─sdb5 ext4   1.0         801720ee-1dc2-4509-8d7d-b4c5b231297a   18.5G     0% /homeSK
├─sdb6 ext4   1.0         7f7a0751-cbbf-427a-8439-661d6f13e55e   18.5G     0% /homeLG/user1
├─sdb7 ext4   1.0         c1898f8a-e125-48e5-b4a8-6ef7a82a79da   18.5G     0% /homeLG/user2
└─sdb8 ext4   1.0         c1f7bd62-050e-4fe3-8410-9cc732a7ca68    9.2G     0% /homeLG/user3
```

모든 파티션이 `/etc/fstab` 설정에 따라 정상적으로 다시 마운트되는 것을 확인하였다.

---

## 실습 결과

- 추가 디스크 `/dev/sdb`의 상태와 용량 확인
- MBR(DOS) 파티션 테이블에서 Primary, Extended, Logical Partition 생성
- Primary Partition `/dev/sdb1` 30G 생성
- Extended Partition `/dev/sdb2` 70G 생성
- Logical Partition `/dev/sdb5~8` 생성
- XFS와 ext4 파일시스템 생성 및 차이 확인
- `mount`, `umount`를 이용한 파일시스템 마운트 및 해제
- `/GIT`, `/homeSK`, `/homeLG/user1~3` 마운트 구성
- `lsblk`, `df`, `fdisk`를 이용한 디스크 및 마운트 상태 확인
- UUID를 이용하여 `/etc/fstab`에 영구 마운트 설정
- `mount -a`를 이용하여 `/etc/fstab` 설정 정상 동작 확인
