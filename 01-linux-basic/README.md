# Rocky Linux 기본 명령어 실습

## 실습 내용

Rocky Linux의 기본적인 명령어를 사용하여 파일과 디렉터리를 관리하는 방법을 실습

## 학습  명령어
| 명령어 | 용도 | 사용 예시 | 설명 |
|---|---|---|---|
| `pwd` | 현재 위치 확인 | `pwd` | 현재 작업 중인 디렉터리의 절대 경로 확인 |
| `cd` | 디렉터리 이동 | `cd /var/log` | `/var/log` 디렉터리로 이동 |
| `ls` | 목록 확인 | `ls -al` | 숨김 파일을 포함한 상세 목록 확인 |
| `mkdir` | 디렉터리 생성 | `mkdir test` | `test` 디렉터리 생성 |
| `rmdir` | 빈 디렉터리 삭제 | `rmdir test` | 비어 있는 `test` 디렉터리 삭제 |
| `touch` | 파일 생성 | `touch test.txt` | 빈 파일 생성 또는 수정 시간 갱신 |
| `cp` | 복사 | `cp file1 file2` | `file1`을 `file2`로 복사 |
| `mv` | 이동/이름 변경 | `mv old.txt new.txt` | 파일 이름을 변경 |
| `rm` | 삭제 | `rm test.txt` | `test.txt` 파일 삭제 |
| `cat` | 파일 내용 확인 | `cat test.txt` | 파일 전체 내용 출력 |
| `head` | 파일 앞부분 확인 | `head test.txt` | 파일의 처음 10줄 출력 |
| `tail` | 파일 뒷부분 확인 | `tail test.txt` | 파일의 마지막 10줄 출력 |
| `less` | 파일 내용 탐색 | `less test.txt` | 긴 파일을 위아래로 이동하며 확인 |
| `find` | 파일 검색 | `find . -name "*.txt"` | 현재 위치 이하의 `.txt` 파일 검색 |
| `grep` | 내용 검색 | `grep "error" test.log` | 파일에서 `error`가 포함된 줄 검색 |

## 실습

### 1. 실습 디렉터리 생성

```bash
[root@localhost ~]# mkdir -p /test/linux-basic
[root@localhost ~]# cd /test/linux-basic/
[root@localhost linux-basic]#
```

`mkdir -p`를 사용하여 `/test/linux-basic` 디렉터리를 생성하고
해당 디렉터리로 이동

---

### 2. 파일 생성

```bash
[root@localhost linux-basic]# touch test.txt test2.txt test3.txt
[root@localhost linux-basic]# ls -l
합계 0
-rw-r--r--. 1 root root 0  9월 12 21:59 test.txt
-rw-r--r--. 1 root root 0  9월 12 21:59 test2.txt
-rw-r--r--. 1 root root 0  9월 12 21:59 test3.txt
```

`touch`를 사용하여 3개의 빈 파일을 생성하고
`ls -l`을 통해 파일이 생성된 것을 확인

---

### 3. 와일드카드를 이용한 파일 복사

```bash
[root@localhost linux-basic]# mkdir backup
[root@localhost linux-basic]# cp *.txt backup/
[root@localhost linux-basic]# ls -l backup/
합계 0
-rw-r--r--. 1 root root 0  9월 12 22:00 test.txt
-rw-r--r--. 1 root root 0  9월 12 22:00 test2.txt
-rw-r--r--. 1 root root 0  9월 12 22:00 test3.txt
```

`*.txt` 와일드카드를 사용하여 `.txt`로 끝나는 파일을
`backup` 디렉터리에 한 번에 복사

---

### 4. 파일 검색

```bash
[root@localhost linux-basic]# find . -type f -name "*.txt"
./test.txt
./test2.txt
./test3.txt
./backup/test.txt
./backup/test2.txt
./backup/test3.txt
```

현재 디렉터리(`.`)부터 하위 디렉터리까지 검색하여
`.txt` 확장자를 가진 일반 파일을 확인

`backup` 디렉터리에 복사한 파일도 검색 결과에 포함되는 것을 확인

---
## 실습 결과

- `mkdir`, `touch`를 사용하여 디렉터리와 파일을 생성
- `*.txt` 와일드카드를 사용하여 여러 텍스트 파일을 한 번에 생성
- `cp`를 사용하여 여러 파일을 `backup` 디렉터리에 복사
- `find`를 사용하여 현재 디렉터리와 하위 디렉터리의 `.txt` 파일 검색
