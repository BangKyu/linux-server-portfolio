# Process / CPU / Memory Resource Monitoring

Rocky Linux Server에서 Process, CPU, Memory, Swap 및 Disk I/O 상태를 확인하고 실제로 부하를 발생시켜 System Resource 변화를 분석하였다.

단순히 명령어 사용법만 확인하는 것이 아니라 다음 흐름으로 실습하였다.

```text
정상 상태 확인
        ↓
CPU 부하 발생
        ↓
문제 Process 확인
        ↓
Process 종료 후 회복 확인
        ↓
Memory 부하 발생
        ↓
RSS / VSZ / %MEM 확인
        ↓
Memory 회복 확인
        ↓
Process 상태 R / S / Z 확인
        ↓
Zombie Process 생성 / 정리
        ↓
Background / Foreground Job 제어
        ↓
nohup Process 유지
        ↓
Disk Write 부하 발생
        ↓
iostat으로 Disk I/O 분석
```

---

# 1. 실습 환경

| 구분 | 내용 |
|---|---|
| Server | Server-A |
| IP | 192.168.111.100 |
| OS | Rocky Linux |
| CPU | 2 vCPU |
| Memory | 약 3.5 GiB |
| Swap | 4.0 GiB |
| Shell | Bash |

---

# CPU 정보 확인

## 2. CPU 개수 확인

```bash
nproc
```

실제 결과:

```text
2
```

Server-A에는 2개의 논리 CPU가 할당되어 있다.

---

## 3. CPU 상세 정보

```bash
lscpu | grep -E '^CPU\(s\):|^Model name:|^Thread\(s\) per core:|^Core\(s\) per socket:|^Socket\(s\):'
```

실제:

```text
CPU(s):                                  2
Model name:                              12th Gen Intel(R) Core(TM) i5-12450H
Thread(s) per core:                      1
Core(s) per socket:                      1
Socket(s):                               2
```

---

# 정상 상태 Baseline

## 4. uptime

```bash
uptime
```

초기 결과:

```text
20:46:30 up 2 min, 2 users, load average: 0.23, 0.22, 0.09
```

현재 System의:

```text
Uptime
Login User 수
Load Average
```

를 확인하였다.

---

# Memory 상태 확인

## 5. free -h

```bash
free -h
```

초기 결과:

```text
               total        used        free      shared  buff/cache   available
Mem:           3.5Gi       961Mi       2.4Gi        15Mi       451Mi       2.6Gi
Swap:          4.0Gi          0B       4.0Gi
```

초기 상태:

```text
Memory Total
→ 3.5 GiB

Memory Available
→ 약 2.6 GiB

Swap Total
→ 4.0 GiB

Swap Used
→ 0 B
```

Memory와 Swap 모두 여유가 있는 상태였다.

---

# Swap 확인

## 6. swapon

```bash
swapon --show
```

실제:

```text
NAME      TYPE      SIZE USED PRIO
/dev/sda1 partition   4G   0B   -2
```

4 GiB Swap Partition이 활성화되어 있지만 실제 사용량은 0 B였다.

---

# Process 확인

## 7. ps -ef

```bash
ps -ef | head
```

Process의:

```text
UID
PID
PPID
실행 시간
Command
```

등을 확인하였다.

---

## 8. CPU 사용 Process 정렬

```bash
ps -eo pid,ppid,user,stat,ni,%cpu,%mem,comm --sort=-%cpu | head -n 10
```

초기에는 높은 CPU를 지속적으로 사용하는 Process가 없었다.

---

## 9. Memory 사용 Process 정렬

```bash
ps -eo pid,ppid,user,stat,ni,%cpu,%mem,comm --sort=-%mem | head -n 10
```

초기에는 `gnome-shell` 등이 상대적으로 많은 Memory를 사용하고 있었다.

---

# vmstat Baseline

## 10. vmstat

```bash
vmstat 1 5
```

