# Linux 아카이브 및 압축 실습

## 실습 개요

Rocky Linux에서 `tar`를 이용하여 여러 파일을 하나의 아카이브로 묶고 해제하는 방법을 실습하였다.

`gzip`, `bzip2`, `xz`를 이용하여 아카이브 파일을 압축하고 다시 해제하였으며, 각 압축 방식의 확장자와 명령어 차이를 확인하였다.

`tar`와 압축 방식을 함께 사용하여 아카이브 생성과 압축을 동시에 수행하고, `-C` 옵션을 이용하여 원하는 디렉터리에 해제하였다.

---

## 실습 과정

### 1. 실습 파일 확인

`source` 디렉터리에 파일 3개가 준비된 상태에서 실습을 진행하였다.

```bash
[root@Server-A linux-archive]# ls source/
file1.txt  file2.txt  file3.txt
```

---

### 2. tar 아카이브 생성

`source` 디렉터리의 파일들을 `backup.tar` 아카이브로 생성

```bash
[root@Server-A linux-archive]# tar -cvf backup.tar source/*
source/file1.txt
source/file2.txt
source/file3.txt

[root@Server-A linux-archive]# ls -l
합계 12
-rw-r--r--. 1 root root 10240  9월 16 16:46 backup.tar
drwxr-xr-x. 2 root root    57  9월 16 16:34 source
```

`tar`를 사용하여 'source' 디렉터리 안에 있는 모든 파일을 하나의 아카이브로 묶었다.

사용한 옵션:

- `c` → 새로운 아카이브 생성
- `v` → 작업 과정 출력
- `f` → 아카이브 파일 이름 지정

---

### 3. tar 아카이브 내부 목록 확인

아카이브를 해제하지 않고 내부 파일 목록 확인

```bash
[root@Server-A linux-archive]# tar -tvf backup.tar
-rw-r--r-- root/root         6 2026-09-16 16:34 source/file1.txt
-rw-r--r-- root/root         6 2026-09-16 16:34 source/file2.txt
-rw-r--r-- root/root         6 2026-09-16 16:34 source/file3.txt
```

`t` 옵션을 사용하여 아카이브 내부의 파일 목록과 정보를 확인하였다.

---

### 4. 지정한 디렉터리에 tar 아카이브 해제

디렉터리를 생성하고 생성한 디렉터리에 아카이브를 해제

```bash
[root@Server-A linux-archive]# mkdir restore
[root@Server-A linux-archive]# tar -xvf backup.tar -C restore
source/file1.txt
source/file2.txt
source/file3.txt
```

사용한 옵션:

- `x` → 아카이브 해제
- `C` → 지정한 디렉터리에서 작업 수행

`-C restore`를 사용하여 현재 디렉터리가 아닌 `restore` 디렉터리에 아카이브를 해제하였다.

---

### 5. gzip 압축 및 해제

`backup.tar` 파일을 gzip 방식으로 압축

```bash
[root@Server-A linux-archive]# gzip backup.tar

[root@Server-A linux-archive]# ls -l backup.tar.gz
-rw-r--r--. 1 root root 180  9월 16 16:46 backup.tar.gz
```

gzip 압축 후 `.gz` 확장자가 추가되어 `backup.tar.gz`라는 파일이 생성되었다.

압축 파일을 다시 해제

```bash
[root@Server-A linux-archive]# gunzip backup.tar.gz

[root@Server-A linux-archive]# ls -l backup.tar
-rw-r--r--. 1 root root 10240  9월 16 16:46 backup.tar
```

`gunzip`을 사용하여 압축 파일을 원래의 `backup.tar`로 압축 해제하였다.

---

### 6. bzip2와 xz 압축 비교

비교를 위해 `backup.tar`를 복사

```bash
[root@Server-A linux-archive]# cp backup.tar backup-bzip2.tar
[root@Server-A linux-archive]# cp backup.tar backup-xz.tar
```

각각 bzip2와 xz 방식으로 압축

```bash
[root@Server-A linux-archive]# bzip2 backup-bzip2.tar
[root@Server-A linux-archive]# xz backup-xz.tar

[root@Server-A linux-archive]# ls -lh
합계 20K
-rw-r--r--. 1 root root 174  9월 16 17:07 backup-bzip2.tar.bz2
-rw-r--r--. 1 root root 212  9월 16 17:07 backup-xz.tar.xz
-rw-r--r--. 1 root root 10K  9월 16 16:46 backup.tar
drwxr-xr-x. 3 root root  20  9월 16 16:47 restore
drwxr-xr-x. 2 root root  57  9월 16 16:34 source
```

각 압축 방식의 확장자:

- gzip → `.gz`
- bzip2 → `.bz2`
- xz → `.xz`

압축 해제

