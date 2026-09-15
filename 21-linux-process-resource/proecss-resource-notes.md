# Process / CPU / Memory Resource 이론 정리

## 1. Process란?

Process는 실행 중인 Program이다.

예를 들어 다음 Command를 실행하면:

```bash
sleep 300
```

`sleep` Program이 실행되면서 하나의 Process가 생성된다.

즉:

```text
Program
→ Disk에 저장된 실행 File

Process
→ Memory에 올라와 실제 실행 중인 Program
```

으로 이해할 수 있다.

---

# PID

## 2. PID란?

PID는 Process ID의 약자이다.

Linux Kernel은 각각의 Process를 PID라는 숫자로 구분한다.

예:

```text
PID 3283
→ sleep Process
```

PID는 Process 관리에서 매우 중요하다.

예:

```bash
ps -ef
kill 3283
```

---

# PPID

## 3. PPID란?

PPID는 Parent Process ID이다.

현재 Process를 생성한 부모 Process의 PID를 의미한다.

예:

```text
bash
PID 3193
   ↓
sleep
PID 3283
PPID 3193
```

즉:

```text
sleep의 PID
→ 3283

sleep의 PPID
→ 3193
```

이다.

---

# Parent / Child Process

## 4. Process 계층

Linux Process는 Parent / Child 관계를 가질 수 있다.

예:

```text
bash
   ↓
sleep
```

Shell에서 Command를 실행하면 Shell이 Child Process를 생성하여 Command를 실행할 수 있다.

이번 실습에서는:

```bash
pstree -p $$
```

결과:

```text
bash(3193)─┬─pstree(3284)
           └─sleep(3283)
```

를 확인하였다.

---

# $$

## 5. 현재 Shell PID

Bash에서:

```bash
echo $$
```

를 사용하면 현재 Shell의 PID를 확인할 수 있다.

이번 실습:

```text
3193
```

이었다.

---

# pstree

## 6. Process Tree

```bash
pstree
```

는 Process의 Parent / Child 관계를 Tree 형태로 보여준다.

PID도 함께 확인:

```bash
pstree -p
```

특정 Shell 기준:

```bash
pstree -p $$
```

---

# ps

## 7. Process 상태 확인

대표적인 Process 확인 Command:

```bash
ps
```

---

## 8. ps -ef

```bash
ps -ef
```

주요 항목:

```text
UID
→ Process 실행 User

PID
→ Process ID

PPID
→ Parent PID

STIME
→ Process 시작 시간

TIME
→ 사용한 CPU 시간

CMD
→ 실행 Command
```

---

## 9. ps aux

```bash
ps aux
```

주요 항목:

```text
USER
PID
%CPU
%MEM
VSZ
RSS
STAT
COMMAND
```

Process Resource 사용량을 확인할 때 자주 사용한다.

---

# Process State

## 10. STAT

`ps`의 `STAT` Column은 Process 상태를 나타낸다.

이번 실습에서 실제로 확인한 상태:

```text
R
S
Z
```

---

# R

## 11. R = Running / Runnable

```text
R
```

은 Process가:

```text
현재 CPU에서 실행 중이거나
CPU에서 실행될 준비가 된 상태
```

를 의미한다.

이번 CPU 부하 실습의 `yes` Process:

```text
STAT
→ R
```

이었다.

---

# S

## 12. S = Sleeping

```text
S
```

는 Process가 Sleep 상태임을 의미한다.

즉 Process가 살아 있지만 어떤 Event나 Timer 등을 기다리고 있는 상태이다.

이번 Memory 실습:

```python
time.sleep(300)
```

을 실행한 Python Process:

```text
STAT
→ S
```

이었다.

---

# Z

## 13. Z = Zombie

```text
Z
```

는 Zombie Process를 의미한다.

Zombie는 실행이 이미 끝났지만 Parent가 종료 상태를 아직 회수하지 않은 Process이다.

---

# Zombie Process

## 14. Zombie 발생 과정

```text
Parent Process
        ↓
Child Process 생성
        ↓
Child 실행 종료
        ↓
Parent가 Child 종료 상태 미회수
        ↓
Zombie
```

이번 실습:

```text
Parent PID
3252

Child PID
3254

Child STAT
Z

CMD
[python3] <defunct>
```

---

## 15. <defunct>

Zombie Process는 다음처럼 표시될 수 있다.

```text
[python3] <defunct>
```

`<defunct>`는 해당 Process의 실행은 끝났지만 Process Table에 종료 정보가 남아 있음을 의미한다.

---

## 16. Zombie는 CPU를 계속 사용하는가?

일반적으로 아니다.