초기 주요 결과:

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 2511816   2072 460884    0    0  1621    75  354  871  2  7 91  0  0
 0  0      0 2511816   2072 461064    0    0     0     0  299  293  0  1 99  0  0
 0  0      0 2511592   2072 461064    0    0     0     0  196  227  0  2 98  0  0
 1  0      0 2511592   2072 461064    0    0     0     0   91  173  0  0 100  0  0
 0  0      0 2511592   2072 461064    0    0     0     0   91  164  0  1 99  0  0
```

정상 상태에서:

```text
r
→ 거의 0

b
→ 0

si / so
→ 0

CPU idle
→ 약 98~100%

I/O Wait
→ 0
```

으로 Resource 사용량이 매우 낮았다.

---

# CPU 부하 실습

## 11. CPU 부하 생성

```bash
yes > /dev/null &
```

`yes` 명령이 계속해서 Data를 생성하고 `/dev/null`로 버리도록 하여 CPU 부하를 발생시켰다.

---

## 12. top에서 CPU Process 확인

실제 주요 결과:

```text
top - 20:47:39 up 3 min, 2 users, load average: 0.36, 0.23, 0.10

%Cpu(s): 19.2 us, 26.9 sy, 0.0 ni, 53.8 id, 0.0 wa

PID  USER  PR  NI  VIRT    RES   SHR  S  %CPU  %MEM COMMAND
3020 root  20   0  220968  1956  1844 R 100.0   0.1 yes
```

`yes` Process:

```text
PID
→ 3020

STAT
→ R

%CPU
→ 100.0
```

를 확인하였다.

---

# 2 CPU 환경과 Process %CPU

## 13. Process 100%와 전체 CPU

Server-A에는 CPU가 2개 존재한다.

따라서 하나의 Process가 CPU 하나를 100% 사용해도 전체 System CPU가 모두 100% 사용되는 것은 아니다.

```text
CPU 1
→ yes Process가 거의 100% 사용

CPU 2
→ 대부분 Idle

전체 CPU
→ 약 절반 수준 사용
```

실제 `top`에서도:

```text
idle
→ 53.8%
```

가 확인되었다.

---

# User CPU / System CPU

## 14. CPU 상태

CPU 부하 발생 시:

```text
us
→ 19.2%

sy
→ 26.9%

id
→ 53.8%
```

가 확인되었다.

`yes > /dev/null`은 반복적으로 System Call을 사용하기 때문에 User CPU뿐 아니라 System CPU 사용량도 증가하였다.

---

# CPU 부하 상태 vmstat

## 15. vmstat 변화

CPU 부하 중:

```text
 r  b   si   so   us  sy  id  wa
 1  0    0    0   19  25  56   0
 1  0    0    0   17  26  57   0
 1  0    0    0   18  25  58   0
```

정상 상태의:

```text
id
→ 약 98~100%
```

에서:

```text
id
→ 약 56~58%
```

로 감소하였다.

또한:

```text
r = 1
```

로 실행 가능한 Process가 존재하는 것을 확인하였다.

---

# CPU Process 종료 후 회복

## 16. 부하 제거

CPU 부하 Process를 종료한 뒤:

```bash
vmstat 1 3
```

실제:

```text
 r  b   swpd   free   si   so   us sy id wa
 0  0      0 2522868    0    0    1  1 98  0
 0  0      0 2522868    0    0    0  2 98  0
```

CPU Idle이 다시:

```text
98%
```

수준으로 회복되었다.

---

# CPU 실습 결과

```text
정상
CPU Idle 98~100%
        ↓
yes Process 실행
        ↓
Process %CPU 100%
        ↓
전체 CPU Idle 약 56~58%
        ↓
Process 종료
        ↓
