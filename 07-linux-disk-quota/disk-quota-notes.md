# Linux Disk Quota 정리

## 1. Disk Quota란?

Disk Quota는 특정 사용자 또는 그룹이 파일시스템의 디스크 공간을 과도하게 사용하지 못하도록 제한하는 기능이다.

여러 사용자가 하나의 서버를 사용하는 환경에서 한 사용자가 디스크 공간을 모두 사용하면 다른 사용자가 정상적으로 파일을 생성하지 못할 수 있다.

Quota를 사용하면 사용자 또는 그룹마다 사용할 수 있는 디스크 용량과 파일 개수를 제한할 수 있다.

```text
파일시스템
│
├── user1 → 최대 10GB
├── user2 → 최대 20GB
└── user3 → 최대 30GB
```

주요 목적:

- 특정 사용자의 과도한 디스크 사용 방지
- 여러 사용자에게 디스크 공간을 분배
- 그룹 단위 저장 공간 제한
- 파일 개수 제한
- 다중 사용자 서버의 저장 공간 관리

---

# 2. Quota의 제한 대상

Quota에서는 크게 두 가지를 제한할 수 있다.

```text
Quota
├── Block Quota → 디스크 용량 제한
└── Inode Quota → 파일/디렉터리 개수 제한
```

---

## 2-1. Block Quota

Block Quota는 사용자가 사용할 수 있는 디스크 공간을 제한한다.

예:

```text
Soft Limit = 10 MiB
Hard Limit = 15 MiB
```

사용자가 파일을 계속 생성하여 15MiB에 도달하면 추가 데이터를 기록할 수 없게 된다.

```text
0 ─────────── 10MiB ─────────── 15MiB
                ↑                  ↑
           Soft Limit          Hard Limit
```

`edquota`에서 `blocks`는 현재 사용 중인 디스크 공간을 나타낸다.

예:

```text
Filesystem    blocks    soft    hard
/dev/sdc1         28    10240   15360
```

의미:

```text
blocks = 현재 사용량
soft   = 경고 기준
hard   = 절대 최대값
```

---

## 2-2. Inode Quota

Inode Quota는 사용자가 생성할 수 있는 파일 또는 디렉터리의 개수를 제한한다.

예:

```text
Inode Soft Limit = 20
Inode Hard Limit = 30
```

사용자가 총 30개의 inode를 사용하면 더 이상 새로운 파일 또는 디렉터리를 생성할 수 없다.

```text
현재 inode 사용량
       ↓
20개 초과
       ↓
Soft Limit 초과
       ↓
Grace Period
       ↓
30개 도달
       ↓
Hard Limit
       ↓
추가 파일 생성 차단
```

작은 파일을 매우 많이 생성하는 사용자를 제한할 때 사용할 수 있다.

---

# 3. Block과 Inode 차이

| 구분 | 의미 | 제한 대상 |
|---|---|---|
| Block | 디스크에서 사용 중인 공간 | 용량 |
| Inode | 파일/디렉터리를 관리하기 위한 정보 | 파일/디렉터리 개수 |

예를 들어:

```text
1GB 파일 1개
```

는 용량은 많이 사용하지만 inode는 적게 사용한다.

반대로:

```text
1KB 파일 100,000개
```

는 각각의 파일 크기는 작지만 많은 inode를 사용한다.

따라서 서버에서는 필요에 따라 Block Quota와 Inode Quota를 각각 설정할 수 있다.

---

# 4. Soft Limit과 Hard Limit

Quota에는 일반적으로 Soft Limit과 Hard Limit이 존재한다.

---

## 4-1. Soft Limit

Soft Limit은 사용량의 경고 기준이다.

```text
Soft Limit 초과
        ↓
즉시 차단하지 않음
        ↓
Grace Period 시작
        ↓
일정 기간 동안 추가 사용 가능
```

예:

```text
Soft Limit = 10 MiB
```

사용량이 10MiB를 초과했다고 바로 파일 생성이 차단되는 것은 아니다.

Grace Period 동안은 Hard Limit까지 추가로 사용할 수 있다.

Soft Limit을 `0`으로 설정하면 해당 제한을 사용하지 않는다.

---