Zombie Process는 이미 실행이 종료된 상태이다.

즉:

```text
Zombie
→ 실행 중인 작업 없음
→ CPU 계산 작업 수행하지 않음
```

하지만 Zombie가 지나치게 많이 쌓이면 Process Table Resource를 소비할 수 있으므로 원인을 확인해야 한다.

---

## 17. Zombie를 kill -9 하면 되는가?

Zombie 자체는 이미 종료된 Process이므로 일반적인 의미에서 다시 죽일 대상이 아니다.

중요한 것은:

```text
Parent가 Child 종료 상태를 회수하지 않는 원인
```

을 찾는 것이다.

Parent가 정상적으로 `wait()` 등의 처리를 수행하면 Zombie가 회수된다.

---

# Process Signal

## 18. Signal이란?

Signal은 Process에게 특정 Event나 명령을 전달하는 Mechanism이다.

예:

```text
종료
일시정지
계속 실행
Interrupt
```

---

# SIGTERM

## 19. SIGTERM

Signal Number:

```text
15
```

이다.

사용:

```bash
kill -15 PID
```

또는 기본 `kill`:

```bash
kill PID
```

일반적으로 Process에게:

```text
정상적으로 종료해 달라
```

고 요청하는 Signal이다.

---

# SIGKILL

## 20. SIGKILL

Signal Number:

```text
9
```

사용:

```bash
kill -9 PID
```

Kernel이 Process를 강제로 종료한다.

Process가 SIGKILL을 무시하거나 처리할 수 없다.

따라서 일반적으로:

```text
SIGTERM 시도
        ↓
정상 종료되지 않음
        ↓
필요할 때 SIGKILL
```

순서가 좋다.

---

# SIGINT

## 21. Ctrl + C

Foreground Process 실행 중:

```text
Ctrl + C
```

를 입력하면 일반적으로:

```text
SIGINT
```

가 전달된다.

많은 Command는 SIGINT를 받아 종료된다.

이번 실습에서도:

```text
fg %1
        ↓
Ctrl + C
        ↓
sleep Process 종료
```

를 확인하였다.

---

# SIGTSTP

## 22. Ctrl + Z

Foreground Process에서:

```text
Ctrl + Z
```

를 입력하면:

```text
SIGTSTP
```

가 전달된다.

Process가 종료되는 것이 아니라:

```text
Stopped
```

상태가 된다.

---

# Ctrl + Z와 Ctrl + C

## 23. 차이

```text
Ctrl + Z
→ 일시 정지
→ Process 살아 있음
→ bg / fg로 다시 실행 가능
```

```text
Ctrl + C
→ Interrupt
→ 일반적으로 Process 종료
→ 다시 bg / fg로 실행할 수 없음
```

---

# Job

## 24. Job이란?

Job은 현재 Shell이 관리하는 작업 단위이다.

Process는 Kernel이 PID로 관리하지만 Shell은 Job Number도 사용한다.

예:

```text
PID
3283

Job Number
%1
```

---

# Background

## 25. &

Command 끝에:

```text
&
```

를 붙이면 Background에서 실행한다.

예:

```bash
sleep 300 &
```

실제:

```text
[1] 3283
```

여기서:

```text
1
→ Job Number

3283
→ PID
```

이다.

---

# jobs

## 26. Shell Job 확인

```bash
jobs
```

또는 PID 포함:

```bash
jobs -l
```

이번 실습:

```text
[1]+ 3283 실행중 sleep 300 &
```

---

# fg

## 27. Foreground로 이동

```bash
fg %1
```

Job Number 1을 Foreground로 가져온다.

Foreground에서는 해당 Process가 종료되거나 정지될 때까지 일반 Shell Prompt가 돌아오지 않는다.

---

# bg

## 28. Background에서 재실행

Ctrl + Z로 일시정지한 Job을:

```bash
bg %1
```

로 Background에서 다시 실행할 수 있다.

이번 흐름:

```text
sleep 300 &
        ↓
fg %1
        ↓
Ctrl + Z
        ↓
Stopped
        ↓
bg %1
        ↓
Running
```

---

# nohup

## 29. nohup이란?

`nohup`은:

```text
no hang up
```

에서 유래한 Command이다.

Terminal 또는 SSH Session 종료 시 발생할 수 있는 SIGHUP의 영향을 받지 않도록 Process를 실행할 때 사용한다.

---

## 30. nohup 사용

이번 실습:

```bash
nohup sleep 600 > /tmp/nohup-test.log 2>&1 &
```

의미:

