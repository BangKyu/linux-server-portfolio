# Rocky Linux 시스템 운영

> Linux 시스템에서 **사용자 계정, 비밀번호, 그룹, 관리자 권한**을 관리하는 방법
> 

---

# 1. 사용자 계정 생성 — `useradd`

Linux에서 사용자를 생성하고 시스템에 접근할 수 있도록 계정 정보를 설정하는 명령어

사용자 계정에는 일반적으로 다음과 같은 정보가 부여된다.

- **UID** : 사용자 식별 번호
- **GID** : 기본 그룹 식별 번호
- **Home Directory** : 사용자의 작업 공간
- **Login Shell** : 로그인 후 사용할 Shell
- **Password** : 사용자 인증을 위한 비밀번호

### 기본 형식

```bash
useradd [옵션] [옵션값] [계정명]
```

예시:

```bash
useradd user1
```

→ `user1` 사용자 생성

---

## 주요 옵션

| 옵션 | 기능 |
| --- | --- |
| `-d` | 사용자의 홈 디렉터리 지정 |
| `-c` | 사용자 설명(Comment) 지정 |
| `-s` | 로그인 Shell 지정 |
| `-g` | 기본 그룹(Primary Group) 지정 |
| `-G` | 보조 그룹(Secondary Group) 지정 |
| `-a` | 기존 보조 그룹에 그룹 추가 시 사용 |

### 홈 디렉터리 지정

```bash
useradd -d /home/test test
```

→ `test` 사용자의 홈 디렉터리를 `/home/test`로 지정

### 사용자 설명 지정

```bash
useradd -c "Test User" test
```

→ 사용자 계정에 `Test User`라는 설명 추가

### 로그인 Shell 지정

```bash
useradd -s /bin/bash test
```

→ `test` 사용자의 로그인 Shell을 `/bin/bash`로 지정

### 기본 그룹 지정

```bash
useradd -g developers test
```

→ `test`의 기본 그룹을 `developers`로 지정

### 보조 그룹 지정

```bash
useradd -G wheel,docker test
```

→ `test`를 `wheel`, `docker` 보조 그룹에 추가

> ⭐ **Primary Group**
> 
> - `g` → 기본 그룹 1개 지정
> 
> **Secondary Group**
> 
> - `G` → 보조 그룹 여러 개 지정 가능

---

# 2. 사용자 계정 관련 주요 파일

Linux에서는 사용자 계정 정보를 여러 설정 파일에서 관리한다.

| 경로 | 역할 |
| --- | --- |
| `/etc/passwd` | 사용자 계정의 기본 정보 |
| `/etc/shadow` | 사용자 비밀번호의 해시 및 비밀번호 정책 정보 |
| `/etc/group` | 그룹 정보 |
| `/etc/default/useradd` | `useradd` 실행 시 적용되는 기본 설정 |
| `/etc/skel` | 새로운 사용자의 홈 디렉터리에 기본적으로 복사되는 파일 |
| `/etc/login.defs` | 계정 및 비밀번호 관련 전역 기본 설정 |
| `/var/spool/mail` | 사용자 로컬 메일 저장 위치 |

---

## `/etc/passwd`

사용자 계정의 기본 정보를 저장한다.

예시:

```
user1:x:1001:1001:Test User:/home/user1:/bin/bash
```

구성:

```
사용자명 : 비밀번호 표시 : UID : GID : 설명 : 홈 디렉터리 : 로그인 Shell
```

> ⚠️ 실제 비밀번호가 `/etc/passwd`에 평문으로 저장되는 것은 아니다.
> 
> 
> 일반적으로 `x`가 표시되고 실제 비밀번호 관련 정보는 `/etc/shadow`에서 관리한다.
> 

---

## `/etc/shadow`

사용자의 **비밀번호 해시와 비밀번호 만료 관련 정보**를 저장한다.

```
/etc/shadow
```

> ⚠️ 비밀번호가 평문으로 저장되는 파일이 아니다.
> 
> 
> 저장된 해시값으로부터 원래 비밀번호를 확인할 수 없다.
> 

---

## `/etc/group`

시스템에 생성된 그룹의 정보를 저장한다.

```
/etc/group
```

예시:

```
developers:x:1002:user1,user2
```

→ `developers` 그룹에 `user1`, `user2` 등이 포함되어 있음을 의미

---

## `/etc/default/useradd`

`useradd`로 사용자를 생성할 때 사용되는 기본 설정을 확인할 수 있다.

```bash
cat /etc/default/useradd
```

예:

