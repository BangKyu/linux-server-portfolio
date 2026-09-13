# Rocky Linux 압축 및 아카이브 관리

> Linux 시스템에서 **gzip, bzip2, xz를 이용한 파일 압축과 tar를 이용한 파일 아카이브 및 압축 관리 방법**

---

# 1. 파일 압축 — Compression

파일 압축은 파일이 사용하는 저장 공간을 줄이는 기능이다.

파일을 압축하면 다음과 같은 장점이 있다.

- 디스크 저장 공간 절약
- 네트워크를 통한 파일 전송 시 전송 용량 감소
- 백업 파일의 효율적인 보관

Linux에서는 다음과 같은 압축 명령어를 사용할 수 있다.

```text
gzip
bzip2
xz
```

각 명령어는 기본적으로 파일 단위로 압축한다.

예를 들어 여러 파일을 다음과 같이 지정하더라도:

```bash
gzip file1 file2 file3
```

하나의 압축 파일이 생성되는 것이 아니라 각각 따로 압축된다.

```text
file1.gz
file2.gz
file3.gz
```

여러 파일과 디렉터리를 **하나의 파일로 묶어서 관리**하려면 `tar`를 사용한다.

---

# 2. gzip

`gzip`은 Linux에서 사용하는 파일 압축 명령어이다.

특징:

- 비교적 빠른 압축 속도
- 파일을 `.gz` 형식으로 압축
- 서버 로그 파일 압축 등에 사용
- 기본 사용 시 압축 후 원본 파일이 `.gz` 파일로 대체됨

### 기본 형식

파일 압축:

```bash
gzip [파일명]
```

예:

```bash
gzip file1.txt
```

결과:

```text
file1.txt
    ↓
file1.txt.gz
```

---

## gzip 압축 해제

방법 1:

```bash
gzip -d file1.txt.gz
```

방법 2:

```bash
gunzip file1.txt.gz
```

결과:

```text
file1.txt.gz
    ↓
file1.txt
```

---

# 3. bzip2

`bzip2`는 파일을 `.bz2` 형식으로 압축하는 명령어이다.

특징:

- gzip보다 압축 속도가 상대적으로 느림
- 높은 압축률을 목적으로 사용할 수 있음
- 서버 백업 및 프로그램 소스 배포 파일 등에 사용
- 기본 사용 시 원본 파일이 `.bz2` 파일로 대체됨

### 기본 형식

```bash
bzip2 [파일명]
```

예:

```bash
bzip2 file2.txt
```

결과:

```text
file2.txt
    ↓
file2.txt.bz2
```

---

## bzip2 압축 해제

방법 1:

```bash
bzip2 -d file2.txt.bz2
```

방법 2:

```bash
bunzip2 file2.txt.bz2
```

결과:

```text
file2.txt.bz2
    ↓
file2.txt
```

---

# 4. xz

`xz`는 파일을 `.xz` 형식으로 압축하는 명령어이다.

특징:

- 높은 압축률을 목적으로 사용
- 압축 과정에서 CPU 사용량과 시간이 증가할 수 있음
- 대용량 파일이나 백업 파일 등의 압축에 사용

### 기본 형식

```bash
xz [파일명]
```

예:

```bash
xz file3.txt
```

결과:

```text
file3.txt
    ↓
file3.txt.xz
```

---

## xz 압축 해제

방법 1:

```bash
xz -d file3.txt.xz
```

방법 2:

```bash
unxz file3.txt.xz
```

결과:

```text
file3.txt.xz
    ↓
file3.txt
```

---

# 5. 압축 명령어 비교

| 명령어 | 압축 파일 확장자 | 압축 해제 명령어 |
| --- | --- | --- |
| `gzip` | `.gz` | `gzip -d`, `gunzip` |
| `bzip2` | `.bz2` | `bzip2 -d`, `bunzip2` |
| `xz` | `.xz` | `xz -d`, `unxz` |

예:

```text
file.txt
   │
   ├─ gzip  → file.txt.gz
   │
   ├─ bzip2 → file.txt.bz2
   │
   └─ xz    → file.txt.xz
```

---

# 6. tar — Tape Archive

`tar`는 **Tape Archive**의 약자이다.

여러 개의 파일과 디렉터리를 하나의 파일로 묶어 관리하는 명령어이다.

중요한 점:

> `tar` 자체의 기본 기능은 압축이 아니라 여러 파일을 하나로 묶는 **아카이브(Archive)** 기능이다.

예를 들어:

```text
file1.txt
file2.txt
file3.txt
```

세 개의 파일을 `tar`로 묶으면:

```text
file1.txt ─┐
file2.txt ─┼─→ archive.tar
file3.txt ─┘
```

하나의 `archive.tar` 파일로 관리할 수 있다.

---

# 7. tar 기본 형식

### 기본 형식

```bash
tar [옵션] [아카이브 파일명] [대상 파일 또는 디렉터리]
```

예:

```bash
tar -cvf archive.tar file1.txt file2.txt file3.txt
```

→ 세 개의 파일을 `archive.tar` 하나로 묶음

---

# 8. tar 주요 옵션