## 4-2. Hard Limit

Hard Limit은 절대 초과할 수 없는 최대값이다.

```text
Hard Limit 도달
        ↓
추가 쓰기 또는 파일 생성 차단
```

Hard Limit에는 Soft Limit과 같은 유예 기간이 적용되지 않는다.

예:

```text
Soft = 10 MiB
Hard = 15 MiB
```

라면:

```text
0 ~ 10MiB
└── 정상 사용

10 ~ 15MiB
└── Soft Limit 초과
    Grace Period 적용

15MiB
└── Hard Limit 도달
    추가 쓰기 차단
```

---

# 5. Grace Period

Grace Period는 Soft Limit을 초과한 후 Hard Limit과 같은 제한이 적용되기 전까지 제공되는 유예 시간이다.

이번 실습에서는 다음과 같이 확인되었다.

```text
7days
```

동작:

```text
Soft Limit 초과
        ↓
Grace Period 시작
        ↓
유예 기간 동안 추가 사용 가능
        ↓
유예 기간 종료
        ↓
Soft Limit을 계속 초과하고 있다면 제한 적용
```

Hard Limit은 Grace Period와 관계없이 즉시 적용된다.

---

# 6. User Quota

User Quota는 사용자 단위로 디스크 사용량을 제한한다.

마운트 옵션:

```text
usrquota
```

예:

```text
/githome
│
├── quser1
├── quser2
└── quser3
```

각 사용자에게 서로 다른 제한을 설정할 수 있다.

예:

```text
quser1
├── Block Soft = 10 MiB
└── Block Hard = 15 MiB

quser2
├── Inode Soft = 20
└── Inode Hard = 30
```

사용자 Quota 설정:

```bash
edquota -u 사용자명
```

예:

```bash
edquota -u quser1
```

확인:

```bash
quota -u quser1
```

---

# 7. Group Quota

Group Quota는 사용자 한 명이 아니라 그룹 단위로 디스크 사용량을 제한한다.

마운트 옵션:

```text
grpquota
```

예:

```text
winteam
├── guser1
├── guser2
└── guser3
```

그룹 전체 제한이 40MiB라면 여러 사용자가 사용한 그룹 사용량이 제한값에 도달했을 때 추가 쓰기가 제한된다.

```text
guser1 → 15MiB
guser2 → 15MiB
guser3 → 10MiB
────────────────
winteam → 40MiB
```

Group Quota 설정:

```bash
edquota -g 그룹명
```

예:

```bash
edquota -g winteam
```

확인:

```bash
quota -g winteam
```

---

# 8. User Quota와 Group Quota 비교

| 구분 | User Quota | Group Quota |
|---|---|---|
| 기준 | 사용자 | 그룹 |
| fstab 옵션 | `usrquota` | `grpquota` |
| 설정 | `edquota -u USER` | `edquota -g GROUP` |
| 확인 | `quota -u USER` | `quota -g GROUP` |
| 용도 | 개인 사용량 제한 | 팀/그룹 사용량 제한 |

---

# 9. /etc/fstab Quota 설정

Quota 기능을 사용할 파일시스템에는 관련 마운트 옵션을 지정할 수 있다.

User Quota:

```text
defaults,usrquota
```

User + Group Quota:

```text
defaults,usrquota,grpquota
```

예:

```text
UUID=...  /githome  ext4  defaults,usrquota           0 2
UUID=...  /winhome  ext4  defaults,usrquota,grpquota  0 2
```

`/etc/fstab` 수정 후 systemd 설정을 갱신한다.

```bash
systemctl daemon-reload
```

기존에 마운트된 파일시스템이라면 필요에 따라 다시 마운트하거나 리마운트하여 변경된 옵션을 적용한다.

예:

```bash
mount -o remount /githome
mount -o remount /winhome
```

현재 마운트 옵션 확인:

```bash
findmnt /githome
findmnt /winhome
```

또는:

```bash
mount | grep githome
mount | grep winhome
```

---

# 10. ext4 Quota Feature

Rocky Linux 9의 ext4에서는 파일시스템 자체의 Quota Feature를 사용할 수 있다.

Quota Feature 확인:

```bash
tune2fs -l /dev/sdc1
```

