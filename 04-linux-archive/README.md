# Rocky Linux 압축 및 아카이브 실습

## 실습 내용

Rocky Linux에서 `gzip`, `bzip2`, `xz`를 이용한 파일 압축 및 압축 해제를 실습
`tar`를 이용하여 여러 파일을 하나의 아카이브로 묶는 방법을 확인

`tar`와 `gzip`, `bzip2`를 함께 사용하여 여러 파일을 하나로 묶고 압축하는 방법과
지정한 디렉터리에 파일을 복원하는 방법을 실습

---

## 1. 실습 환경 구성

실습 디렉터리 생성:

```bash
mkdir -p /test/archive
cd /test/archive
```

디렉터리 권한 확인:

```bash
ls -ld /test/archive/
```

실행 결과:

```text
drwxr-xr-x. 2 root root 6  9월 13 11:57 /test/archive/
```

압축 실습을 위한 테스트 파일 생성:

```bash
seq 1 1000 > file1.txt
seq 1 2000 > file2.txt
seq 1 3000 > file3.txt

ls -lh
```

실행 결과:

```text
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

---

## 2. gzip 압축 및 압축 해제

`gzip`을 이용하여 `file1.txt`를 압축하였다.

```bash
gzip file1.txt
```

압축 결과:

```text
-rw-r--r--. 1 root root 1.9K  9월 13 11:58 file1.txt.gz
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

압축 전후 용량:

```text
file1.txt     → 3.9K
file1.txt.gz  → 1.9K
```

기본 `gzip` 명령을 사용하면 원본 파일이 압축 파일로 대체되는 것을 확인

압축 해제:

```bash
gunzip file1.txt.gz
```

압축 해제 결과:

```text
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

→ `file1.txt.gz`가 없어지고 원본 `file1.txt`가 복원되는 것을 확인

---

## 3. bzip2 압축 및 압축 해제

`bzip2`를 이용하여 `file2.txt`를 압축

```bash
bzip2 file2.txt
```

압축 결과:

```text
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 2.1K  9월 13 11:58 file2.txt.bz2
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

압축 전후 용량:

```text
file2.txt      → 8.7K
file2.txt.bz2  → 2.1K
```

압축 해제:

```bash
bunzip2 file2.txt.bz2
```

압축 해제 결과:

```text
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

→ `file2.txt.bz2`가 없어지고 원본 `file2.txt`가 복원되는 것을 확인

---

## 4. xz 압축 및 압축 해제

`xz`를 이용하여 `file3.txt`를 압축

```bash
xz file3.txt
```

압축 결과:

```text
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root 1.3K  9월 13 11:58 file3.txt.xz
```

압축 전후 용량:

```text
file3.txt     → 14K
file3.txt.xz  → 1.3K
```

압축 해제:

```bash
unxz file3.txt.xz
```

압축 해제 결과:

```text
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

→ `file3.txt.xz`가 없어지고 원본 `file3.txt`가 복원되는 것을 확인

---

## 5. tar 아카이브 생성

`tar`를 이용하여 파일 3개를 하나의 아카이브 파일로 묶음

```bash
tar -cvf archive.tar file1.txt file2.txt file3.txt
```

확인:

```bash
ls -lh
```

실행 결과:

```text
-rw-r--r--. 1 root root  30K  9월 13 12:01 archive.tar
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

`tar`로 아카이브를 생성해도 원본 파일은 삭제되지 않는 것을 확인

아카이브 내부 파일 확인:

```bash
tar -tvf archive.tar
```

실행 결과:

```text
-rw-r--r-- root/root      3893 2026-09-13 11:58 file1.txt
-rw-r--r-- root/root      8893 2026-09-13 11:58 file2.txt
-rw-r--r-- root/root     13893 2026-09-13 11:58 file3.txt
```

---

## 6. tar + gzip 압축

`tar`와 `gzip`을 함께 사용하여 여러 파일을 하나로 묶으면서 압축

```bash
tar -czvf archive.tar.gz file1.txt file2.txt file3.txt
```

확인:

```bash
ls -lh
```

실행 결과:

```text
-rw-r--r--. 1 root root  30K  9월 13 12:01 archive.tar
-rw-r--r--. 1 root root 6.7K  9월 13 12:02 archive.tar.gz
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

용량 비교:

```text
archive.tar     → 30K
archive.tar.gz  → 6.7K
```

압축 파일 내부 목록 확인:

```bash
tar -tzvf archive.tar.gz
```

실행 결과:

```text
-rw-r--r-- root/root      3893 2026-09-13 11:58 file1.txt
-rw-r--r-- root/root      8893 2026-09-13 11:58 file2.txt
-rw-r--r-- root/root     13893 2026-09-13 11:58 file3.txt
```

압축을 해제하지 않고 내부 파일 목록을 확인