CPU Idle 98%
```

---

# Memory 부하 실습

## 17. Memory Baseline

Memory 부하 전:

```bash
free -h
```

실제:

```text
Mem: 3.5Gi  1.0Gi  2.3Gi  15Mi  483Mi  2.5Gi
Swap: 4.0Gi 0B     4.0Gi
```

---

## 18. 512 MiB Memory 할당

Python을 이용하여 약 512 MiB Memory를 할당하고 일정 시간 유지하였다.

```bash
python3 -c 'import time; x=bytearray(512*1024*1024); time.sleep(300)' &
```

실제 Job:

```text
[1]+ 3231 실행중 python3 ...
```

---

# Memory Process 분석

## 19. RSS / VSZ / %MEM

```bash
ps -p "$MEM_PID" -o pid,ppid,user,stat,ni,rss,vsz,pcpu,pmem,cmd
```

실제:

```text
PID   PPID USER STAT NI   RSS    VSZ    %CPU %MEM
3231  3193 root S     0 531944 752096   1.4 14.3
```

주요 값:

```text
RSS
→ 531944 KB
→ 실제 Physical Memory에 약 519 MiB 존재

VSZ
→ 752096 KB
→ Process의 Virtual Address Space

%MEM
→ 14.3%
```

---

# Memory 사용량 변화

## 20. free -h

512 MiB 할당 후:

```text
               total        used        free      shared  buff/cache   available
Mem:           3.5Gi       1.5Gi       1.8Gi        15Mi       484Mi       2.0Gi
Swap:          4.0Gi          0B       4.0Gi
```

부하 전:

```text
used
→ 약 1.0 GiB

available
→ 약 2.5 GiB
```

부하 후:

```text
used
→ 약 1.5 GiB

available
→ 약 2.0 GiB
```

로 변화하였다.

---

# Memory 사용 Process 정렬

## 21. RSS 기준 Process 확인

```bash
ps -eo pid,ppid,user,stat,rss,vsz,pcpu,pmem,comm --sort=-rss | head -n 10
```

실제:

```text
PID   PPID USER STAT   RSS    VSZ    %CPU %MEM COMMAND
3231  3193 root S    531944 752096   1.4 14.3 python3
1818  1702 gdm  Sl+  199332 ...      0.1  5.3 gnome-shell
```

`python3`가 가장 많은 Physical Memory를 사용하는 Process로 확인되었다.

---

# Memory 부하 중 Swap

## 22. vmstat

```text
swpd
→ 0

si
→ 0

so
→ 0
```

Memory 사용량은 증가했지만 RAM이 충분했기 때문에 Swap은 발생하지 않았다.

---

# Memory Process 상태

## 23. STAT S

Memory를 할당한 Python Process는:

```text
STAT
→ S
```

였다.

Python Code가 Memory를 확보한 뒤:

```python
time.sleep(300)
```

상태였기 때문에 Process가 Sleeping 상태였다.

---

# Process 종료 후 Memory 회복

## 24. Memory 회복

Memory Process 종료 후:

```bash
vmstat 1 3
```

실제:

```text
r  b  swpd    free     si so us sy id wa
0  0     0 2448180     0  0  0  1 99  0
0  0     0 2448180     0  0  0  1 99  0
```

Memory가 다시 회복되고 Swap은 계속 사용되지 않았다.

---

# Linux Memory 확인 시 주의

## 25. free와 available

Linux에서는 단순히:

```text
free
```

값만 보고 Memory 부족을 판단하지 않는다.

주요 항목:

```text
free
→ 현재 완전히 사용하지 않는 RAM

buff/cache
→ Cache 등에 사용하는 Memory

available
→ 새로운 Process가 비교적 무리 없이 사용할 수 있다고 추정되는 Memory
```

따라서 System Memory 상태를 판단할 때:

```text
available
```

도 함께 확인해야 한다.

---

# Process 상태

## 26. R / S / Z

이번 실습에서 실제로 다음 상태를 확인하였다.

```text
R
→ Running / Runnable

S
→ Sleeping