| 옵션 | 기능 |
| --- | --- |
| `-c` | 새로운 tar 아카이브 생성 |
| `-x` | tar 아카이브 해제 |
| `-t` | tar 내부 파일 목록 확인 |
| `-v` | 작업 과정 출력 |
| `-f` | 사용할 아카이브 파일명 지정 |
| `-z` | gzip 사용 |
| `-j` | bzip2 사용 |
| `-C` | 지정한 디렉터리에서 작업 |

---

# 9. tar 아카이브 생성

여러 파일을 하나의 tar 파일로 묶는다.

```bash
tar -cvf archive.tar file1.txt file2.txt file3.txt
```

옵션:

```text
-c → 새로운 아카이브 생성
-v → 작업 과정 출력
-f → 아카이브 파일명 지정
```

결과:

```text
file1.txt
file2.txt
file3.txt
archive.tar
```

`tar`를 사용하여 파일을 하나로 묶어도 원본 파일은 삭제되지 않는다.

즉:

```text
원본 파일 유지
+
archive.tar 생성
```

---

# 10. tar 내부 파일 확인

tar 파일을 해제하지 않고 내부에 어떤 파일이 들어 있는지 확인할 수 있다.

```bash
tar -tvf archive.tar
```

옵션:

```text
-t → 아카이브 내부 목록 확인
-v → 상세 정보 출력
-f → 확인할 아카이브 파일 지정
```

예:

```text
archive.tar
├── file1.txt
├── file2.txt
└── file3.txt
```

---

# 11. tar 아카이브 해제

tar로 묶은 파일을 다시 원래의 파일들로 복원한다.

```bash
tar -xvf archive.tar
```

옵션:

```text
-x → 아카이브 해제
-v → 작업 과정 출력
-f → 아카이브 파일명 지정
```

결과:

```text
archive.tar
    ↓
file1.txt
file2.txt
file3.txt
```

---

# 12. 지정한 디렉터리에 tar 해제 — `-C`

`-C` 옵션을 사용하면 원하는 디렉터리에 아카이브를 해제할 수 있다.

예:

```bash
mkdir restore
```

```bash
tar -xvf archive.tar -C restore/
```

결과:

```text
restore/
├── file1.txt
├── file2.txt
└── file3.txt
```

즉:

```text
-C
→ 압축 또는 아카이브 해제 위치 지정
```

---

# 13. tar + gzip

`tar`에 gzip 옵션을 함께 사용하면 여러 파일을 하나로 묶은 후 gzip 방식으로 압축할 수 있다.

일반적으로 확장자는 다음과 같이 사용한다.

```text
.tar.gz
```

또는:

```text
.tgz
```

### 압축

```bash
tar -czvf archive.tar.gz file1.txt file2.txt file3.txt
```

옵션:

```text
-c → 새로운 아카이브 생성
-z → gzip 사용
-v → 작업 과정 출력
-f → 파일명 지정
```

동작 과정:

```text
file1.txt ─┐
file2.txt ─┼─→ tar로 하나로 묶기 → gzip 압축 → archive.tar.gz
file3.txt ─┘
```

---

## tar.gz 내부 목록 확인

```bash
tar -tzvf archive.tar.gz
```

옵션:

```text
-t → 내부 목록 확인
-z → gzip 형식
-v → 상세 출력
-f → 파일명 지정
```

---

## tar.gz 압축 해제

```bash
tar -xzvf archive.tar.gz
```

지정한 디렉터리에 해제:

```bash
tar -xzvf archive.tar.gz -C restore/
```

---

# 14. tar + bzip2

`tar`에 bzip2 옵션을 함께 사용하면 여러 파일을 하나로 묶은 후 bzip2 방식으로 압축할 수 있다.

일반적으로 확장자는 다음과 같이 사용한다.

```text
.tar.bz2
```

### 압축

```bash
tar -cjvf archive.tar.bz2 file1.txt file2.txt file3.txt
```

옵션:

```text
-c → 새로운 아카이브 생성
-j → bzip2 사용
-v → 작업 과정 출력
-f → 파일명 지정
```

동작 과정:

```text
file1.txt ─┐
file2.txt ─┼─→ tar로 하나로 묶기 → bzip2 압축 → archive.tar.bz2
file3.txt ─┘
```

---

## tar.bz2 내부 목록 확인

```bash
tar -tjvf archive.tar.bz2
```

---

## tar.bz2 압축 해제

```bash
tar -xjvf archive.tar.bz2
```

지정한 디렉터리에 해제:

```bash
tar -xjvf archive.tar.bz2 -C restore/
```

---

# 15. tar와 압축의 차이

`tar`와 `gzip`, `bzip2`, `xz`의 역할을 구분하는 것이 중요하다.

### gzip / bzip2 / xz

```text
목적
→ 파일 용량을 줄이는 압축
```

예:

```text
file.txt
    ↓ gzip
file.txt.gz
```

---

### tar

```text
목적
→ 여러 파일과 디렉터리를 하나의 파일로 묶는 아카이브
```

예:

```text
file1
file2
file3
  ↓
archive.tar
```

---

### tar + 압축 프로그램

