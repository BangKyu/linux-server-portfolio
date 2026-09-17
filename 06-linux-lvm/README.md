# Linux LVM 구성 및 용량 확장 실습

## 실습 개요

Rocky Linux에서 LVM(Logical Volume Manager)을 이용하여 디스크를 논리적으로 관리하는 방법을 실습하였다.

빈 디스크를 PV(Physical Volume)로 생성하고 VG(Volume Group), LV(Logical Volume)를 구성한 후 ext4 파일시스템을 생성하여 마운트하였다.

`lvextend`, `resize2fs`, `lvextend -r`을 이용하여 LV와 파일시스템의 용량을 확장하고 `/etc/fstab`을 이용한 영구 마운트를 실습하였다.

---

## 실습 과정

### 1. 디스크 및 기존 LVM 상태 확인

현재 디스크 상태 확인

```bash
[root@Server-A ~]# lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
NAME    SIZE TYPE FSTYPE  MOUNTPOINTS
sda     100G disk
├─sda1    4G part swap    [SWAP]
└─sda2   96G part xfs     /
sdb     100G disk
├─sdb1   30G part xfs     /GIT
├─sdb2    1K part
├─sdb5   20G part ext4    /homeSK
├─sdb6   20G part ext4    /homeLG/user1
├─sdb7   20G part ext4    /homeLG/user2
└─sdb8   10G part ext4    /homeLG/user3
sdc     100G disk
sdd     100G disk
sde     100G disk
sr0    14.2G rom  iso9660
```

`/dev/sdc`가 파티션, 파일시스템, 마운트가 없는 100G 빈 디스크인 것을 확인하고 LVM 실습용으로 사용하였다.

기존 LVM 구성 확인

```bash
[root@Server-A ~]# pvs
[root@Server-A ~]# vgs
[root@Server-A ~]# lvs
```

기존 PV, VG, LV가 없는 상태에서 실습을 시작하였다.

---

### 2. PV 생성

`/dev/sdc`를 LVM에서 사용할 수 있도록 PV로 생성하였다.

```bash
[root@Server-A ~]# pvcreate /dev/sdc
  Physical volume "/dev/sdc" successfully created.
  Creating devices file /etc/lvm/devices/system.devices
```

PV 상태 확인

```bash
[root@Server-A ~]# pvs
  PV         VG Fmt  Attr PSize   PFree
  /dev/sdc      lvm2 ---  100.00g 100.00g
```

`/dev/sdc` 전체 100G가 PV로 생성되었으며 아직 VG에 할당되지 않아 전체 공간이 사용 가능한 상태임을 확인하였다.

---

### 3. VG 생성

PV 상태인 `/dev/sdc`를 이용하여 `vg01` Volume Group을 생성하였다.

```bash
[root@Server-A ~]# vgcreate vg01 /dev/sdc
  Volume group "vg01" successfully created
```

VG 상태 확인

```bash
[root@Server-A ~]# vgs
  VG   #PV #LV #SN Attr   VSize    VFree
  vg01   1   0   0 wz--n- <100.00g <100.00g
```

`vg01`이 하나의 PV로 구성되었으며 약 100G의 공간을 사용할 수 있는 것을 확인하였다.

---

### 4. LV 생성

`vg01`에서 30G 크기의 `lv01` Logical Volume을 생성하였다.

```bash
[root@Server-A ~]# lvcreate -L 30G -n lv01 vg01
  Logical volume "lv01" created.
```

`-L 30G`를 사용하여 LV 크기를 30G로 설정하였다.

`-n lv01`을 사용하여 Logical Volume의 이름을 `lv01`로 설정하였다.

LV 상태 확인

```bash
[root@Server-A ~]# lvs
  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv01 vg01 -wi-a----- 30.00g
```

`vg01` 내부에 30G 크기의 `lv01`이 생성된 것을 확인하였다.

LVM 구성 구조:

```text
/dev/sdc
   ↓
PV
   ↓
vg01
   ↓
lv01 30G
```

---

### 5. ext4 파일시스템 생성 및 마운트

`lv01`에 ext4 파일시스템을 생성하였다.

```bash
[root@Server-A ~]# mkfs.ext4 /dev/vg01/lv01
```

마운트 디렉터리를 생성하고 LV를 마운트하였다.

```bash
[root@Server-A ~]# mkdir -p /lvm
[root@Server-A ~]# mount /dev/vg01/lv01 /lvm
```

마운트 상태 확인

```bash
[root@Server-A ~]# df -hT /lvm
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg01-lv01 ext4   30G   24K   28G   1% /lvm
```

LVM 구조 확인

```bash
[root@Server-A ~]# lsblk /dev/sdc
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdc           8:32   0  100G  0 disk
└─vg01-lv01 253:0    0   30G  0 lvm  /lvm
```

`/dev/vg01/lv01`이 ext4 파일시스템으로 생성되어 `/lvm`에 정상적으로 마운트된 것을 확인하였다.

---

### 6. LV 용량 확장

기존 30G LV에 20G를 추가하여 50G로 확장하였다.

```bash
[root@Server-A ~]# lvextend -L +20G /dev/vg01/lv01
```

LV와 파일시스템의 크기를 각각 확인하였다.