```bash
[root@Server-A linux-archive]# bunzip2 backup-bzip2.tar.bz2
[root@Server-A linux-archive]# unxz backup-xz.tar.xz

[root@Server-A linux-archive]# ls -lh
합계 36K
-rw-r--r--. 1 root root 10K  9월 16 17:07 backup-bzip2.tar
-rw-r--r--. 1 root root 10K  9월 16 17:07 backup-xz.tar
-rw-r--r--. 1 root root 10K  9월 16 16:46 backup.tar
drwxr-xr-x. 3 root root  20  9월 16 16:47 restore
drwxr-xr-x. 2 root root  57  9월 16 16:34 source
```

`bunzip2`, `unxz`를 사용하여 각각 원래의 tar 파일로 압축 해제되는 것을 확인하였다.

---

### 7. tar와 압축을 동시에 수행

`tar` 명령에서 압축 옵션을 함께 사용하여 아카이브 생성과 압축을 한 번에 수행

#### gzip

```bash
[root@Server-A linux-archive]# tar -czvf backup-gzip.tar.gz source/
source/
source/file1.txt
source/file2.txt
source/file3.txt
```

#### bzip2

```bash
[root@Server-A linux-archive]# tar -cjvf backup-bzip2.tar.bz2 source/
source/
source/file1.txt
source/file2.txt
source/file3.txt
```

#### xz

```bash
[root@Server-A linux-archive]# tar -cJvf backup-xz.tar.xz source/
source/
source/file1.txt
source/file2.txt
source/file3.txt
```

생성된 압축 파일 확인

```bash
[root@Server-A linux-archive]# ls -l
합계 24
-rw-r--r--. 1 root root   197  9월 16 17:15 backup-bzip2.tar.bz2
-rw-r--r--. 1 root root   195  9월 16 17:15 backup-gzip.tar.gz
-rw-r--r--. 1 root root   228  9월 16 17:15 backup-xz.tar.xz
-rw-r--r--. 1 root root 10240  9월 16 16:46 backup.tar
drwxr-xr-x. 3 root root    20  9월 16 16:47 restore
drwxr-xr-x. 2 root root    57  9월 16 16:34 source
```

압축 방식별 `tar` 옵션:

- `z` → gzip
- `j` → bzip2
- `J` → xz

---

### 8. 압축 파일을 지정한 디렉터리에 복원

복원 테스트를 위해 각각의 디렉터리 생성

```bash
[root@Server-A linux-archive]# mkdir restore-gzip restore-bzip2 restore-xz
```

각 압축 방식에 맞는 옵션을 사용하여 복원

```bash
[root@Server-A linux-archive]# tar -xzvf backup-gzip.tar.gz -C restore-gzip
source/
source/file1.txt
source/file2.txt
source/file3.txt

[root@Server-A linux-archive]# tar -xjvf backup-bzip2.tar.bz2 -C restore-bzip2
source/
source/file1.txt
source/file2.txt
source/file3.txt

[root@Server-A linux-archive]# tar -xJvf backup-xz.tar.xz -C restore-xz
source/
source/file1.txt
source/file2.txt
source/file3.txt
```

gzip 복원 결과 확인

```bash
[root@Server-A linux-archive]# ls -l restore-gzip/source/
합계 12
-rw-r--r--. 1 root root 6  9월 16 16:34 file1.txt
-rw-r--r--. 1 root root 6  9월 16 16:34 file2.txt
-rw-r--r--. 1 root root 6  9월 16 16:34 file3.txt
```

bzip2 복원 결과 확인

```bash
[root@Server-A linux-archive]# ls -l restore-bzip2/source/
합계 12
-rw-r--r--. 1 root root 6  9월 16 16:34 file1.txt
-rw-r--r--. 1 root root 6  9월 16 16:34 file2.txt
-rw-r--r--. 1 root root 6  9월 16 16:34 file3.txt
```

xz 복원 결과 확인

```bash
[root@Server-A linux-archive]# ls -l restore-xz/source/
합계 12
-rw-r--r--. 1 root root 6  9월 16 16:34 file1.txt
-rw-r--r--. 1 root root 6  9월 16 16:34 file2.txt
-rw-r--r--. 1 root root 6  9월 16 16:34 file3.txt
```

gzip, bzip2, xz 방식으로 생성한 압축 파일이 모두 정상적으로 복원되는 것을 확인하였다.

---

## 실습 결과

- `tar`를 이용하여 여러 파일을 하나의 아카이브로 생성
- `tar -t`를 이용하여 아카이브를 해제하지 않고 내부 파일 목록 확인
- `tar -x`와 `-C` 옵션을 이용하여 지정한 디렉터리에 아카이브 복원
- `gzip`, `bzip2`, `xz`를 이용한 압축 및 압축 해제 실습
- `tar`의 `z`, `j`, `J` 옵션을 이용하여 아카이브 생성과 압축을 동시에 수행
- gzip, bzip2, xz 압축 파일을 각각 별도의 디렉터리에 복원하여 원본 파일 확인
- `tar`는 파일을 하나의 아카이브로 묶는 기능이고, gzip·bzip2·xz는 데이터를 압축하는 기능임을 확인