```text
여러 파일
    ↓
tar
    ↓
하나의 아카이브
    ↓
gzip / bzip2
    ↓
압축된 아카이브
```

예:

```text
file1.txt
file2.txt
file3.txt
    ↓
archive.tar
    ↓ gzip
archive.tar.gz
```

---

# 16. tar를 먼저 묶고 따로 압축하는 방법

tar 파일을 먼저 만든 후 별도의 압축 명령어를 사용할 수도 있다.

예:

```bash
tar -cvf archive.tar file1.txt file2.txt file3.txt
```

그다음 bzip2 압축:

```bash
bzip2 archive.tar
```

결과:

```text
archive.tar
    ↓
archive.tar.bz2
```

---

# 17. tar와 압축을 한 번에 수행하는 방법

별도의 압축 명령어를 실행하지 않고 `tar` 옵션을 이용하여 한 번에 처리할 수도 있다.

gzip:

```bash
tar -czvf archive.tar.gz file1.txt file2.txt file3.txt
```

bzip2:

```bash
tar -cjvf archive.tar.bz2 file1.txt file2.txt file3.txt
```

즉:

```text
방법 1

tar로 묶기
    ↓
gzip / bzip2로 압축
```

또는:

```text
방법 2

tar 명령어에서
-z 또는 -j 옵션 사용
    ↓
묶기 + 압축을 한 번에 수행
```

---

# 18. 주요 확장자

| 확장자 | 의미 |
| --- | --- |
| `.gz` | gzip으로 압축된 파일 |
| `.bz2` | bzip2로 압축된 파일 |
| `.xz` | xz로 압축된 파일 |
| `.tar` | tar로 묶은 아카이브 |
| `.tar.gz` | tar + gzip |
| `.tar.bz2` | tar + bzip2 |

확장자를 적절하게 사용하면 파일이 어떤 방식으로 만들어졌는지 쉽게 확인할 수 있다.

예:

```text
backup.tar
→ tar 아카이브

backup.tar.gz
→ tar로 묶고 gzip 압축

backup.tar.bz2
→ tar로 묶고 bzip2 압축
```

---

# 19. 주요 명령어 정리

| 명령어 | 기능 |
| --- | --- |
| `gzip file` | gzip 압축 |
| `gzip -d file.gz` | gzip 압축 해제 |
| `gunzip file.gz` | gzip 압축 해제 |
| `bzip2 file` | bzip2 압축 |
| `bzip2 -d file.bz2` | bzip2 압축 해제 |
| `bunzip2 file.bz2` | bzip2 압축 해제 |
| `xz file` | xz 압축 |
| `xz -d file.xz` | xz 압축 해제 |
| `unxz file.xz` | xz 압축 해제 |
| `tar -cvf archive.tar files` | tar 아카이브 생성 |
| `tar -tvf archive.tar` | tar 내부 목록 확인 |
| `tar -xvf archive.tar` | tar 아카이브 해제 |
| `tar -czvf archive.tar.gz files` | tar + gzip |
| `tar -tzvf archive.tar.gz` | tar.gz 내부 목록 확인 |
| `tar -xzvf archive.tar.gz` | tar.gz 압축 해제 |
| `tar -cjvf archive.tar.bz2 files` | tar + bzip2 |
| `tar -tjvf archive.tar.bz2` | tar.bz2 내부 목록 확인 |
| `tar -xjvf archive.tar.bz2` | tar.bz2 압축 해제 |
| `tar ... -C directory` | 지정한 디렉터리에서 작업 |

---

# 20. 압축 및 아카이브 전체 흐름

```text
파일 확인
    ↓
한 개의 파일 용량을 줄여야 하는가?
    ↓
gzip / bzip2 / xz
    ↓
압축 파일 생성
.gz / .bz2 / .xz
```

여러 파일을 하나로 관리해야 하는 경우:

```text
여러 파일 및 디렉터리
        ↓
       tar
        ↓
하나의 아카이브 생성
        ↓
     archive.tar
```

여러 파일을 하나로 묶으면서 압축해야 하는 경우:

```text
여러 파일 및 디렉터리
        ↓
       tar
        ↓
 gzip 또는 bzip2
        ↓
archive.tar.gz
archive.tar.bz2
```

---

# 21. 핵심 정리

```text
gzip
→ 파일을 .gz 형식으로 압축

bzip2
→ 파일을 .bz2 형식으로 압축

xz
→ 파일을 .xz 형식으로 압축

tar
→ 여러 파일과 디렉터리를 하나의 파일로 묶음

tar + gzip
→ 여러 파일을 하나로 묶고 gzip 압축
→ .tar.gz

tar + bzip2
→ 여러 파일을 하나로 묶고 bzip2 압축
→ .tar.bz2
```

가장 중요한 차이:

```text
압축
→ 파일의 용량을 줄이는 것

아카이브
→ 여러 파일을 하나의 파일로 묶는 것
```

따라서 `tar` 자체는 기본적으로 압축 명령어가 아니라 **아카이브 명령어**이며,
필요한 경우 gzip 또는 bzip2와 함께 사용하여 아카이브와 압축을 동시에 수행할 수 있다.