```text
nohup
→ Session 종료 영향 최소화

sleep 600
→ 실행 Command

> /tmp/nohup-test.log
→ Standard Output 저장

2>&1
→ Standard Error도 같은 곳으로 전달

&
→ Background 실행
```

---

# SSH Session과 Process

## 31. Session 종료

일반적으로 Terminal / Shell과 연결된 Process는 Session 종료의 영향을 받을 수 있다.

`nohup`을 사용하면 장시간 실행 작업을 Session 종료와 분리하여 사용할 수 있다.

이번 실습에서는:

```text
nohup sleep 600 실행
        ↓
SSH Session 종료
        ↓
재접속
        ↓
sleep Process 여전히 실행
```

을 확인하였다.

---

# PPID 1

## 32. Parent 종료 후

Session을 종료한 후:

```text
PID 3285
PPID 1
sleep 600
```

을 확인하였다.

원래 Parent Shell이 종료된 뒤 Process가 살아남으면서 Parent 관계가 변경되었다.

Rocky Linux에서 PID 1은 일반적으로:

```text
systemd
```

이다.

---

# CPU

## 33. CPU란?

CPU는 Process의 명령을 실제로 처리하는 Resource이다.

Linux에서는 여러 Process가 CPU를 공유하여 사용한다.

---

# nproc

## 34. CPU 개수 확인

```bash
nproc
```

이번 환경:

```text
2
```

즉 2개의 논리 CPU가 존재하였다.

---

# lscpu

## 35. CPU 상세 정보

```bash
lscpu
```

확인 가능:

```text
CPU 개수
Architecture
Core
Socket
Thread
CPU Model
```

---

# Process %CPU

## 36. %CPU

```text
%CPU
```

는 특정 Process의 CPU 사용량을 나타낸다.

이번 `yes` Process:

```text
%CPU
→ 100.0
```

이었다.

---

# Process 100%와 System 100%

## 37. 차이

Server-A에는 CPU가 2개 존재한다.

Process 하나가 CPU 하나를 100% 사용한다면:

```text
Process %CPU
→ 약 100%

System 전체
→ CPU 2개 중 1개 중심 사용
```

이 될 수 있다.

따라서:

```text
Process 100%
≠
전체 System CPU 100%
```

이다.

---

# CPU Load 생성

## 38. yes

이번 실습:

```bash
yes > /dev/null &
```

`yes`가 계속 Data를 생성하고 `/dev/null`로 버리면서 CPU를 지속적으로 사용하도록 하였다.

---

# top

## 39. top이란?

`top`은 System Resource와 Process 상태를 실시간으로 확인하는 대표적인 Command이다.

```bash
top
```

주요 영역:

```text
Load Average
Tasks
CPU
Memory
Swap
Process 목록
```

---

# top CPU

## 40. %Cpu(s)

대표 항목:

```text
us
sy
ni
id
wa
hi
si
st
```

---

# us

## 41. User CPU

```text
us
```

User Space Program이 사용한 CPU 비율이다.

예:

```text
Application
Shell Program
User Process
```

등의 실행과 관련된다.

---

# sy

## 42. System CPU

```text
sy
```

Kernel Space에서 사용한 CPU 비율이다.

예:

```text
System Call
Kernel 작업
Device 관련 처리
```

등이 포함된다.

이번 `yes > /dev/null` 실습에서는 `write()` System Call이 반복되어 `sy`도 증가하였다.

---

# id

## 43. Idle CPU

```text
id
```

CPU가 아무 작업도 하지 않고 쉬고 있던 비율이다.

이번 정상 상태:

```text
98~100%
```

CPU 부하 중:

```text
약 56~58%
```

로 감소하였다.

---

# wa

## 44. I/O Wait

```text
wa
```

CPU가 I/O 완료를 기다리는 시간과 관련된 비율이다.

Disk I/O 병목 분석에서 중요한 값이다.

---

# ni

## 45. Nice CPU

```text
ni
```

Nice Priority가 조정된 User Process가 사용한 CPU 시간과 관련된다.

---

# hi / si

## 46. Interrupt

```text
hi
→ Hardware Interrupt

si
→ Software Interrupt
```

처리에 사용된 CPU 비율이다.

---

# st

## 47. Steal Time

```text
st
```

Virtual Machine 환경에서 Hypervisor가 다른 VM에 CPU를 사용하게 하면서 현재 VM이 기다린 시간과 관련된다.

가상화 환경의 CPU 문제 분석에서 참고할 수 있다.

---

# Load Average

## 48. Load Average란?

```bash
uptime
```

또는:

```bash
top
```

에서 확인할 수 있다.

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

이다.

---

