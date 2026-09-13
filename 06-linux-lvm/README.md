# Linux LVM 실습

Rocky Linux 환경에서 LVM(Logical Volume Manager)을 구성하고  
PV → VG → LV 생성, 파일시스템 생성 및 마운트, 자동 마운트 설정,  
추가 디스크를 이용한 VG/LV/파일시스템 확장을 실습하였다.

---

## 1. 실습 환경

기존 시스템 디스크와 스토리지 실습용 디스크는 유지한 상태에서  
LVM 실습용 10GB 디스크 2개를 추가하였다.

```bash
lsblk
```

확인한 LVM 실습용 디스크:

```text
sdc    10G
sdd    10G
```

기존 `/dev/sda`, `/dev/sdb`는 다른 용도로 사용 중이므로  
LVM 실습에서는 사용하지 않았다.

---

## 2. LVM 파티션 생성

`/dev/sdc`, `/dev/sdd`에 각각 전체 용량을 사용하는 파티션을 생성하였다.

```bash
fdisk /dev/sdc
fdisk /dev/sdd
```

DOS/MBR 파티션 테이블을 사용하였으며  
파티션 타입은 `Linux LVM(8e)`으로 설정하였다.

확인:

```bash
fdisk -l /dev/sdc
fdisk -l /dev/sdd
```

실제 결과:

```text
Disk /dev/sdc: 10 GiB, 10737418240 bytes, 20971520 sectors
Disklabel type: dos

Device     Boot Start      End  Sectors Size Id Type
/dev/sdc1        2048 20971519 20969472  10G 8e Linux LVM
```

```text
Disk /dev/sdd: 10 GiB, 10737418240 bytes, 20971520 sectors
Disklabel type: dos

Device     Boot Start      End  Sectors Size Id Type
/dev/sdd1        2048 20971519 20969472  10G 8e Linux LVM
```

---

## 3. PV(Physical Volume) 생성

LVM에서 사용할 수 있도록 두 파티션을 PV로 초기화하였다.

```bash
pvcreate /dev/sdc1 /dev/sdd1
```

실행 결과:

```text
Physical volume "/dev/sdc1" successfully created.
Physical volume "/dev/sdd1" successfully created.
Creating devices file /etc/lvm/devices/system.devices
```

PV 확인:

```bash
pvs
```

```text
PV         VG Fmt  Attr PSize   PFree
/dev/sdc1     lvm2 ---  <10.00g <10.00g
/dev/sdd1     lvm2 ---  <10.00g <10.00g
```

PV를 생성했지만 아직 VG에 포함되지 않았기 때문에  
두 PV의 전체 공간이 사용 가능한 상태이다.

---

## 4. VG(Volume Group) 생성

두 PV를 하나의 Volume Group으로 묶었다.

VG 이름은 `SOLLVM`으로 설정하였다.

```bash
vgcreate SOLLVM /dev/sdc1 /dev/sdd1
```

실행 결과:

```text
Volume group "SOLLVM" successfully created
```

확인:

```bash
vgs
pvs
```

```text
VG     #PV #LV #SN Attr   VSize  VFree
SOLLVM   2   0   0 wz--n- 19.99g 19.99g
```

```text
PV         VG     Fmt  Attr PSize   PFree
/dev/sdc1  SOLLVM lvm2 a--  <10.00g <10.00g
/dev/sdd1  SOLLVM lvm2 a--  <10.00g <10.00g
```

약 10GB의 PV 2개를 묶어  
약 20GB 크기의 `SOLLVM` VG를 구성하였다.

---

## 5. LV(Logical Volume) 생성

`SOLLVM`에서 3개의 LV를 생성하였다.

```bash
lvcreate -L 8G -n 8G_LV1 SOLLVM
lvcreate -L 6G -n 6G_LV2 SOLLVM
lvcreate -l 100%FREE -n 6G_LV3 SOLLVM
```

확인:

```bash
lvs
vgs
```