Z
→ Zombie
```

---

# R 상태

## 27. CPU 부하 Process

`yes` Process:

```text
STAT
→ R
```

CPU에서 실행 중이거나 실행 가능한 상태였다.

---

# S 상태

## 28. sleep Process

Memory를 할당한 Python Process:

```text
STAT
→ S
```

Memory는 보유하고 있지만 `sleep()` 상태이므로 CPU를 계속 사용하는 것은 아니었다.

---

# Zombie Process

## 29. Zombie 생성

Python `fork()`를 이용하여 Child가 먼저 종료되고 Parent가 Child의 종료 상태를 회수하지 않도록 하였다.

```bash
python3 -c 'import os,time; pid=os.fork(); os._exit(0) if pid==0 else time.sleep(300)' &
```

Parent PID:

```text
3252
```

---

## 30. Parent Process

```bash
ps -p "$ZPARENT" -o pid,ppid,stat,cmd
```

실제:

```text
PID   PPID STAT CMD
3252  3193 S    python3 ...
```

---

## 31. Zombie Child

```bash
ps --ppid "$ZPARENT" -o pid,ppid,stat,cmd
```

실제:

```text
PID   PPID STAT CMD
3254  3252 Z    [python3] <defunct>
```

Zombie 상태:

```text
STAT
→ Z

CMD
→ <defunct>
```

를 직접 확인하였다.

---

# Zombie 동작 원리

## 32. 구조

```text
Parent Process
PID 3252
        ↓
Child Process
PID 3254
        ↓
Child 실행 종료
        ↓
Parent가 종료 상태를 아직 회수하지 않음
        ↓
Zombie
STAT Z
```

Zombie는 이미 실행이 끝난 Process이므로 CPU를 계속 사용하는 Process가 아니다.

---

# Zombie 정리

## 33. Parent 종료

```bash
kill -15 "$ZPARENT"
```

실제:

```text
[1]+ 종료됨 python3 ...
```

이후:

```bash
ps -p "$ZPARENT"
ps -p "$ZCHILD"
```

결과에서 Parent와 Zombie Child 모두 사라진 것을 확인하였다.

---

# PID / PPID

## 34. 현재 Shell PID

```bash
echo $$
```

실제:

```text
3193
```

현재 Bash의 PID가 `3193`이었다.

---

## 35. Process Tree

```bash
pstree -p $$
```

`sleep` Background Process 실행 후:

```text
bash(3193)─┬─pstree(3284)
           └─sleep(3283)
```

구조:

```text
bash
PID 3193
   ↓
sleep
PID 3283
```

Parent / Child 관계를 확인하였다.

---

# Background Job

## 36. &

```bash
sleep 300 &
```

실제:

```text
[1] 3283
```

Job 확인:

```bash
jobs -l
```

실제:

```text
[1]+ 3283 실행중 sleep 300 &
```

`&`를 이용하여 Process를 Background에서 실행하였다.

---

# Foreground

## 37. fg

```bash
fg %1
```

Background Job을 Foreground로 가져왔다.

---

# Ctrl + Z

## 38. 일시 정지

Foreground 상태에서:

```text
Ctrl + Z
```

실행.

실제:

```text
[1]+ 멈춤 sleep 300
```

`jobs -l`:

```text
[1]+ 3283 정지됨 sleep 300
```

Process가 종료된 것이 아니라 일시정지된 상태이다.

---

# bg

## 39. Background 재실행

```bash
bg %1
```

실제:

```text
[1]+ sleep 300 &
```

확인:

```text
[1]+ 3283 실행중 sleep 300 &
```

일시 정지된 Job을 Background에서 다시 실행하였다.

---

# Ctrl + C

## 40. Process 종료

다시:

```bash
fg %1
```

로 Foreground로 가져온 후:

```text
Ctrl + C
```

를 실행하였다.

이후:

```bash
jobs -l
```

결과가 비어 있어 Process가 종료된 것을 확인하였다.

---

# Job 제어 정리

```text
command &
→ Background 실행

jobs
→ 현재 Shell Job 확인

fg
→ Foreground로 이동

Ctrl + Z
→ 일시 정지

bg
→ Background에서 다시 실행