## 49. CPU 사용률과 Load Average는 다르다

중요하다.

```text
CPU % 사용률
→ CPU가 실제로 얼마나 바쁜가

Load Average
→ 실행 가능하거나 특정 대기 상태에 있는 Task 수의 평균
```

따라서:

```text
Load Average = CPU 사용률
```

이 아니다.

---

## 50. CPU 개수와 Load 해석

2 CPU System에서 단순하게 생각하면:

```text
Load 1
→ CPU 하나 정도의 작업량

Load 2
→ 두 CPU 정도의 작업량

Load가 2보다 지속적으로 큼
→ CPU 처리 능력보다 많은 Task가 대기할 가능성
```

정도로 볼 수 있다.

하지만 실제 장애 판단에서는 CPU 사용률, Run Queue, I/O Wait 등을 같이 봐야 한다.

---

# vmstat

## 51. vmstat란?

```bash
vmstat
```

는 Virtual Memory Statistics의 약자로:

```text
Process
Memory
Swap
I/O
System
CPU
```

상태를 한 화면에서 확인할 수 있다.

---

## 52. vmstat 1 5

```bash
vmstat 1 5
```

의미:

```text
1초 간격
5회 출력
```

---

## 53. 첫 번째 vmstat 줄

`vmstat`의 첫 번째 통계 행은 부팅 이후 누적 평균 성격을 가진다.

따라서 실시간 상태를 볼 때는 이후 interval 행을 함께 확인하는 것이 좋다.

---

# vmstat procs

## 54. r

```text
r
```

CPU에서 실행 중이거나 실행되기를 기다리는 Runnable Task 수와 관련된다.

CPU 부하 실습에서:

```text
r = 1
```

을 확인하였다.

---

## 55. b

```text
b
```

I/O 등의 이유로 Uninterruptible Sleep 상태에 있는 Process 수와 관련된다.

Disk 병목에서 증가할 수 있다.

---

# vmstat Memory

## 56. swpd

```text
swpd
```

현재 사용 중인 Swap Memory 양이다.

이번 실습:

```text
0
```

이었다.

---

## 57. free

```text
free
```

현재 사용되지 않는 Physical Memory.

---

## 58. buff

```text
buff
```

Buffer로 사용되는 Memory와 관련된다.

---

## 59. cache

```text
cache
```

File Cache 등에 사용되는 Memory와 관련된다.

---

# vmstat Swap

## 60. si

```text
si
```

Swap In.

Disk의 Swap 영역에서 RAM으로 읽어 들인 Memory 양과 관련된다.

---

## 61. so

```text
so
```

Swap Out.

RAM의 Memory Page가 Swap 영역으로 이동하는 것과 관련된다.

---

## 62. Swap Activity

이번 실습에서는:

```text
si = 0
so = 0
```

이었다.

Memory Pressure가 거의 없었음을 확인할 수 있다.

---

# vmstat I/O

## 63. bi

```text
bi
```

Block Device에서 읽은 Data 양과 관련된다.

---

## 64. bo

```text
bo
```

Block Device로 쓴 Data 양과 관련된다.

---

# Memory

## 65. Physical Memory

RAM은 Process와 Kernel이 실행 중 Data를 저장하는 빠른 Memory이다.

이번 Server-A:

```text
약 3.5 GiB
```

였다.

---

# free

## 66. free -h

```bash
free -h
```

대표 항목:

```text
total
used
free
shared
buff/cache
available
```

---

# total

## 67. total

System에서 사용할 수 있는 전체 Physical Memory.

이번:

```text
3.5 GiB
```

---

# used

## 68. used

현재 사용 중인 Memory 양.

단순 Process Memory만을 의미하는 값으로 보면 안 되고 Kernel의 Memory 계산 방식에 따라 구성된다.

---

# free

## 69. free

현재 어떤 용도로도 사용되지 않는 Memory.

Linux에서는 Free Memory를 Cache 등에 적극적으로 활용하기 때문에 `free`가 작다고 바로 Memory 부족으로 판단하면 안 된다.

---

# buff/cache

## 70. Cache Memory

Linux는 사용하지 않는 RAM을 File Cache 등으로 활용하여 성능을 높인다.

필요하면 Cache Memory의 일부를 다시 Application에 사용할 수 있다.

---

# available

## 71. Available Memory

```text
available
```

은 새로운 Application이 사용할 수 있다고 Kernel이 추정하는 Memory 양이다.

System Memory 상태를 볼 때 `free`보다 `available`을 함께 확인하는 것이 중요하다.

---

# Linux는 빈 RAM을 아껴두지 않는다

## 72. Cache 활용