실제 결과:

```text
LV     VG     Attr       LSize
6G_LV2 SOLLVM -wi-a----- 6.00g
6G_LV3 SOLLVM -wi-a----- 5.99g
8G_LV1 SOLLVM -wi-a----- 8.00g
```

```text
VG     #PV #LV #SN Attr   VSize  VFree
SOLLVM   2   3   0 wz--n- 19.99g    0
```

구성 결과:

```text
SOLLVM 약 20GB
├── 8G_LV1 : 8GB
├── 6G_LV2 : 6GB
└── 6G_LV3 : 약 6GB
```

VG의 모든 여유 공간을 LV에 할당하여 `VFree`가 0이 되었다.

---

## 6. ext4 파일시스템 생성

생성한 LV에 ext4 파일시스템을 생성하였다.

```bash
mkfs.ext4 /dev/SOLLVM/8G_LV1
mkfs.ext4 /dev/SOLLVM/6G_LV2
mkfs.ext4 /dev/SOLLVM/6G_LV3
```

---

## 7. 마운트 디렉터리 생성 및 마운트

마운트할 디렉터리를 생성하였다.

```bash
mkdir -p /CU
mkdir -p /GS
mkdir -p /LG
```

각 LV를 마운트하였다.

```bash
mount /dev/SOLLVM/8G_LV1 /CU
mount /dev/SOLLVM/6G_LV2 /GS
mount /dev/SOLLVM/6G_LV3 /LG
```

마운트 결과:

```text
8G_LV1 → /CU
6G_LV2 → /GS
6G_LV3 → /LG
```

확인:

```bash
df -hT /CU /GS /LG
```

```text
Filesystem                Type  Size  Used Avail Use% Mounted on
/dev/mapper/SOLLVM-8G_LV1 ext4  7.8G   24K  7.4G   1% /CU
/dev/mapper/SOLLVM-6G_LV2 ext4  5.9G   24K  5.6G   1% /GS
/dev/mapper/SOLLVM-6G_LV3 ext4  5.9G   24K  5.5G   1% /LG
```

---

## 8. `/etc/fstab` 자동 마운트 설정

재부팅 후에도 LV가 자동으로 마운트되도록 UUID를 사용하여  
`/etc/fstab`에 등록하였다.

먼저 기존 설정을 백업하였다.

```bash
cp -p /etc/fstab /etc/fstab.lvm.bak
```

LV UUID 확인:

```bash
lsblk -f
```

확인한 UUID:

```text
8G_LV1
dd84dfea-90d4-4562-be52-528800201931

6G_LV2
2fe5a703-5893-482e-a3ba-4c644cf8bcb4

6G_LV3
19b567c0-cffa-4774-ba2a-5525344e0e5e
```

`/etc/fstab`에 다음 내용을 추가하였다.

```text
UUID=dd84dfea-90d4-4562-be52-528800201931  /CU  ext4  defaults  0 2
UUID=2fe5a703-5893-482e-a3ba-4c644cf8bcb4  /GS  ext4  defaults  0 2
UUID=19b567c0-cffa-4774-ba2a-5525344e0e5e  /LG  ext4  defaults  0 2
```

설정 반영 및 오류 확인:

```bash
systemctl daemon-reload
mount -a
```

마운트 상태 확인:

```bash
findmnt /CU
findmnt /GS
findmnt /LG
```

실제 결과:

```text
TARGET SOURCE                    FSTYPE OPTIONS
/CU    /dev/mapper/SOLLVM-8G_LV1 ext4   rw,relatime,seclabel
```

```text
TARGET SOURCE                    FSTYPE OPTIONS
/GS    /dev/mapper/SOLLVM-6G_LV2 ext4   rw,relatime,seclabel
```

```text
TARGET SOURCE                    FSTYPE OPTIONS
/LG    /dev/mapper/SOLLVM-6G_LV3 ext4   rw,relatime,seclabel
```

---