Ctrl + C
→ Foreground Process 종료
```

---

# nohup

## 41. SSH Session 종료 후 Process 유지

다음 Process를 실행하였다.

```bash
nohup sleep 600 > /tmp/nohup-test.log 2>&1 &
```

PID:

```text
3285
```

PID를 File에 저장:

```bash
echo "$NOHUP_PID" > /tmp/nohup-test.pid
```

---

# SSH Session 종료

## 42. Session 종료 후 재접속

SSH Session을 완전히 종료하고 Server-A에 다시 접속하였다.

재접속 후:

```bash
cat /tmp/nohup-test.pid
```

실제:

```text
3285
```

---

## 43. nohup Process 확인

```bash
ps -p "$(cat /tmp/nohup-test.pid)" -o pid,ppid,user,stat,cmd
```

실제:

```text
PID   PPID USER STAT CMD
3285     1 root S    sleep 600
```

SSH Session이 종료되었음에도 `sleep 600` Process가 계속 실행되고 있었다.

---

# PPID 변화

## 44. PPID 1

재접속 후 Process:

```text
PPID
→ 1
```

원래 Process를 실행했던 Shell이 종료된 뒤 Parent 관계가 변경된 것을 확인하였다.

현재 System의 PID 1은 `systemd`이다.

---

# nohup 구조

```text
SSH Session
        ↓
bash
        ↓
nohup sleep 600
        ↓
SSH Session 종료
        ↓
bash 종료
        ↓
sleep은 계속 실행
        ↓
PPID 1
```

`nohup`은 장시간 실행되는 Command를 Terminal / SSH Session 종료의 영향으로부터 보호할 때 사용할 수 있다.

---

# Disk I/O Monitoring

## 45. sysstat / iostat

`iostat` 사용을 위해 `sysstat` Package를 설치하였다.

```bash
dnf -y install sysstat
```

초기에는:

```text
sysstat 패키지가 설치되어 있지 않습니다
no iostat
```

상태였으며 설치 후 `iostat`을 사용할 수 있었다.

---

# iostat Baseline

## 46. 정상 Disk 상태

```bash
iostat -xz 1 3
```

정상 상태에서 interval 구간의 CPU:

```text
%user    0.00
%system  1.50
%iowait  0.00
%idle   98.50
```

다음 구간:

```text
%iowait
→ 0.00

%idle
→ 98~99%
```

로 Disk I/O 부하가 거의 없었다.

---

# iostat 첫 번째 출력

## 47. 누적 평균

`iostat -xz 1 3`의 첫 번째 출력은 부팅 이후 누적 통계 성격을 가진다.

그 이후 출력은 지정한:

```text
1초 간격
```

의 현재 상태를 보여준다.

따라서 실시간 상태 분석 시 두 번째 이후 구간도 함께 확인하였다.

---

# -z Option

## 48. Zero Activity Device 생략

```bash
iostat -xz
```

의 `-z`는 해당 측정 구간에 Activity가 없는 Device를 출력에서 생략한다.

정상 상태에서는 후속 구간에 Device가 거의 표시되지 않았다.

---

# Disk Write 부하

## 49. dd Write Test

Disk Write 부하를 발생시키기 위해:

```bash
( for i in {1..10}; do
    dd if=/dev/zero of=/tmp/iostat-test.bin bs=1M count=512 oflag=direct status=none
done ) &
```

를 실행하였다.

512 MiB File을 반복적으로 Write하면서 Disk Activity를 관찰하였다.

---

# iostat Write 결과

## 50. Write 부하 발생

부하 발생 시 CPU:

```text
%user     0.00
%system  31.50
%iowait   2.50
%idle    66.00
```

Disk `sda`:

```text
w/s
→ 1535.64

wkB/s
→ 782753.47

w_await
→ 0.69 ms

aqu-sz
→ 1.06