```bash
[root@Server-A ~]# lvs
  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv01 vg01 -wi-ao---- 50.00g

[root@Server-A ~]# df -hT /lvm
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg01-lv01 ext4   30G   24K   28G   1% /lvm
```

LV는 50G로 확장되었지만 ext4 파일시스템은 기존 30G 크기를 유지하고 있는 것을 확인하였다.

```text
LV 크기 확장 ≠ 파일시스템 크기 확장
```

---

### 7. ext4 파일시스템 확장

확장된 LV 크기에 맞춰 ext4 파일시스템을 확장하였다.

```bash
[root@Server-A ~]# resize2fs /dev/vg01/lv01
```

확인

```bash
[root@Server-A ~]# df -hT /lvm
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg01-lv01 ext4   50G   24K   47G   1% /lvm

[root@Server-A ~]# lvs
  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv01 vg01 -wi-ao---- 50.00g
```

`resize2fs`를 사용하여 ext4 파일시스템도 약 50G로 확장된 것을 확인하였다.

---

### 8. LV와 파일시스템 동시 확장

`lvextend -r` 옵션을 이용하여 LV와 파일시스템을 동시에 확장하였다.

기존 50G에서 10G를 추가하여 60G로 확장

```bash
[root@Server-A ~]# lvextend -r -L +10G /dev/vg01/lv01
```

확인

```bash
[root@Server-A ~]# lvs
  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv01 vg01 -wi-ao---- 60.00g

[root@Server-A ~]# df -hT /lvm
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg01-lv01 ext4   59G   24K   57G   1% /lvm

[root@Server-A ~]# vgs
  VG   #PV #LV #SN Attr   VSize    VFree
  vg01   1   1   0 wz--n- <100.00g <40.00g
```

`lv01`은 60G로 확장되었으며 파일시스템도 함께 확장된 것을 확인하였다.

`vg01`에는 약 40G의 여유 공간이 남아 있는 것을 확인하였다.

```text
lvextend -L +20G
→ LV만 확장

resize2fs
→ ext4 파일시스템 확장

lvextend -r -L +10G
→ LV와 파일시스템을 동시에 확장
```

---

### 9. /etc/fstab 영구 마운트 설정

LV의 UUID를 확인하여 `/etc/fstab`에 등록하였다.

```bash
[root@Server-A ~]# blkid /dev/vg01/lv01
/dev/vg01/lv01: UUID="5f17682f-8d29-455e-b0dc-13ba2b99ef91" TYPE="ext4"
```

`/etc/fstab`에 다음 내용을 추가하였다.

```text
UUID=5f17682f-8d29-455e-b0dc-13ba2b99ef91 /lvm ext4 defaults 0 0
```

변경된 설정을 반영하고 현재 마운트를 해제하고 다시 마운트하였다.

```bash
[root@Server-A ~]# systemctl daemon-reload
[root@Server-A ~]# umount /lvm
[root@Server-A ~]# mount -a
```

최종 확인

```bash
[root@Server-A ~]# df -hT /lvm
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg01-lv01 ext4   59G   24K   57G   1% /lvm
```

`umount` 후 `mount -a`를 실행했을 때 `/lvm`이 다시 정상적으로 마운트되어 영구 마운트 설정이 정상적으로 적용된 것을 확인하였다.

---

## 트러블슈팅

### LV 생성 시 용량 단위 누락

처음 LV를 생성할 때 용량 단위를 지정하지 않았다.

```bash
[root@Server-A ~]# lvcreate -L 30 -n lv01 vg01
  Rounding up size to full physical extent 32.00 MiB
  Logical volume "lv01" created.

[root@Server-A ~]# lvs
  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv01 vg01 -wi-a----- 32.00m
```

원하는 크기는 30G였지만 실제로는 32MiB 크기의 LV가 생성되었다.

잘못 생성된 LV를 제거한 후 용량 단위 `G`를 명시하여 다시 생성하였다.

```bash
[root@Server-A ~]# lvremove /dev/vg01/lv01
[root@Server-A ~]# lvcreate -L 30G -n lv01 vg01
  Logical volume "lv01" created.
```

```bash
[root@Server-A ~]# lvs
  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv01 vg01 -wi-a----- 30.00g
```

LVM에서 용량을 지정할 때 `30G`, `500M`과 같이 단위를 명확하게 지정해야 함을 확인하였다.

---

## 실습 결과

- 빈 디스크 `/dev/sdc`를 LVM용 PV로 구성
- `vg01` Volume Group 생성
- `vg01`에서 `lv01` Logical Volume 30G 생성
- `lv01`에 ext4 파일시스템 생성 및 `/lvm` 마운트
- `pvs`, `vgs`, `lvs`를 이용하여 PV, VG, LV 상태 확인
- `lvextend`를 이용하여 LV를 30G에서 50G로 확장
- LV 확장만으로는 ext4 파일시스템 크기가 자동으로 증가하지 않는 것을 확인
- `resize2fs`를 이용하여 ext4 파일시스템을 50G로 확장
- `lvextend -r`을 이용하여 LV와 파일시스템을 60G까지 동시에 확장
- VG의 남은 여유 공간 약 40G 확인
- UUID와 `/etc/fstab`을 이용하여 `/lvm` 영구 마운트 설정