---

## 7. 지정한 디렉터리에 압축 해제

복원용 디렉터리 생성:

```bash
mkdir restore
```

`-C` 옵션을 이용하여 `restore` 디렉터리에 압축 파일을 해제하였다.

```bash
tar -xzvf archive.tar.gz -C restore/
```

복원 결과 확인:

```bash
ls -lh restore/
```

실행 결과:

```text
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
```

`archive.tar.gz` 내부의 파일 3개가 `restore` 디렉터리에 정상적으로 복원되는 것을 확인

---

## 8. tar + bzip2 압축

`tar`와 `bzip2`를 함께 사용하여 여러 파일을 하나로 묶으면서 압축

```bash
tar -cjvf archive.tar.bz2 file1.txt file2.txt file3.txt
```

확인:

```bash
ls -lh
```

실행 결과:

```text
-rw-r--r--. 1 root root  30K  9월 13 12:01 archive.tar
-rw-r--r--. 1 root root 6.1K  9월 13 12:03 archive.tar.bz2
-rw-r--r--. 1 root root 6.7K  9월 13 12:02 archive.tar.gz
-rw-r--r--. 1 root root 3.9K  9월 13 11:58 file1.txt
-rw-r--r--. 1 root root 8.7K  9월 13 11:58 file2.txt
-rw-r--r--. 1 root root  14K  9월 13 11:58 file3.txt
drwxr-xr-x. 2 root root   57  9월 13 12:03 restore
```

용량 비교:

```text
archive.tar      → 30K
archive.tar.gz   → 6.7K
archive.tar.bz2  → 6.1K
```

아카이브 내부 파일 확인:

```bash
tar -tjvf archive.tar.bz2
```

실행 결과:

```text
-rw-r--r-- root/root      3893 2026-09-13 11:58 file1.txt
-rw-r--r-- root/root      8893 2026-09-13 11:58 file2.txt
-rw-r--r-- root/root     13893 2026-09-13 11:58 file3.txt
```

---

## 9. tar 주요 옵션

| 옵션 | 기능 |
| --- | --- |
| `-c` | 새로운 아카이브 생성 |
| `-x` | 아카이브 해제 |
| `-t` | 아카이브 내부 목록 확인 |
| `-v` | 작업 과정 출력 |
| `-f` | 아카이브 파일명 지정 |
| `-z` | gzip 사용 |
| `-j` | bzip2 사용 |
| `-C` | 지정한 디렉터리에서 작업 |

---

## 10. 주요 명령어 정리

| 명령어 | 기능 |
| --- | --- |
| `gzip file` | gzip 압축 |
| `gunzip file.gz` | gzip 압축 해제 |
| `bzip2 file` | bzip2 압축 |
| `bunzip2 file.bz2` | bzip2 압축 해제 |
| `xz file` | xz 압축 |
| `unxz file.xz` | xz 압축 해제 |
| `tar -cvf archive.tar files` | 여러 파일을 하나의 tar 아카이브로 묶기 |
| `tar -tvf archive.tar` | tar 내부 목록 확인 |
| `tar -czvf archive.tar.gz files` | tar + gzip 압축 |
| `tar -tzvf archive.tar.gz` | tar.gz 내부 목록 확인 |
| `tar -xzvf archive.tar.gz` | tar.gz 압축 해제 |
| `tar -cjvf archive.tar.bz2 files` | tar + bzip2 압축 |
| `tar -tjvf archive.tar.bz2` | tar.bz2 내부 목록 확인 |
| `tar -xjvf archive.tar.bz2` | tar.bz2 압축 해제 |
| `tar ... -C directory` | 지정한 디렉터리에 복원 |

---

## 11. 압축과 아카이브의 차이

```text
gzip / bzip2 / xz
→ 파일의 용량을 줄이는 압축

tar
→ 여러 파일과 디렉터리를 하나의 파일로 묶는 아카이브

tar + gzip
→ 여러 파일을 하나로 묶은 뒤 gzip 압축

tar + bzip2
→ 여러 파일을 하나로 묶은 뒤 bzip2 압축
```

---

## 실습 결과

`gzip`, `bzip2`, `xz`를 이용하여 파일을 압축하고 다시 원본 파일로 복원하는 과정을 확인하였다.

`tar`를 이용하여 여러 파일을 하나의 아카이브로 묶었으며, 아카이브 생성 후에도 원본 파일이 유지되는 것을 확인하였다.

또한 `tar`와 `gzip`, `bzip2`를 함께 사용하여 여러 파일을 하나의 압축 파일로 생성하고,
`-t` 옵션으로 압축을 해제하지 않고 내부 파일 목록을 확인하였다.

마지막으로 `-C` 옵션을 이용하여 압축 파일을 원하는 디렉터리에 복원하는 방법을 확인하였다.