Linux는 사용하지 않는 RAM을 그대로 비워두기보다 Cache로 활용할 수 있다.

따라서:

```text
free가 낮음
```

이라는 이유 하나만으로:

```text
Memory 부족
```

이라고 판단하면 안 된다.

확인:

```text
available
Swap 사용
si / so
OOM 발생 여부
Process RSS
```

등을 함께 확인한다.

---

# RSS

## 73. Resident Set Size

```text
RSS
```

Process의 Memory 중 현재 실제 Physical RAM에 올라와 있는 부분과 관련된다.

이번 Python Process:

```text
RSS
531944 KB
```

약:

```text
519 MiB
```

였다.

Memory 사용 Process를 찾을 때 중요한 지표이다.

---

# VSZ

## 74. Virtual Memory Size

```text
VSZ
```

Process의 Virtual Address Space 크기이다.

이번 Python:

```text
VSZ
752096 KB
```

이었다.

---

# RSS와 VSZ 차이

## 75. 비교

```text
VSZ
→ Process가 사용하는 Virtual Address Space 전체 크기

RSS
→ 그중 실제 Physical RAM에 올라와 있는 부분
```

따라서:

```text
VSZ가 크다고
실제 RAM을 동일한 크기로 사용한다는 뜻은 아님
```

이다.

---

# %MEM

## 76. Memory 사용률

```text
%MEM
```

Process가 전체 Physical Memory에서 차지하는 비율이다.

이번:

```text
14.3%
```

였다.

---

# Memory 부하 실습

## 77. Python Memory Allocation

```bash
python3 -c 'import time; x=bytearray(512*1024*1024); time.sleep(300)' &
```

약 512 MiB Memory를 확보한 후 Process를 Sleep 상태로 유지하였다.

---

## 78. Memory 변화

부하 전:

```text
used
약 1.0 GiB

available
약 2.5 GiB
```

부하 후:

```text
used
약 1.5 GiB

available
약 2.0 GiB
```

로 변화하였다.

---

# Swap

## 79. Swap이란?

Swap은 Physical RAM이 부족한 상황에서 Memory Page 일부를 Disk 영역에 저장하는 Mechanism이다.

구조:

```text
RAM 부족
        ↓
일부 Memory Page
        ↓
Swap Area
Disk
```

---

## 80. Swap은 RAM과 같은가?

아니다.

Disk는 RAM보다 느리므로 Swap 사용이 과도해지면 System 성능이 크게 떨어질 수 있다.

---

## 81. Swap이 조금 사용됐다고 바로 장애인가?

아니다.

과거에 사용하던 Memory Page가 Swap에 남아 있을 수도 있다.

중요한 것은:

```text
현재 Memory Pressure
+
si / so가 지속적으로 발생하는가
+
Application 응답이 느린가
```

등이다.

---

# Swap 확인

## 82. swapon

```bash
swapon --show
```

이번 Server-A:

```text
/dev/sda1
4G
```

Swap Partition이 활성화되어 있었다.

---

# OOM

## 83. Out Of Memory

Physical Memory와 Swap이 부족해져 새로운 Memory 할당이 어려워지는 심각한 상황이 발생할 수 있다.

Kernel은 상황에 따라 OOM Killer를 통해 Process를 종료할 수 있다.

실제 운영에서는:

```text
Memory 부족
        ↓
Swap 급증
        ↓
System 느려짐
        ↓
OOM 발생 가능
```

흐름을 주의해야 한다.

이번 실습에서는 OOM을 발생시키지 않았다.

---

# nice

## 84. nice란?

Nice Value는 CPU Scheduling Priority에 영향을 주는 값이다.

일반적인 범위:

```text
-20
~
19
```

---

## 85. Nice 값 의미

```text
값이 낮을수록
→ CPU Scheduling 우선순위 높음

값이 높을수록
→ 다른 Process에게 CPU를 더 양보
```

기본 Nice Value:

```text
0
```

인 경우가 일반적이다.

---

# renice

## 86. 실행 중 Priority 변경

```bash
renice
```

는 실행 중인 Process의 Nice Value를 변경한다.

예:

```bash
renice 10 -p PID
```

이 경우 해당 Process를 더 낮은 CPU Scheduling Priority로 조정하는 의미이다.

---

## 87. nice는 CPU 사용량 제한 기능이 아니다

매우 중요하다.

```text
nice 10
```

이라고 해서:

```text
CPU 100% → CPU 10%
```

이 되는 것이 아니다.

Nice는 CPU 경쟁이 발생했을 때 Scheduling 우선순위에 영향을 주는 기능이다.