%util
→ 58.51
```

실제 Write 부하가 `sda`에서 크게 증가한 것을 확인하였다.

---

# iostat 주요 항목

## 51. w/s

```text
w/s
```

초당 Write Request 수.

이번 결과:

```text
1535.64
```

---

## 52. wkB/s

```text
wkB/s
```

초당 Write된 Data 양.

이번 결과:

```text
782753.47 KB/s
```

---

## 53. w_await

```text
w_await
```

Write Request가 완료될 때까지 걸린 평균 시간.

이번:

```text
0.69 ms
```

---

## 54. aqu-sz

```text
aqu-sz
```

평균 I/O Queue 크기.

이번:

```text
1.06
```

---

## 55. %util

```text
%util
```

해당 측정 구간에 Device가 I/O 작업을 처리하느라 바빴던 정도를 나타내는 지표이다.

이번:

```text
58.51%
```

---

# I/O 부하 종료 후

## 56. Disk 상태 회복

Write가 종료된 이후:

```text
%idle
→ 약 98~100%

%iowait
→ 0

Device 출력
→ 없음
```

으로 다시 정상 상태에 가까워졌다.

---

# Disk 병목 판단

## 57. 한 값만 보면 안 된다

Disk 병목은 단순히 `%util` 하나만 보고 판단하지 않는다.

다음 지표를 함께 확인한다.

```text
%iowait
await
aqu-sz
%util
Read / Write 처리량
Application 응답 상태
```

예:

```text
%iowait 지속적으로 높음
        +
await 증가
        +
Queue 증가
        +
Device 높은 Busy 상태
        ↓
Disk I/O 병목 의심
```

이번 실습에서는 Write Activity가 명확히 증가했지만 심각한 Disk 병목 상황은 아니었다.

---

# vmstat 주요 항목

## 58. Process

```text
r
→ CPU에서 실행 중이거나 실행을 기다리는 Process

b
→ I/O 등의 이유로 Block된 Process
```

---

## 59. Memory

```text
swpd
→ 사용 중인 Swap 양

free
→ Free Memory

buff
→ Buffer Memory

cache
→ Cache Memory
```

---

## 60. Swap

```text
si
→ Swap In

so
→ Swap Out
```

이번 실습에서는:

```text
si = 0
so = 0
```

으로 Swap Activity가 발생하지 않았다.

---

## 61. CPU

```text
us
→ User Space CPU

sy
→ Kernel / System CPU

id
→ Idle CPU

wa
→ I/O Wait
```

---

# top 주요 항목

## 62. Process 정보

```text
PID
→ Process ID

USER
→ 실행 User

PR
→ Priority

NI
→ Nice 값

VIRT
→ Virtual Memory

RES
→ 실제 Physical Memory

SHR
→ Shared Memory

S
→ Process 상태

%CPU
→ CPU 사용률

%MEM
→ Memory 사용률

COMMAND
→ 실행 Command
```

---

# Load Average

## 63. Load Average 확인

```bash
uptime
```

예:

```text
load average: 0.36, 0.23, 0.10
```

순서:

```text
1분
5분
15분
```

평균 Load를 나타낸다.

Load Average는 단순 CPU 사용률과 동일한 값이 아니다.

CPU에서 실행되기를 기다리는 작업과 특정 Uninterruptible 상태의 작업 등이 Load에 영향을 줄 수 있다.

---

# Process와 Job의 차이

## 64. Process

Kernel이 관리하는 실행 단위이다.

```text
PID
```

로 식별한다.

예:

```text
PID 3283
```

---

## 65. Job

현재 Shell이 관리하는 작업 단위이다.

```text
%1
%2
```

형태의 Job 번호를 사용할 수 있다.

예:

```bash
fg %1
bg %1
```

---

# Signal

## 66. Ctrl + Z

```text
Ctrl + Z
→ SIGTSTP
→ Process 일시 정지
```

Process는 살아 있으므로 `bg` 또는 `fg`로 다시 실행할 수 있다.

---

## 67. Ctrl + C

```text
Ctrl + C
→ SIGINT
→ Foreground Process에 Interrupt 전달
```

일반적인 Command는 이를 받아 종료된다.

---

## 68. kill

Process에 Signal을 전달할 때:

```bash
kill PID
```

를 사용할 수 있다.

기본 Signal은 일반적으로 `SIGTERM`이다.

명시적으로:

```bash
kill -15 PID
```

형태로 사용할 수도 있다.

이번 Zombie Parent Process 정리에도 `SIGTERM`을 사용하였다.

---

# Troubleshooting 기본 흐름

## 69. 서버가 느릴 때

단순히 Server가 느리다고 바로 재부팅하지 않고 Resource 상태를 확인한다.

```text
1. uptime
   ↓