필요한 부분만 확인:

```bash
tune2fs -l /dev/sdc1 | grep -E 'Filesystem features|User quota inode|Group quota inode'
```

Quota Feature가 활성화되어 있으면 다음과 같이 `quota`가 포함된다.

```text
Filesystem features: ... quota ...
User quota inode:  3
Group quota inode: 4
```

---

## ext4 Quota Feature 활성화

파일시스템을 먼저 마운트 해제한다.

```bash
umount /githome
```

Quota Feature 활성화:

```bash
tune2fs -O quota /dev/sdc1
```

확인:

```bash
tune2fs -l /dev/sdc1 | grep -E 'Filesystem features|User quota inode|Group quota inode'
```

다시 마운트:

```bash
mount /githome
```

---

# 11. 기존 외부 Quota 파일 방식

기존 방식에서는 `quotacheck`를 사용하여 다음과 같은 외부 Quota 파일을 생성할 수 있다.

```text
aquota.user
aquota.group
```

예:

```bash
quotacheck -cu /githome
quotacheck -cg /winhome
```

옵션:

| 옵션 | 의미 |
|---|---|
| `-c` | 새로운 Quota 파일 생성 |
| `-u` | User Quota |
| `-g` | Group Quota |

Rocky Linux 9의 ext4 환경에서는 외부 Quota 파일 방식 사용 시 다음과 같은 경고가 발생할 수 있다.

```text
Your kernel probably supports ext4 quota feature
but you are using external quota files.

Please switch your filesystem to use ext4 quota feature
as external quota files on ext4 are deprecated.
```

따라서 이번 실습에서는 외부 `aquota.user`, `aquota.group` 파일 방식 대신 ext4 자체 Quota Feature를 사용하였다.

```bash
tune2fs -O quota DEVICE
```

---

# 12. edquota

`edquota`는 사용자 또는 그룹의 Quota 제한값을 편집하는 명령어이다.

사용자:

```bash
edquota -u USER
```

그룹:

```bash
edquota -g GROUP
```

예:

```bash
edquota -u quser1
```

화면 예:

```text
Filesystem    blocks    soft    hard    inodes    soft    hard
/dev/sdc1         28   10240   15360         7       0       0
```

각 항목의 의미:

| 항목 | 의미 |
|---|---|
| Filesystem | Quota를 적용하는 파일시스템 |
| blocks | 현재 사용 중인 디스크 용량 |
| block soft | 디스크 용량 Soft Limit |
| block hard | 디스크 용량 Hard Limit |
| inodes | 현재 사용 중인 inode 개수 |
| inode soft | inode Soft Limit |
| inode hard | inode Hard Limit |

`0`으로 설정된 Limit은 해당 항목을 제한하지 않는다는 의미이다.

---

# 13. quota 명령어

특정 사용자 또는 그룹의 Quota 상태를 확인한다.

사용자:

```bash
quota -u USER
```

예:

```bash
quota -u quser1
```

그룹:

```bash
quota -g GROUP
```

예:

```bash
quota -g winteam
```

`quota` 출력은 다음과 같은 형태로 표시된다.

```text
Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
```

여기서:

```text
blocks
└── 현재 디스크 사용량

첫 번째 quota
└── Block Soft Limit

첫 번째 limit
└── Block Hard Limit

files
└── 현재 inode 사용량

두 번째 quota
└── Inode Soft Limit

두 번째 limit
└── Inode Hard Limit

grace
└── Soft Limit 초과 후 남은 유예 시간
```

---

# 14. repquota

`repquota`는 파일시스템에 설정된 여러 사용자 또는 그룹의 Quota 상태를 한 번에 확인할 때 사용한다.

User Quota:

```bash
repquota -u /githome
```

또는:

```bash
repquota /githome
```

Group Quota:

```bash
repquota -g /winhome
```

출력에서는 Block Limit과 File Limit을 함께 확인할 수 있다.

```text
Block limits              File limits
used soft hard grace      used soft hard grace
```

---

# 15. repquota 상태 표시

`repquota`에서는 제한 초과 상태를 기호로 나타낼 수 있다.

```text
--
+-
-+
++
```

의미:

| 표시 | 의미 |
|---|---|
| `--` | Block과 Inode 모두 Soft Limit 이하 |
| `+-` | Block Soft Limit 초과 |
| `-+` | Inode Soft Limit 초과 |
| `++` | Block과 Inode 모두 Soft Limit 초과 |

예:

```text
quser1  +-  15360  10240  15360  7days
```

`+-`는 Block 사용량이 Soft Limit을 초과했다는 의미이다.

---

# 16. quotaon

`quotaon`은 Quota를 활성화하거나 활성화 상태를 확인하는 명령어이다.

Quota 활성화:

```bash
quotaon /githome
```

User/Group Quota:

```bash
quotaon -ug /winhome
```

현재 상태 확인:

```bash
quotaon -p /githome
```

예:

```text
group quota on /githome (/dev/sdc1) is on
user quota on /githome (/dev/sdc1) is on
project quota on /githome (/dev/sdc1) is off
```

---

# 17. quotaoff

Quota 기능을 비활성화할 때 사용할 수 있다.

```bash
quotaoff /githome
```

User/Group Quota 비활성화:

```bash
quotaoff -ug /winhome
```

Quota 설정 변경 또는 관리 작업을 할 때 필요에 따라 사용할 수 있다.

---

# 18. Quota 실습 기본 흐름

전체 흐름은 다음과 같이 정리할 수 있다.

```text
1. 디스크 준비
        ↓
2. 파티션 생성
        ↓
3. ext4 파일시스템 생성
        ↓
4. 마운트 포인트 생성
        ↓
5. /etc/fstab에 usrquota / grpquota 설정
        ↓
6. ext4 Quota Feature 확인 및 활성화
        ↓
7. 파일시스템 마운트
        ↓
8. quotaon 상태 확인
        ↓
9. 사용자 / 그룹 생성
        ↓
10. edquota로 Soft / Hard Limit 설정
        ↓
11. quota / repquota로 확인
        ↓
12. 실제 파일 생성
        ↓
13. Hard Limit 초과 동작 검증
```

---

# 19. 실제 제한 발생 시 메시지

Block Hard Limit에 도달하면 다음과 같은 형태의 오류가 발생할 수 있다.

```text
write failed, user block limit reached.
디스크 할당량이 초과됨
```

Group Block Limit:

```text
write failed, group block limit reached.
디스크 할당량이 초과됨
```

Inode Hard Limit:

```text
write failed, user file limit reached.
디스크 할당량이 초과됨
```

즉 Quota는 단순히 설정값만 저장하는 기능이 아니라 실제 파일 쓰기 또는 생성 작업을 제한한다.

---

# 20. Block Hard Limit에서 파일이 일부 생성되는 이유

예를 들어 현재 사용량이 13MiB이고 Hard Limit이 15MiB인 상태에서 5MiB 파일 생성을 요청한다고 가정한다.

```text
현재 사용량 = 13MiB
Hard Limit  = 15MiB

추가 요청 = 5MiB
```

하지만 남은 허용 공간은 약 2MiB뿐이다.

따라서:

```text
5MiB 기록 요청
      ↓
약 2MiB까지 기록
      ↓
Hard Limit 도달
      ↓
나머지 기록 차단
```

이 때문에 Quota 제한 테스트에서는 파일 자체가 생성되었는지만 확인해서는 안 된다.

다음 항목을 함께 확인해야 한다.

```bash
quota -u USER
ls -lh FILE
```

---

# 21. Inode Quota 테스트 시 주의점

새 사용자를 생성하고 홈 디렉터리를 만들면 홈 디렉터리가 완전히 비어 있지 않을 수 있다.

예:

```text
.bashrc
.bash_profile
.bash_logout
```

등의 기본 파일이 존재할 수 있으므로 inode 사용량이 처음부터 `0`이 아닐 수 있다.

따라서 Inode Quota 테스트 전:

```bash
quota -u USER
```

로 현재 inode 사용량을 먼저 확인해야 한다.

예:

```text
현재 inode = 7
Hard Limit = 30
```

이라면 추가로 생성할 수 있는 inode는 최대 23개이다.

```text
7 + 23 = 30
```