```
SHELL=/bin/bash
```

→ 별도로 Shell을 지정하지 않았을 때 기본 Shell 설정 등에 영향을 준다.

---

## `/etc/skel`

새로운 사용자의 홈 디렉터리를 생성할 때 기본 파일을 제공하는 디렉터리

```
/etc/skel
```

예를 들어 `/etc/skel`에 다음 파일이 있다면:

```
/etc/skel/.bashrc
/etc/skel/.bash_profile
```

사용자 생성 시 해당 파일들이 사용자의 홈 디렉터리에 복사될 수 있다.

```
/etc/skel
      ↓
useradd user1
      ↓
/home/user1/
```

> ⭐ `/etc/skel` = **새 사용자에게 제공되는 기본 파일 보관 장소**
> 

---

## `/etc/login.defs`

사용자 계정 및 비밀번호와 관련된 시스템 전역 기본 설정을 저장한다.

예:

- UID/GID 관련 기본 범위
- 비밀번호 만료 정책
- 계정 생성 관련 기본 정책

---

# 3. 사용자 계정 수정 — `usermod`

이미 생성된 사용자 계정의 속성을 변경하는 명령어

### 기본 형식

```bash
usermod [옵션] [옵션값] [계정명]
```

### 주요 옵션

| 옵션 | 기능 |
| --- | --- |
| `-s` | 로그인 Shell 변경 |
| `-d` | 홈 디렉터리 경로 변경 |
| `-m` | 기존 홈 디렉터리의 내용을 새 위치로 이동 |
| `-d + -m` | 홈 디렉터리 경로 변경 + 기존 데이터 이동 |
| `-g` | 기본 그룹 변경 |
| `-G` | 보조 그룹 변경 |
| `-aG` | 기존 보조 그룹을 유지하면서 새로운 보조 그룹 추가 |

---

## 로그인 Shell 변경

```bash
usermod -s /bin/bash user1
```

→ `user1`의 로그인 Shell을 `/bin/bash`로 변경

---

## 홈 디렉터리 경로 변경

```bash
usermod -d /home/newhome user1
```

→ `/etc/passwd`에 기록된 홈 디렉터리 경로를 변경

> ⚠️ `-d`만 사용하면 **기존 홈 디렉터리의 실제 파일을 자동으로 이동시키지 않는다.**
> 

---

## 홈 디렉터리 이동

```bash
usermod -d /home/newhome -m user1
```

→ 홈 디렉터리 경로를 변경하면서 기존 홈 디렉터리의 내용도 이동

### 핵심 비교

```
usermod -d
→ 경로만 변경

usermod -d -m
→ 경로 변경 + 기존 데이터 이동
```

---

## 보조 그룹 추가

```bash
usermod -aG wheel user1
```

→ `user1`을 `wheel` 보조 그룹에 추가

> ⭐ `-aG`는 매우 중요
> 
> - `G`만 사용하면 기존 보조 그룹 목록이 새로운 목록으로 **대체될 수 있다.**
> 
> 기존 그룹을 유지하면서 추가하려면:
> 
> ```bash
> usermod -aG 그룹명 계정명
> ```
> 

---

# 4. 사용자 계정 삭제 — `userdel`

사용자 계정을 삭제하는 명령어

### 기본 형식

```bash
userdel [옵션] [계정명]
```

예시:

```bash
userdel user1
```

→ `user1` 계정 삭제

### 홈 디렉터리까지 삭제

```bash
userdel -r user1
```

→ 사용자 계정과 해당 사용자의 홈 디렉터리 및 관련 메일 스풀 등을 함께 삭제

> ⚠️ `userdel -r`은 사용자의 파일을 삭제할 수 있으므로 실행 전에 반드시 확인한다.
> 

### ⭐ 핵심

```
userdel user1
→ 계정 삭제

userdel -r user1
→ 계정 + 홈 디렉터리 관련 데이터 삭제
```

---

# 5. Password 관리 — `passwd`

사용자의 비밀번호를 설정하거나 비밀번호 상태를 관리하는 명령어

### 기본 형식

```bash
passwd [옵션] [계정명]
```

일반 사용자가 자신의 비밀번호를 변경할 경우:

```bash
passwd
```

관리자가 특정 사용자의 비밀번호를 변경할 경우:

```bash
passwd user1
```

---

## 주요 옵션

| 옵션 | 기능 |
| --- | --- |
| `-S` | 비밀번호 상태 확인 |
| `-l` | 계정의 비밀번호를 잠금(Lock) |
| `-u` | 비밀번호 잠금 해제 |
| `-d` | 비밀번호 삭제 |