2. top
   ↓
3. CPU 사용 Process 확인
   ↓
4. free -h
   ↓
5. Memory 사용 Process 확인
   ↓
6. vmstat
   ↓
7. iostat
   ↓
8. PID 확인
   ↓
9. 원인 Process / Service 분석
   ↓
10. 필요한 조치 수행
```

---

# CPU 문제 확인

## 70. 확인 예

```bash
uptime

top

ps -eo pid,ppid,user,stat,pcpu,pmem,comm --sort=-pcpu | head
```

확인:

```text
Load Average
CPU Idle
%CPU
Process PID
```

---

# Memory 문제 확인

## 71. 확인 예

```bash
free -h

ps -eo pid,ppid,user,stat,rss,vsz,pcpu,pmem,comm --sort=-rss | head
```

확인:

```text
available
Swap
RSS
%MEM
```

---

# Swap 문제 확인

## 72. 확인

```bash
free -h
vmstat 1 5
```

특히:

```text
si
so
```

가 지속적으로 발생하는지 확인한다.

Swap을 사용하고 있다는 사실 하나만으로 바로 장애라고 판단하지 않고, 실제 Memory 압박 및 Swap Activity를 함께 분석해야 한다.

---

# Disk I/O 문제 확인

## 73. 확인

```bash
iostat -xz 1 5
```

확인:

```text
%iowait
r/s
w/s
rkB/s
wkB/s
await
aqu-sz
%util
```

---

# Zombie 확인

## 74. Z 상태 검색

```bash
ps -eo pid,ppid,stat,cmd | grep '[d]efunct'
```

또는 Process 상태를 확인하여:

```text
STAT Z
```

를 찾을 수 있다.

Zombie 자체는 이미 실행이 종료된 상태이므로 Parent Process가 Child 종료 상태를 정상적으로 회수하는지 확인해야 한다.

---

# 주요 명령어

## 75. Process

```bash
ps -ef
ps aux
ps -eo ...
pgrep
pstree
```

---

## 76. CPU

```bash
nproc
lscpu
uptime
top
vmstat
```

---

## 77. Memory

```bash
free -h
ps --sort=-rss
vmstat
```

---

## 78. Job Control

```bash
jobs
fg
bg
```

Keyboard:

```text
Ctrl + Z
Ctrl + C
```

---

## 79. Background

```bash
command &
```

---

## 80. Session과 분리

```bash
nohup command &
```

---

## 81. Disk I/O

```bash
iostat -xz 1 5
```

---

# 실습 결과

```text
System Baseline
✓ CPU 2 vCPU 확인
✓ Memory 3.5 GiB 확인
✓ Swap 4 GiB 확인
✓ Load Average 확인
✓ vmstat 정상 상태 확인

CPU
✓ yes를 이용한 CPU 부하 생성
✓ Process %CPU 100% 확인
✓ STAT R 확인
✓ 전체 CPU Idle 감소 확인
✓ vmstat CPU 변화 확인
✓ Process 종료 후 CPU 회복 확인

Memory
✓ Python으로 512 MiB Memory 할당
✓ RSS 약 519 MiB 확인
✓ VSZ 확인
✓ %MEM 14.3% 확인
✓ used Memory 증가 확인
✓ available Memory 감소 확인
✓ Swap 사용 없음 확인
✓ Process 종료 후 Memory 회복 확인

Process State
✓ R 상태 확인
✓ S 상태 확인
✓ Z 상태 확인

Zombie
✓ fork를 이용한 Zombie 생성
✓ Parent PID 확인
✓ Child PPID 확인
✓ STAT Z 확인
✓ <defunct> 확인
✓ Parent 종료 후 Zombie 정리 확인