CPU가 충분히 남아 있다면 낮은 Priority Process도 CPU를 많이 사용할 수 있다.

---

# Disk I/O

## 88. I/O란?

Input / Output.

Storage 관점에서는:

```text
Disk Read
Disk Write
```

작업을 의미한다.

Server가 느릴 때 CPU와 Memory가 정상인데 Disk I/O가 병목일 수도 있다.

---

# iostat

## 89. iostat란?

```bash
iostat
```

은 CPU와 Block Device I/O 상태를 확인하는 Command이다.

Rocky Linux에서는 `sysstat` Package에 포함되어 있다.

설치:

```bash
dnf install sysstat
```

---

# iostat -xz

## 90. Option

```bash
iostat -xz 1 3
```

의미:

```text
-x
→ Extended Statistics

-z
→ Activity 없는 Device 생략

1
→ 1초 간격

3
→ 3회 출력
```

---

# iostat 첫 번째 Block

## 91. 누적 평균

첫 번째 출력 Block은 부팅 이후 누적 평균 성격을 가진다.

후속 Block은 지정한 interval 동안의 상태를 나타낸다.

따라서 현재 I/O 부하를 볼 때 후속 Block이 중요하다.

---

# iostat CPU

## 92. %iowait

```text
%iowait
```

CPU가 I/O가 완료되기를 기다리는 시간과 관련된 비율이다.

Disk 문제가 있을 때 참고할 수 있다.

---

# r/s

## 93. Read Requests

```text
r/s
```

초당 Read 요청 수.

---

# w/s

## 94. Write Requests

```text
w/s
```

초당 Write 요청 수.

이번 Disk Write 실습:

```text
1535.64
```

까지 증가하였다.

---

# rkB/s

## 95. Read Throughput

```text
rkB/s
```

초당 읽은 Data 양.

---

# wkB/s

## 96. Write Throughput

```text
wkB/s
```

초당 쓴 Data 양.

이번 실습:

```text
782753.47 KB/s
```

이었다.

---

# await

## 97. await

I/O Request가 완료될 때까지 걸리는 평균 시간과 관련된 지표이다.

Read와 Write를 따로:

```text
r_await
w_await
```

형태로 볼 수도 있다.

---

# w_await

## 98. Write Wait Time

이번:

```text
0.69 ms
```

이었다.

---

# aqu-sz

## 99. Average Queue Size

```text
aqu-sz
```

평균 I/O Queue 크기와 관련된다.

이번:

```text
1.06
```

이었다.

---

# %util

## 100. Device Utilization

```text
%util
```

측정 구간 동안 Device가 I/O 작업을 처리하느라 바빴던 정도를 나타내는 지표이다.

이번 Write 부하:

```text
58.51%
```

이었다.

---

# %util 해석 주의

## 101. 100% = 무조건 장애는 아니다

Storage 종류와 병렬 처리 구조에 따라 `%util` 해석은 달라질 수 있다.

따라서 Disk 병목은:

```text
%util
await
Queue
Throughput
%iowait
Application Response
```

를 함께 확인해야 한다.

---

# Disk 부하 생성

## 102. dd

이번 실습:

```bash
dd if=/dev/zero of=/tmp/iostat-test.bin bs=1M count=512 oflag=direct
```

을 반복하여 Disk Write Activity를 발생시켰다.

---

# oflag=direct

## 103. Direct I/O

```text
oflag=direct
```

는 가능한 경우 Page Cache 영향을 줄이고 Direct I/O 형태로 Write를 수행하도록 사용하였다.

Disk I/O 변화를 관찰하기 위한 실습 목적이었다.

---

# Resource Troubleshooting

## 104. 서버가 느릴 때

Server가 느리다고 바로 CPU 문제라고 판단하면 안 된다.

가능한 원인:

```text
CPU
Memory
Swap
Disk I/O
Network
Application
Database
Lock
External Service
```

등이 있다.

---

# 기본 확인 순서

## 105. 1차 진단

추천 흐름:

```text
uptime
        ↓
top
        ↓
free -h
        ↓
vmstat
        ↓
ps
        ↓
iostat
        ↓
원인 Process / Resource 분석
```

---

# CPU 장애 분석

## 106. CPU가 높을 때

확인:

```bash
top
```

```bash
ps -eo pid,ppid,user,stat,pcpu,pmem,comm --sort=-pcpu | head
```

확인 항목:

```text
CPU Idle
Load Average
%CPU
us
sy
wa
PID
Process
```

---

# CPU 사용률이 높은 경우

## 107. 예