### 비밀번호 상태 확인

```bash
passwd -S user1
```

→ `user1`의 비밀번호 상태 확인

### 비밀번호 잠금

```bash
passwd -l user1
```

→ 비밀번호를 잠가 해당 방식의 인증을 통한 로그인을 제한

### 잠금 해제

```bash
passwd -u user1
```

→ 비밀번호 잠금 해제

### 비밀번호 삭제

```bash
passwd -d user1
```

→ `user1`의 비밀번호를 삭제

> ⚠️ `passwd -d`는 보안상 주의해야 한다.
> 
> 
> 비밀번호가 없는 상태가 될 수 있으므로 일반적인 계정 관리에서는 신중하게 사용한다.
> 

---

# 6. Group 관리

여러 사용자를 하나의 **권한 관리 단위**로 묶는 기능

그룹을 사용하면 특정 파일이나 디렉터리에 대해 여러 사용자의 접근 권한을 효율적으로 관리할 수 있다.

### 그룹을 사용하는 이유

```
사용자 1 ─┐
사용자 2 ─┼→ developers 그룹 → 특정 파일/디렉터리 접근
사용자 3 ─┘
```

---

# 7. 그룹 생성 — `groupadd`

새로운 그룹을 생성하는 명령어

### 기본 형식

```bash
groupadd [옵션] [그룹명]
```

예시:

```bash
groupadd developers
```

→ `developers` 그룹 생성

---

# 8. 그룹 삭제 — `groupdel`

기존 그룹을 삭제하는 명령어

### 기본 형식

```bash
groupdel [옵션] [그룹명]
```

예시:

```bash
groupdel developers
```

→ `developers` 그룹 삭제

> ⚠️ 사용자의 기본 그룹으로 사용 중인 그룹은 삭제할 때 주의해야 한다.
> 

---

# 9. Super User — `su`

다른 사용자 계정으로 전환할 때 사용하는 명령어

## `su`

```bash
su
```

→ 기본적으로 `root` 사용자로 전환을 시도한다.

다만 `su`는 **root의 로그인 환경 전체를 그대로 가져오는 것과는 다르다.**

---

## `su -`

```bash
su -
```

→ root로 전환하면서 **root의 로그인 환경을 적용**

즉:

```
su
→ root로 사용자 전환
→ 기존 환경을 일부 유지

su -
→ root로 사용자 전환
→ root의 로그인 환경 적용
```

### 특정 사용자로 전환

```bash
su - user1
```

→ `user1`의 로그인 환경으로 전환

---

# 10. `su` vs `su -` 핵심 비교

| 명령 | 의미 |
| --- | --- |
| `su` | 사용자 전환, 기존 환경을 일부 유지 |
| `su -` | 로그인 방식으로 사용자 전환, 대상 사용자의 환경 적용 |
| `su user1` | `user1`로 전환 |
| `su - user1` | `user1`의 로그인 환경으로 전환 |

> ⭐ 시험에서 자주 나오는 포인트
> 
> 
> `su`와 `su -`의 가장 큰 차이는 **환경 변수 및 작업 환경의 적용 여부**이다.
> 

---

# 11. 사용자 / 그룹 관계

Linux에서는 사용자를 그룹에 소속시켜 권한을 관리한다.

```
                 사용자 계정
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
     기본 그룹              보조 그룹
     Primary Group        Secondary Group
          │                     │
          ↓                     ↓
      기본 소속              추가 소속
```

예시:

```
user1
 ├─ Primary Group    → developers
 ├─ Secondary Group → wheel
 └─ Secondary Group → docker
```

### 관련 명령어

```bash
useradd user1
```

→ 사용자 생성

```bash
groupadd developers
```

→ 그룹 생성

```bash
usermod -g developers user1
```

→ 기본 그룹 변경

```bash
usermod -aG wheel user1
```

→ 보조 그룹 추가

```bash
passwd user1
```

→ 비밀번호 설정

```bash
userdel -r user1
```

→ 사용자 및 홈 디렉터리 삭제

---

# 12. 계정 관리 전체 흐름

```
그룹 생성
    ↓
groupadd
    ↓
사용자 생성
    ↓
useradd
    ↓
비밀번호 설정
    ↓
passwd
    ↓
그룹 추가 / 계정 설정
    ↓
usermod
    ↓
계정 정보 확인
    ↓
/etc/passwd
/etc/shadow
/etc/group
    ↓
필요 시 계정 삭제
    ↓
userdel
```