## 9. 재부팅 후 자동 마운트 확인

시스템을 재부팅하였다.

```bash
reboot
```

재부팅 후 별도의 `mount` 또는 `mount -a` 명령을 실행하지 않고 확인하였다.

```bash
findmnt /CU
findmnt /GS
findmnt /LG
```

실제 결과:

```text
TARGET SOURCE                    FSTYPE OPTIONS
/CU    /dev/mapper/SOLLVM-8G_LV1 ext4   rw,relatime,seclabel
```

```text
TARGET SOURCE                    FSTYPE OPTIONS
/GS    /dev/mapper/SOLLVM-6G_LV2 ext4   rw,relatime,seclabel
```

```text
TARGET SOURCE                    FSTYPE OPTIONS
/LG    /dev/mapper/SOLLVM-6G_LV3 ext4   rw,relatime,seclabel
```

재부팅 이후에도 자동으로 마운트되는 것을 확인하였다.

---

## 10. LVM 확장을 위한 새 디스크 추가

기존 VG의 여유 공간이 없기 때문에  
VMware에서 새로운 10GB 디스크를 추가하였다.

```bash
lsblk
```

새 디스크 확인:

```text
sde    10G
```

새 디스크에 LVM용 파티션 `/dev/sde1`을 생성하였다.

```bash
fdisk /dev/sde
```

DOS/MBR 환경에서 파티션 타입을 `Linux LVM(8e)`으로 설정하였다.

---

## 11. 새로운 PV 생성 및 VG 확장

새로운 파티션을 PV로 생성하였다.

```bash
pvcreate /dev/sde1
```

기존 `SOLLVM` VG에 `/dev/sde1`을 추가하였다.

```bash
vgextend SOLLVM /dev/sde1
```

확인:

```bash
pvs
vgs
```

실제 결과:

```text
PV         VG     Fmt  Attr PSize   PFree
/dev/sdc1  SOLLVM lvm2 a--  <10.00g      0
/dev/sdd1  SOLLVM lvm2 a--  <10.00g      0
/dev/sde1  SOLLVM lvm2 a--  <10.00g <10.00g
```

```text
VG     #PV #LV #SN Attr   VSize   VFree
SOLLVM   3   3   0 wz--n- <29.99g <10.00g
```

VG가 약 20GB에서 약 30GB로 확장되었으며  
약 10GB의 새로운 여유 공간이 확보되었다.

---

## 12. 기존 LV 용량 확장

기존 `8G_LV1`을 1GB 확장하였다.

```bash
lvextend -L +1G /dev/SOLLVM/8G_LV1
```

LV 확인:

```bash
lvs
```

확장 결과:

```text
LV     VG     Attr       LSize
6G_LV2 SOLLVM -wi-ao---- 6.00g
6G_LV3 SOLLVM -wi-ao---- 5.99g
8G_LV1 SOLLVM -wi-ao---- 9.00g
```

LV 이름은 `8G_LV1`이지만 실제 크기는 9GB로 증가하였다.

LV 이름은 단순한 식별 이름이므로  
LV의 실제 크기가 변경되어도 자동으로 변경되지 않는다.

---

## 13. ext4 파일시스템 확장

`lvextend`는 LV의 공간을 확장하지만  
파일시스템의 크기까지 자동으로 확장한 것은 아니므로  
ext4 파일시스템도 확장하였다.

```bash
resize2fs /dev/SOLLVM/8G_LV1
```

확장 후 확인:

```bash
df -hT /CU
```

실제 결과:

```text
Filesystem                Type  Size  Used Avail Use% Mounted on
/dev/mapper/SOLLVM-8G_LV1 ext4  8.8G   24K  8.4G   1% /CU
```

`/CU`에서 사용하는 ext4 파일시스템까지 정상적으로 확장된 것을 확인하였다.

---

## 14. 최종 LVM 상태 확인

```bash
pvs
vgs
lvs
```

최종 결과:

```text
PV         VG     Fmt  Attr PSize   PFree
/dev/sdc1  SOLLVM lvm2 a--  <10.00g     0
/dev/sdd1  SOLLVM lvm2 a--  <10.00g     0
/dev/sde1  SOLLVM lvm2 a--  <10.00g <9.00g
```

```text
VG     #PV #LV #SN Attr   VSize   VFree
SOLLVM   3   3   0 wz--n- <29.99g <9.00g
```

```text
LV     VG     Attr       LSize
6G_LV2 SOLLVM -wi-ao---- 6.00g
6G_LV3 SOLLVM -wi-ao---- 5.99g
8G_LV1 SOLLVM -wi-ao---- 9.00g
```

새로운 10GB PV를 VG에 추가한 뒤  
그중 약 1GB를 기존 LV 확장에 사용하여 약 9GB가 남아 있다.

---

## 15. 최종 파일시스템 및 마운트 상태

```bash
df -hT /CU /GS /LG
```

실제 결과:

```text
Filesystem                Type  Size  Used Avail Use% Mounted on
/dev/mapper/SOLLVM-8G_LV1 ext4  8.8G   24K  8.4G   1% /CU
/dev/mapper/SOLLVM-6G_LV2 ext4  5.9G   24K  5.6G   1% /GS
/dev/mapper/SOLLVM-6G_LV3 ext4  5.9G   24K  5.5G   1% /LG
```

---

## 16. 최종 구조

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

SOLLVM 남은 공간: 약 9GB
```

---

## 17. 실습을 통해 확인한 내용

- 디스크 파티션을 LVM PV로 구성
- 여러 PV를 하나의 VG로 통합
- VG 공간을 이용하여 여러 LV 생성
- LV에 ext4 파일시스템 생성
- LV를 일반 디스크 파티션처럼 디렉터리에 마운트
- UUID를 이용한 `/etc/fstab` 자동 마운트 구성
- 재부팅 후 자동 마운트 동작 확인
- 새로운 디스크를 PV로 추가
- `vgextend`를 이용한 기존 VG 용량 확장
- `lvextend`를 이용한 기존 LV 용량 확장
- `resize2fs`를 이용한 ext4 파일시스템 확장
- 하나의 LV가 여러 PV의 공간을 사용할 수 있음을 확인

---

## 18. 핵심 명령어

| 명령어 | 용도 |
|---|---|
| `pvcreate` | PV 생성 |
| `pvs` | PV 상태 요약 확인 |
| `pvdisplay` | PV 상세 정보 확인 |
| `vgcreate` | VG 생성 |
| `vgextend` | 기존 VG에 PV 추가 |
| `vgs` | VG 상태 요약 확인 |
| `lvcreate` | LV 생성 |
| `lvextend` | LV 용량 확장 |
| `lvs` | LV 상태 요약 확인 |
| `mkfs.ext4` | ext4 파일시스템 생성 |
| `resize2fs` | ext4 파일시스템 크기 조정 |
| `lsblk -f` | 디스크/LVM/파일시스템 구조 확인 |
| `findmnt` | 마운트 상태 확인 |
| `df -hT` | 파일시스템 용량과 타입 확인 |
| `mount -a` | `/etc/fstab` 설정 테스트 |

---

## 정리

일반 파티션만 사용하는 방식과 달리 LVM은  
여러 물리 저장장치의 공간을 하나의 VG로 묶고 필요한 크기의 LV를 생성하여 사용할 수 있다.

이번 실습에서는 처음 약 20GB의 VG를 생성한 후  
추가 10GB 디스크를 기존 VG에 편입하여 약 30GB로 확장하였다.

또한 기존 `8G_LV1`을 삭제하거나 새로 생성하지 않고  
8GB에서 9GB로 확장한 뒤 ext4 파일시스템까지 확장하였다.

이를 통해 LVM의 PV → VG → LV 구조와  
스토리지 용량 확장 과정을 실제 환경에서 확인하였다.
