# Rocky Linux 기본 명령어 실습

## 실습 개요

Rocky Linux의 기본 명령어를 사용하여 파일과 디렉터리를 생성하고 확인하는 방법을 실습하였다.

파일 생성, 파일 복사, 파일 검색, 와일드카드 사용,리다이렉션 기호, 파일 이동 및 이름 변경 등의 기본적인 리눅스 CLI 작업을 실습하였다.

---

## 실습 과정

### 1. 디렉터리 생성 후 이동

```bash
[root@Server-A ~]# mkdir -p /test/linux-basic
[root@Server-A ~]# cd /test/linux-basic/
[root@Server-A linux-basic]# pwd
/test/linux-basic
```

`mkdir -p`를 사용하여 `/test/linux-basic` 디렉터리 생성

`cd`를 사용하여 `/test/linux-basic` 디렉터리로 이동

`pwd`를 사용하여 현재 작업 디렉터리가 `/test/linux-basic`인지 확인

---

### 2. 파일 생성

```bash
[root@Server-A linux-basic]# touch test1.txt test2.txt test3.txt
[root@Server-A linux-basic]# ls -l
합계 0
-rw-r--r--. 1 root root 0  9월 16 11:36 test1.txt
-rw-r--r--. 1 root root 0  9월 16 11:36 test2.txt
-rw-r--r--. 1 root root 0  9월 16 11:36 test3.txt
```

`touch`를 사용하여 `test1.txt`, `test2.txt`, `test3.txt` 파일 생성

`ls -l`을 사용하여 생성된 파일과 기본 정보 확인

---

### 3. 와일드카드를 이용한 파일 복사

```bash
[root@Server-A linux-basic]# mkdir backup
[root@Server-A linux-basic]# cp *.txt backup/
[root@Server-A linux-basic]# ls -l backup/
합계 0
-rw-r--r--. 1 root root 0  9월 16 11:37 test1.txt
-rw-r--r--. 1 root root 0  9월 16 11:37 test2.txt
-rw-r--r--. 1 root root 0  9월 16 11:37 test3.txt
```

`mkdir`를 사용하여 `backup` 디렉터리 생성

`cp`와 `*.txt` 와일드카드를 사용하여 `.txt` 파일을 한 번에 `backup` 디렉터리로 복사

---

### 4. 파일 검색

```bash
[root@Server-A linux-basic]# find . -type f -name "*.txt"
./test1.txt
./test2.txt
./test3.txt
./backup/test1.txt
./backup/test2.txt
./backup/test3.txt
```

`find .`을 사용하여 현재 디렉터리와 하위 디렉터리 검색

`-type f`를 사용하여 일반 파일만 검색

`-name "*.txt"`를 사용하여 `.txt` 파일만 검색

---

### 5. 파일 내용 작성 및 확인

```bash
[root@Server-A linux-basic]# echo "first" > echo-test.txt
[root@Server-A linux-basic]# ls -l
합계 8
drwxr-xr-x. 2 root root 57  9월 16 11:37 backup
-rw-r--r--. 1 root root  6  9월 16 11:50 echo-test.txt
-rw-r--r--. 1 root root  6  9월 16 11:49 test1.txt
-rw-r--r--. 1 root root  0  9월 16 11:36 test2.txt
-rw-r--r--. 1 root root  0  9월 16 11:36 test3.txt
[root@Server-A linux-basic]# cat echo-test.txt
first
```

`echo`와 `>`를 사용하여 `echo-test.txt` 파일을 생성하고 `first` 내용 저장

```bash
[root@Server-A linux-basic]# echo "second" >> echo-test.txt
[root@Server-A linux-basic]# cat echo-test.txt
first
second
```

`echo`와 `>>`를 사용하여 `echo-test.txt`에 `second` 내용 추가

```bash
[root@Server-A linux-basic]# echo "clear" > echo-test.txt
[root@Server-A linux-basic]# cat echo-test.txt
clear
```

`echo`와 `>`를 사용하여 기존 내용을 `clear`로 덮어쓰기

---

### 6. 여러 파일 내용 합치기

```bash
[root@Server-A linux-basic]# echo "test1" > test1.txt
[root@Server-A linux-basic]# echo "test2" > test2.txt
[root@Server-A linux-basic]# echo "test3" > test3.txt
[root@Server-A linux-basic]# cat test1.txt test2.txt test3.txt
test1
test2
test3
[root@Server-A linux-basic]# cat test1.txt test2.txt test3.txt > all.txt
[root@Server-A linux-basic]# cat all.txt
test1
test2
test3
```