이후 추가 inode 생성은 Hard Limit에 의해 차단된다.

---

# 22. Quota 설정 시 0의 의미

`edquota`의 Soft 또는 Hard 값을 `0`으로 설정하면 해당 항목은 제한하지 않는다.

예:

```text
blocks
soft = 0
hard = 0

inodes
soft = 20
hard = 30
```

의미:

```text
디스크 용량 제한 없음
파일 개수만 제한
```

반대로:

```text
blocks
soft = 10240
hard = 15360

inodes
soft = 0
hard = 0
```

이면:

```text
디스크 용량만 제한
파일 개수 제한 없음
```

---

# 23. Quota에서 기억할 핵심

```text
Quota
│
├── User Quota
│   └── 사용자 단위 제한
│
├── Group Quota
│   └── 그룹 단위 제한
│
├── Block Quota
│   └── 디스크 용량 제한
│
└── Inode Quota
    └── 파일/디렉터리 개수 제한
```

그리고 제한값은:

```text
Soft Limit
└── 경고 기준
    Grace Period 적용

Hard Limit
└── 절대 최대값
    즉시 제한
```

---

# 24. 주요 명령어 정리

| 명령어 | 설명 |
|---|---|
| `quota -u USER` | 특정 사용자의 Quota 확인 |
| `quota -g GROUP` | 특정 그룹의 Quota 확인 |
| `edquota -u USER` | 사용자 Quota 설정 |
| `edquota -g GROUP` | 그룹 Quota 설정 |
| `repquota -u MOUNTPOINT` | 사용자 Quota 전체 현황 확인 |
| `repquota -g MOUNTPOINT` | 그룹 Quota 전체 현황 확인 |
| `quotaon MOUNTPOINT` | Quota 활성화 |
| `quotaon -p MOUNTPOINT` | Quota 활성화 상태 확인 |
| `quotaoff MOUNTPOINT` | Quota 비활성화 |
| `tune2fs -l DEVICE` | ext4 파일시스템 Feature 확인 |
| `tune2fs -O quota DEVICE` | ext4 Quota Feature 활성화 |
| `mount -o remount MOUNTPOINT` | 변경된 마운트 옵션 적용 |
| `findmnt MOUNTPOINT` | 마운트 및 옵션 확인 |

---

# 25. 이번 실습에서 사용한 구성

```text
/dev/sdc
│
├── /dev/sdc1
│   ├── ext4
│   └── /githome
│       ├── User Block Quota
│       └── User Inode Quota
│
└── /dev/sdc2
    ├── ext4
    └── /winhome
        └── Group Block Quota
```

User Block Quota:

```text
quser1

Soft = 10 MiB
Hard = 15 MiB
```

실제 Hard Limit 도달 후 추가 쓰기가 차단되는 것을 확인하였다.

User Inode Quota:

```text
quser2

Soft = 20
Hard = 30
```

총 inode 30개에 도달한 후 새로운 파일 생성이 차단되는 것을 확인하였다.

Group Block Quota:

```text
winteam

Soft = 20 MiB
Hard = 40 MiB
```

`guser1`, `guser2`, `guser3`의 그룹 사용량이 약 40MiB에 도달한 후 추가 쓰기가 차단되는 것을 확인하였다.

---

# 26. 최종 정리

Disk Quota는 다중 사용자 Linux 서버에서 디스크 자원을 사용자 또는 그룹별로 제한하기 위한 기능이다.

핵심은 다음과 같다.

```text
Block
→ 디스크 용량 제한

Inode
→ 파일/디렉터리 개수 제한

Soft Limit
→ 경고 기준
→ Grace Period 적용

Hard Limit
→ 절대 최대값
→ 즉시 제한

User Quota
→ 사용자 단위 제한

Group Quota
→ 그룹 단위 제한
```

Quota 설정 후에는 단순히 설정값만 확인하는 것이 아니라 실제 사용자 권한으로 파일을 생성하여 제한이 정상적으로 동작하는지 검증하는 것이 중요하다.

이번 실습에서는 User Block, Group Block, User Inode Quota를 각각 설정하고 Hard Limit 도달 시 실제로 파일 쓰기 및 생성이 차단되는 것을 확인하였다.