```text
CPU idle
→ 매우 낮음

특정 Process %CPU
→ 매우 높음

r
→ CPU 개수보다 지속적으로 큼
```

이라면 CPU Bottleneck을 의심할 수 있다.

하지만 실제 장애 원인까지 확인하려면 해당 Process가 왜 CPU를 사용하는지 조사해야 한다.

---

# Memory 장애 분석

## 108. Memory 부족 확인

```bash
free -h
vmstat 1 5
```

확인:

```text
available
Swap Used
si
so
```

---

# Memory Process 확인

## 109. RSS 기준

```bash
ps -eo pid,user,rss,vsz,pmem,comm --sort=-rss | head
```

많은 Physical Memory를 사용하는 Process를 확인한다.

---

# Swap이 지속될 때

## 110. si / so

```text
si / so
```

가 지속적으로 증가하면서:

```text
available Memory 낮음
System Response 느림
Disk Activity 증가
```

등이 함께 발생하면 Memory Pressure를 의심할 수 있다.

---

# Disk 장애 분석

## 111. Disk Bottleneck

```bash
iostat -xz 1 5
```

확인:

```text
%iowait
await
aqu-sz
%util
r/s
w/s
rkB/s
wkB/s
```

---

# CPU는 낮은데 Server가 느릴 때

## 112. I/O 확인

예:

```text
CPU idle 높음

하지만
%iowait 높음

Disk await 높음
Queue 증가
```

라면 CPU가 아니라 Disk I/O가 병목일 가능성이 있다.

---

# Process 종료 전 확인

## 113. 무조건 kill -9 하지 않는다

문제 Process를 찾았다고 바로:

```bash
kill -9 PID
```

하는 것은 좋은 운영 습관이 아니다.

먼저 확인:

```text
어떤 Service인지
업무 영향은 없는지
왜 Resource를 많이 쓰는지
정상 종료가 가능한지
```

를 확인한다.

일반적으로:

```text
SIGTERM
        ↓
정상 종료 대기
        ↓
필요하면 SIGKILL
```

순서가 좋다.

---

# Service Process

## 114. systemctl 고려

Process가 systemd Service에 의해 관리되고 있다면 개별 PID를 직접 죽이는 것보다:

```bash
systemctl status SERVICE
systemctl restart SERVICE
```

등 Service 단위 관리가 더 적절할 수 있다.

---

# Monitoring과 원인 분석

## 115. 수치가 높다는 것은 증상이다

예:

```text
CPU 100%
```

은 문제의 원인이 아니라 증상일 수 있다.

진짜 질문:

```text
어떤 Process가 CPU를 쓰는가?
왜 쓰는가?
정상 작업인가?
Traffic 증가인가?
무한 Loop인가?
설정 오류인가?
```

를 확인해야 한다.

---

# Baseline

## 116. 정상 상태를 알아야 한다

장애 상황만 보면 수치가 높은지 낮은지 판단하기 어렵다.

따라서 평소 정상 상태의:

```text
CPU
Memory
Load Average
Disk I/O
Process 수
```

등을 알고 있으면 장애 비교에 도움이 된다.

이번 실습에서도 부하를 발생시키기 전 Baseline을 먼저 확인하였다.

---

# 이번 실습 Baseline

## 117. 정상 상태

대략:

```text
CPU Idle
98~100%

Memory Available
약 2.5~2.6 GiB

Swap
0 B 사용

I/O Wait
0%

Disk Activity
거의 없음
```

이었다.

---

# CPU 부하 비교

## 118. CPU

```text
정상
CPU idle 약 98~100%
        ↓
yes 실행
        ↓
Process CPU 약 100%
        ↓
System idle 약 56~58%
        ↓
Process 종료
        ↓
System idle 약 98%
```

---

# Memory 부하 비교

## 119. Memory

```text
정상
used 약 1.0 GiB
available 약 2.5 GiB
        ↓
512 MiB 할당
        ↓
RSS 약 519 MiB
%MEM 14.3%
        ↓
used 약 1.5 GiB
available 약 2.0 GiB
        ↓
Process 종료
        ↓
Memory 회복
```

---

# Disk I/O 비교

## 120. Disk

```text
정상
%iowait 0
Disk Activity 거의 없음
        ↓
dd Write
        ↓
w/s 증가
wkB/s 증가
%util 증가
%iowait 증가
        ↓
Write 종료
        ↓
Disk Idle
```

---

# 주요 Command 정리

## 121. Process

```bash
ps -ef
ps aux
pgrep
pstree
```

---

## 122. Process Resource

```bash
top
ps --sort=-pcpu
ps --sort=-rss
```

---

## 123. CPU