`echo`와 `>`를 사용하여 각 파일에 내용 저장

`cat`을 사용하여 여러 파일의 내용을 연속으로 출력

`cat`과 `>`를 사용하여 세 파일의 내용을 `all.txt` 파일로 저장

---

### 7. 파일 앞부분과 뒷부분 확인

```bash
[root@Server-A linux-basic]# head -n 2 all.txt
test1
test2
```

`head -n 2`를 사용하여 `all.txt` 파일의 처음 2줄 확인

```bash
[root@Server-A linux-basic]# tail -n 2 all.txt
test2
test3
```

`tail -n 2`를 사용하여 `all.txt` 파일의 마지막 2줄 확인

---

### 8. 파일 이동 및 이름 변경

```bash
[root@Server-A linux-basic]# ls -l
합계 20
-rw-r--r--. 1 root root 18  9월 16 11:55 all.txt
drwxr-xr-x. 2 root root 57  9월 16 12:02 backup
-rw-r--r--. 1 root root  6  9월 16 11:51 echo-test.txt
-rw-r--r--. 1 root root  6  9월 16 11:54 test1.txt
-rw-r--r--. 1 root root  6  9월 16 11:54 test2.txt
-rw-r--r--. 1 root root  6  9월 16 11:54 test3.txt

[root@Server-A linux-basic]# ls -l backup/
합계 0
-rw-r--r--. 1 root root 0  9월 16 11:37 test1.txt
-rw-r--r--. 1 root root 0  9월 16 11:37 test2.txt
-rw-r--r--. 1 root root 0  9월 16 11:37 test3.txt
```

`ls -l`을 사용하여 현재 디렉터리와 `backup` 디렉터리의 파일 목록 확인

```bash
[root@Server-A linux-basic]# mv all.txt backup/
[root@Server-A linux-basic]# ls -l backup/
합계 4
-rw-r--r--. 1 root root 18  9월 16 11:55 all.txt
-rw-r--r--. 1 root root  0  9월 16 11:37 test1.txt
-rw-r--r--. 1 root root  0  9월 16 11:37 test2.txt
-rw-r--r--. 1 root root  0  9월 16 11:37 test3.txt
```

`mv`를 사용하여 `all.txt` 파일을 `backup` 디렉터리로 이동

```bash
[root@Server-A linux-basic]# mv backup/all.txt move.txt
[root@Server-A linux-basic]# ls -l
합계 20
drwxr-xr-x. 2 root root 57  9월 16 12:04 backup
-rw-r--r--. 1 root root  6  9월 16 11:51 echo-test.txt
-rw-r--r--. 1 root root 18  9월 16 11:55 move.txt
-rw-r--r--. 1 root root  6  9월 16 11:54 test1.txt
-rw-r--r--. 1 root root  6  9월 16 11:54 test2.txt
-rw-r--r--. 1 root root  6  9월 16 11:54 test3.txt
[root@Server-A linux-basic]# ls -l backup/
합계 0
-rw-r--r--. 1 root root 0  9월 16 11:37 test1.txt
-rw-r--r--. 1 root root 0  9월 16 11:37 test2.txt
-rw-r--r--. 1 root root 0  9월 16 11:37 test3.txt
```

`mv`를 사용하여 `backup/all.txt` 파일을 현재 디렉터리로 이동하면서 `move.txt`로 이름 변경

---

## 실습 결과

- `mkdir -p`, `cd`, `pwd`를 사용하여 실습 디렉터리 생성 및 현재 위치 확인
- `touch`를 사용하여 여러 파일 생성
- `*.txt` 와일드카드와 `cp`를 사용하여 여러 파일을 한 번에 복사
- `find`를 사용하여 조건에 맞는 파일 검색
- `echo`, `리다이렉션 기호`를 사용하여 파일 내용 저장, 추가 및 덮어쓰기
- `cat`, `리다이렉션 기호`를 사용하여 파일 내용을 확인하고 여러 파일의 내용을 하나의 파일로 저장
- `head`, `tail`을 사용하여 파일의 앞부분과 뒷부분 내용을 확인
- `mv`를 사용하여 파일 이동 및 이름 변경