Job Control
✓ Background 실행
✓ jobs 확인
✓ fg 사용
✓ Ctrl + Z 일시 정지
✓ bg 재실행
✓ Ctrl + C 종료
✓ pstree로 Parent / Child 구조 확인

nohup
✓ nohup Process 실행
✓ SSH Session 종료
✓ 재접속
✓ Process 생존 확인
✓ PPID 1 확인

Disk I/O
✓ sysstat / iostat 사용
✓ 정상 Disk Baseline 확인
✓ dd Write 부하 발생
✓ sda Write 증가 확인
✓ w/s 증가 확인
✓ wkB/s 증가 확인
✓ w_await 확인
✓ aqu-sz 확인
✓ %util 증가 확인
✓ 부하 종료 후 Disk Idle 확인
```

---

# 최종 구조

```text
                    Server-A
                       |
            -----------------------
            |          |          |
           CPU       Memory     Disk I/O
            |          |          |
           top       free        iostat
            |          |          |
           ps         ps        %util
            |          |        await
         vmstat      vmstat     wkB/s
            |          |          |
            -------- Process -------
                       |
                    PID / PPID
                       |
                   R / S / Z
                       |
                  jobs / fg / bg
                       |
                     nohup
```

---

# 최종 결과

Server-A의 정상 Resource 상태를 먼저 확인한 뒤 CPU, Memory 및 Disk I/O 부하를 직접 발생시켜 Monitoring Command의 값이 어떻게 변화하는지 확인하였다.

CPU 실습에서는:

```bash
yes > /dev/null &
```

을 이용하여 하나의 Process가 약 100% CPU를 사용하도록 하였고:

```text
Process %CPU
→ 100%

전체 CPU Idle
→ 약 56~58%
```

까지 감소하는 것을 확인하였다.

Process 종료 후:

```text
CPU Idle
→ 약 98%
```

로 회복되었다.

Memory 실습에서는 Python으로 약 512 MiB를 할당하여:

```text
RSS
→ 약 519 MiB

%MEM
→ 14.3%

Memory used
→ 약 1.0 GiB → 1.5 GiB

available
→ 약 2.5 GiB → 2.0 GiB
```

변화를 확인하였다.

RAM이 충분했기 때문에:

```text
Swap Used
→ 0 B

si
→ 0

so
→ 0
```

으로 Swap Activity는 발생하지 않았다.

또한 Process 상태를 직접 확인하여:

```text
R
→ Running / Runnable

S
→ Sleeping

Z
→ Zombie
```

를 실습하였다.

Zombie Process에서는:

```text
Parent PID 3252
        ↓
Child PID 3254
STAT Z
<defunct>
```

상태를 확인하고 Parent 종료 후 Zombie가 정리되는 것까지 확인하였다.

Shell Job Control에서는:

```text
&
jobs
fg
Ctrl + Z
bg
Ctrl + C
```

를 이용하여 Background / Foreground Process를 직접 제어하였다.

`nohup` 실습에서는 SSH Session을 종료한 이후에도:

```text
PID 3285
PPID 1
sleep 600
```

Process가 계속 실행되는 것을 확인하였다.

마지막으로 `iostat`과 `dd`를 이용하여 Disk Write 부하를 발생시켰고:

```text
sda

w/s
1535.64

wkB/s
782753.47

w_await
0.69 ms

aqu-sz
1.06

%util
58.51%
```

까지 변화하는 것을 확인하였다.

이번 실습을 통해 Server 장애 분석 시 단순히 한 가지 수치만 보는 것이 아니라:

```text
Process
CPU
Load Average
Memory
Swap
Process State
Disk I/O
```

를 함께 확인하여 병목 위치를 판단하는 기본적인 Resource Monitoring 흐름을 실습하였다.

---

## 관련 이론

Process / PID / PPID, Process State, Zombie, Signal, Load Average, CPU `us/sy/id/wa`, Memory `free/available/cache`, RSS / VSZ, Swap, Job Control, `nohup`, `vmstat`, `iostat` 및 Resource Troubleshooting에 대한 자세한 내용은 `process-resource-notes.md`에서 정리한다.