```bash
nproc
lscpu
uptime
top
vmstat
```

---

## 124. Memory

```bash
free -h
vmstat
ps
```

---

## 125. Swap

```bash
swapon --show
free -h
vmstat
```

---

## 126. Signal

```bash
kill PID
kill -15 PID
kill -9 PID
```

---

## 127. Job

```bash
jobs -l
fg %1
bg %1
```

---

## 128. Background

```bash
command &
```

---

## 129. Session 유지

```bash
nohup command &
```

---

## 130. Disk I/O

```bash
iostat -xz 1 5
```

---

# 최종 Troubleshooting 흐름

## 131. Server가 느릴 때

```text
서버 느림
   ↓
uptime
   ↓
Load Average 확인
   ↓
top
   ↓
CPU / Memory 확인
   ↓
Process 확인
   ↓
free -h
   ↓
available / Swap 확인
   ↓
vmstat
   ↓
r / b / si / so / us / sy / id / wa 확인
   ↓
iostat
   ↓
Disk I/O 확인
   ↓
원인 PID / Service 확인
   ↓
Log 확인
   ↓
원인 분석
   ↓
필요한 조치
```

---

# 핵심 정리

## 132. Process

```text
Process
=
실행 중인 Program
```

Kernel은 PID로 Process를 관리한다.

---

## 133. PID / PPID

```text
PID
→ 현재 Process ID

PPID
→ Parent Process ID
```

---

## 134. Process 상태

```text
R
→ Running / Runnable

S
→ Sleeping

Z
→ Zombie
```

---

## 135. Zombie

```text
Child 종료
        ↓
Parent가 종료 정보 미회수
        ↓
Zombie
```

Zombie는 이미 실행이 끝난 Process이다.

---

## 136. CPU

```text
%CPU
→ Process CPU 사용량

us
→ User CPU

sy
→ System CPU

id
→ Idle CPU

wa
→ I/O Wait
```

---

## 137. Load Average

```text
1분
5분
15분
```

평균 System Load를 나타낸다.

CPU 사용률 자체와 같은 값은 아니다.

---

## 138. Memory

```text
free
→ 완전히 사용되지 않는 Memory

buff/cache
→ Cache 등에 사용

available
→ 새 작업에 사용할 수 있다고 추정되는 Memory
```

---

## 139. RSS / VSZ

```text
RSS
→ 실제 RAM에 올라와 있는 Memory

VSZ
→ Virtual Address Space
```

---

## 140. Swap

```text
Swap
=
RAM이 부족할 때 사용할 수 있는 Disk 기반 Memory 영역
```

RAM보다 느리기 때문에 과도한 Swap은 성능 저하를 유발할 수 있다.

---

## 141. Job Control

```text
&
→ Background

fg
→ Foreground

Ctrl + Z
→ 일시 정지

bg
→ Background에서 다시 실행

Ctrl + C
→ Interrupt / 일반적으로 종료
```

---

## 142. nohup

```text
nohup
=
Terminal / SSH Session 종료 후에도
Process를 계속 실행하는 데 사용할 수 있는 방법
```

---

## 143. vmstat

```text
vmstat
=
Process + Memory + Swap + I/O + CPU
종합 상태 확인
```

---

## 144. iostat

```text
iostat
=
CPU + Disk I/O 상태 확인
```

주요 지표:

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

# 최종 이해

## 145. Resource Monitoring 핵심

Server Resource Monitoring에서 중요한 것은 단순히:

```text
CPU가 몇 %
Memory가 몇 %
```

만 보는 것이 아니다.

실제 분석은:

```text
현재 증상 확인
        ↓
정상 Baseline과 비교
        ↓
CPU 확인
        ↓
Memory 확인
        ↓
Swap 확인
        ↓
Disk I/O 확인
        ↓
문제 Process 확인
        ↓
PID / Service 확인
        ↓
Log 확인
        ↓
원인 분석
        ↓
조치
```

순서로 진행한다.

이번 실습에서는 직접 CPU, Memory, Disk I/O 부하를 발생시켜 다음 변화를 확인하였다.

```text
CPU 부하
→ %CPU / us / sy / id 변화

Memory 부하
→ RSS / %MEM / available 변화

Process 상태
→ R / S / Z 확인

Job
→ Background / Foreground / Stop / Resume

Session 종료
→ nohup Process 유지

Disk 부하
→ w/s / wkB/s / await / %util 변화
```

따라서 단순 Command 사용법이 아니라 실제 Server Resource 상태를 관찰하고 문제 위치를 판단하는 기본적인 Monitoring / Troubleshooting 흐름을 실습하였다.
