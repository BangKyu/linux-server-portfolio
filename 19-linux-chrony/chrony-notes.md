# Chrony / NTP 이론 정리

## 1. NTP란?

NTP는 Network Time Protocol의 약자이다.

Network를 통해 여러 System의 시간을 서로 동기화하기 위한 Protocol이다.

예:

```text
외부 NTP Server
        ↓
Server-A
        ↓
Client-L
```

여러 Server의 시간이 서로 다르면 다음과 같은 문제가 발생할 수 있다.

```text
Log 시간 불일치
인증 문제
장애 분석 어려움
분산 System 시간 불일치
작업 실행 시간 혼란
```

따라서 Server 환경에서는 시간 동기화가 중요하다.

---

# Chrony

## 2. Chrony란?

Chrony는 Linux에서 NTP를 이용하여 System 시간을 동기화하는 Software이다.

Rocky Linux에서는 Chrony를 이용하여 NTP Client 또는 NTP Server 역할을 구성할 수 있다.

이번 실습 Package:

```text
chrony
```

Service:

```text
chronyd
```

관리 명령:

```text
chronyc
```

---

## 3. chronyd란?

`chronyd`는 실제 시간 동기화를 수행하는 Daemon이다.

역할:

```text
NTP Server와 통신
        ↓
시간 차이 측정
        ↓
System Clock 조정
```

Server로 구성할 경우에는 다른 Client의 NTP 요청에도 응답할 수 있다.

Service 확인:

```bash
systemctl is-active chronyd
systemctl is-enabled chronyd
```

---

## 4. chronyc란?

`chronyc`는 실행 중인 `chronyd`를 확인하고 관리하는 Command Line Client이다.

대표 명령:

```bash
chronyc sources -v
chronyc tracking
chronyc clients
chronyc burst 4/4
```

즉:

```text
chronyd
→ 실제 시간 동기화 Daemon

chronyc
→ chronyd 상태 확인 / 관리 명령
```

이다.

---

# 이번 실습 구조

## 5. 전체 구성

```text
External NTP Servers
        |
        | NTP
        v
Server-A
192.168.111.100
Stratum 3
        |
        | UDP 123
        v
Client-L
192.168.111.150
Stratum 4
```

Server-A:

```text
외부 NTP Server로부터 시간 동기화
+
내부 Client에게 시간 제공
```

Client-L:

```text
Server-A를 시간 Source로 사용
```

---

# 기존 구조

## 6. 실습 전 상태

실습 전에는 Server-A와 Client-L이 각각 외부 NTP Server를 직접 사용하고 있었다.

```text
External NTP
     ↓
Server-A
```

```text
External NTP
     ↓
Client-L
```

Client-L 초기 Source:

```text
^* 211.108.117.211
```

즉 Client-L이 외부 NTP Server를 직접 사용하고 있었다.

---

# 변경된 구조

## 7. 실습 후 상태

```text
External NTP
        ↓
Server-A
        ↓
Client-L
```

Client-L은 외부 NTP Server를 직접 사용하지 않고 내부 Server-A를 시간 Source로 사용하도록 변경하였다.

---

# /etc/chrony.conf

## 8. Chrony 설정 파일

Rocky Linux에서 주요 Chrony 설정 파일:

```text
/etc/chrony.conf
```

이다.

이번 실습에서는 Server와 Client 양쪽의 이 파일을 수정하였다.

---

# pool

## 9. pool Directive

기존 설정:

```conf
pool 2.rocky.pool.ntp.org iburst
```

`pool`은 여러 NTP Server 중 사용할 수 있는 Source를 찾도록 구성할 때 사용한다.

```text
NTP Pool
        ↓
여러 NTP Server 후보
        ↓
chronyd가 적절한 Source 선택
```

---

# server

## 10. server Directive

Client-L:

```conf
server 192.168.111.100 iburst
```

의미:

```text
192.168.111.100
→ NTP Server

Client-L
→ 해당 Server를 시간 Source로 사용
```

이번 실습에서는 Client-L의 Source를 명확히 Server-A 하나로 지정하였다.

---

# iburst

## 11. iburst란?

```text
iburst
```

는 NTP Source와 처음 통신을 시작할 때 초기 Sample을 빠르게 수집하도록 하는 Option이다.

즉:

```text
chronyd 시작
        ↓
초기 NTP Sample 빠르게 수집
        ↓
동기화 판단을 더 빠르게 수행
```

하는 데 도움이 된다.

---

# sourcedir

## 12. sourcedir /run/chrony-dhcp

기존 설정:

```conf
sourcedir /run/chrony-dhcp
```

은 DHCP 등을 통해 전달된 NTP Source 정보를 사용할 수 있도록 하는 설정이다.

이번 실습에서는 Client-L이 Server-A만 사용하도록 명확히 하기 위해:

```conf
#sourcedir /run/chrony-dhcp
```

로 주석 처리하였다.

---

# allow

## 13. allow Directive

Server-A에 추가:

```conf
allow 192.168.111.0/24
```

의미:

```text
192.168.111.0/24 Network의 Client가
Server-A의 chronyd에 NTP 요청 가능
```

이다.

이 설정이 없으면 Server-A가 자신의 시간을 동기화하는 Client 역할은 가능하지만, 다른 Host에게 시간을 제공하는 Server 역할은 제한될 수 있다.

---

## 14. NTP Client와 Server 역할

Server-A는 동시에 두 가지 역할을 수행한다.

```text
External NTP Server 기준

Server-A
→ NTP Client
```

그리고:

```text
Client-L 기준

Server-A
→ NTP Server
```

즉 하나의 Host가 상황에 따라 NTP Client이면서 동시에 NTP Server가 될 수 있다.

---

# UDP 123

## 15. NTP Port

NTP의 기본 Port:

```text
UDP 123
```

이다.

Server-A에서:

```bash
ss -lunp | grep ':123'
```

실제 결과:

```text
UNCONN 0 0 0.0.0.0:123 0.0.0.0:* users:(("chronyd",pid=8827,fd=7))
```

`chronyd`가 UDP 123에서 Client 요청을 받을 수 있는 상태임을 확인하였다.

---

# Firewall

## 16. NTP Firewall 설정

Server-A:

```bash
firewall-cmd --permanent --add-service=ntp
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-services
```

실제:

```text
ntp
```

가 포함되었다.

Chrony 설정이 정상이어도 Firewall에서 UDP 123이 차단되면 Client가 Server-A와 NTP 통신을 할 수 없다.

---

# chronyc sources

## 17. sources 명령

```bash
chronyc sources -v
```

현재 chronyd가 알고 있는 NTP Source 상태를 확인한다.

확인 가능:

```text
Source 종류
현재 선택 상태
Stratum
Polling
Reach
마지막 응답 시간
Offset
오차 범위
```

---

# Source Mode 기호

## 18. ^

```text
^
```

는 Source가 NTP Server임을 의미한다.

예:

```text
^* www.bangkyu.com
```

에서 `^`는:

```text
NTP Server Source
```

이다.

---

## 19. =

```text
=
```

는 Peer Mode Source를 나타낸다.

---

## 20. #

```text
#
```

는 Local Reference Clock을 나타낸다.

---

# Source State 기호

## 21. *

가장 중요한 상태:

```text
*
```

는 현재 System Clock을 동기화하는 데 실제로 선택된 Source이다.

이번 Client-L:

```text
^* www.bangkyu.com
```

즉:

```text
^
→ NTP Server

*
→ 현재 실제 동기화 Source
```

이다.

---

## 22. +

```text
+
```

는 현재 선택된 Source와 함께 시간 계산에 사용할 수 있는 Source이다.

---

## 23. -

```text
-
```

는 사용할 수 있는 Source이지만 현재 시간 계산에서는 선택되지 않은 상태를 의미한다.

---

## 24. ?

```text
?
```

는 현재 사용할 수 없는 Source이다.

초기 통신 전이나 Server 응답이 없을 때 나타날 수 있다.

---

## 25. x

```text
x
```

는 해당 Source가 잘못된 시간을 제공하는 것으로 판단될 가능성이 있는 상태이다.

---

## 26. ~

```text
~
```

는 시간 변화가 너무 불안정한 Source로 판단될 때 나타날 수 있다.

---

# Client-L 실제 Source

## 27. 실제 결과

Client-L:

```bash
chronyc sources -v
```

실제 결과:

```text
^* www.bangkyu.com               3   6    37    56  +7851ns[ -138us] +/- 6106us
```

현재 실제 시간 Source:

```text
www.bangkyu.com
```

이다.

---

# 왜 IP 대신 Domain이 보이는가?

## 28. DNS 이름 변환

Client-L의 설정은:

```conf
server 192.168.111.100 iburst
```

였지만 결과에서는:

```text
www.bangkyu.com
```

으로 표시되었다.

현재 DNS:

```text
www.bangkyu.com
→ 192.168.111.100
```

이므로 Chrony가 IP Address를 Hostname으로 표시한 것이다.

즉:

```text
www.bangkyu.com
=
192.168.111.100
=
Server-A
```

이다.

---

# chronyc tracking

## 29. tracking 명령

```bash
chronyc tracking
```

현재 System Clock의 전체적인 동기화 상태를 확인한다.

대표 항목:

```text
Reference ID
Stratum
Ref time
System time
Last offset
RMS offset
Frequency
Root delay
Root dispersion
Leap status
```

---

# Reference ID

## 30. Reference ID란?

현재 System이 기준으로 사용하는 NTP Source를 식별하는 값이다.

Client-L:

```text
Reference ID : C0A86F64 (www.bangkyu.com)
```

---

## 31. C0A86F64

IPv4 Source인 경우 Reference ID가 IPv4 주소를 16진수 형태로 나타낼 수 있다.

```text
C0 → 192
A8 → 168
6F → 111
64 → 100
```

따라서:

```text
C0A86F64
→ 192.168.111.100
```

이다.

즉 Client-L의 Reference Server는 Server-A이다.

---

# Stratum

## 32. Stratum이란?

Stratum은 NTP에서 시간 기준 Source와 얼마나 가까운 계층인지 나타내는 값이다.

개념적으로:

```text
정확한 기준 Clock
        ↓
상위 NTP Server
        ↓
하위 NTP Server
        ↓
Client
```

형태로 계층이 형성된다.

숫자가 단순히 "정확도 점수"라는 의미는 아니고 시간 Source와의 계층적 거리를 나타낸다.

---

## 33. 이번 실습 Stratum

Server-A:

```text
Stratum 3
```

Client-L:

```text
Stratum 4
```

구조:

```text
External NTP Source
Stratum 2
        ↓
Server-A
Stratum 3
        ↓
Client-L
Stratum 4
```

Client-L이 Server-A를 Source로 사용하면서 Stratum이 한 단계 증가하였다.

---

# Ref time

## 34. Ref time

```text
Ref time
```

은 현재 Reference Source로부터 마지막으로 유효한 시간 정보를 받은 시점을 나타낸다.

Client-L 실제:

```text
Ref time (UTC) : Mon Sep 14 08:10:18 2026
```

---

# System time

## 35. System time

Client-L:

```text
System time : 0.000039654 seconds slow of NTP time
```

의미:

```text
현재 System Clock이
NTP 기준 시간보다
약 0.000039654초 느림
```

이라는 의미이다.

시간 차이가 매우 작은 상태였다.

---

# Last offset

## 36. Last offset

```text
Last offset
```

은 최근 NTP Update에서 측정된 시간 차이를 나타낸다.

Client-L:

```text
Last offset : -0.000145764 seconds
```

---

# RMS offset

## 37. RMS offset

```text
RMS offset
```

은 일정 기간 동안 측정된 Offset의 크기를 통계적으로 나타내는 값이다.

값이 작을수록 일반적으로 Clock이 NTP Source에 가깝게 유지되고 있다고 볼 수 있다.

---

# Frequency

## 38. Frequency

Computer Clock은 완전히 정확한 속도로 흐르지 않을 수 있다.

Chrony는 Clock이 얼마나 빠르거나 느리게 흐르는지 학습하여 보정한다.

```text
Frequency
```

는 이러한 Clock 보정과 관련된 값을 나타낸다.

단위:

```text
ppm
```

이다.

---

# Root delay

## 39. Root delay

```text
Root delay
```

는 현재 System에서 최상위 기준 Clock까지의 Network Delay와 관련된 값이다.

---

# Root dispersion

## 40. Root dispersion

```text
Root dispersion
```

은 기준 시간에 대한 예상 최대 오차와 관련된 값이다.

---

# Leap status

## 41. Leap status

이번 Client-L:

```text
Leap status : Normal
```

이었다.

정상적인 시간 동기화 상태인지 확인할 때 참고할 수 있다.

---

# timedatectl

## 42. timedatectl이란?

System의 시간 및 Time Zone 관련 상태를 확인하고 설정하는 명령이다.

```bash
timedatectl
```

이번 Client-L 결과:

```text
Local time: 월 2026-09-14 17:12:34 KST
Universal time: 월 2026-09-14 08:12:34 UTC
RTC time: 월 2026-09-14 08:12:34
Time zone: Asia/Seoul (KST, +0900)
System clock synchronized: yes
NTP service: active
RTC in local TZ: no
```

---

# Local Time

## 43. Local Time

현재 System에 설정된 Time Zone이 적용된 시간이다.

이번 환경:

```text
Asia/Seoul
KST
UTC +09:00
```

---

# UTC

## 44. Universal Time

```text
Universal time
```

은 UTC 기준 시간이다.

이번 결과:

```text
Local
17:12

UTC
08:12
```

9시간 차이가 난다.

```text
KST = UTC + 9
```

이기 때문이다.

---

# RTC

## 45. RTC란?

RTC는 Real-Time Clock의 약자이다.

System이 꺼져 있어도 시간을 유지하는 Hardware Clock이다.

이번 결과:

```text
RTC in local TZ: no
```

이므로 RTC를 Local Time이 아닌 UTC 기준으로 사용하는 구성이다.

---

# System Clock

## 46. System Clock이란?

운영체제가 동작하면서 사용하는 시간이다.

```text
Kernel / OS
→ System Clock 사용
```

Application Log, File Timestamp, Process 등이 이 시간을 기준으로 동작한다.

---

# RTC와 System Clock

## 47. 관계

```text
RTC
→ Hardware Clock

System Clock
→ OS가 사용하는 Clock

Chrony
→ Network 기준으로 System Clock 조정
```

필요에 따라 System Clock과 RTC의 관계도 유지된다.

---

# System clock synchronized

## 48. synchronized: yes

```text
System clock synchronized: yes
```

는 System Clock이 정상적으로 시간 동기화 상태에 있음을 나타낸다.

---

# NTP service

## 49. NTP service: active

```text
NTP service: active
```

NTP 시간 동기화 Service가 활성화되어 있음을 의미한다.

이번 환경에서는 `chronyd`가 해당 역할을 수행하였다.

---

# chronyc clients

## 50. NTP Server에서 Client 확인

Server-A:

```bash
chronyc clients
```

실제:

```text
Hostname                      NTP   Drop Int IntL Last     Cmd   Drop Int  Last
===============================================================================
192.168.111.150                 5      0   4   -    18       0      0   -     -
```

Server-A가 Client-L의 NTP 요청을 실제로 받고 있음을 확인하였다.

---

## 51. NTP

```text
NTP 5
```

Client-L에서 NTP 요청이 들어온 기록이 있음을 보여준다.

---

## 52. Drop

```text
Drop 0
```

해당 Client의 NTP 요청 중 Drop된 요청이 없음을 나타낸다.

---

## 53. Last

```text
Last 18
```

최근 Client 요청을 받은 이후 경과 시간을 나타낸다.

이번 출력에서는 약 18초 전에 Client-L의 요청을 받은 상태였다.

---

# Client와 Server 양쪽 검증

## 54. Client-L 관점

```bash
chronyc sources -v
```

결과:

```text
^* www.bangkyu.com
```

의미:

```text
Server-A를 실제 NTP Source로 사용
```

---

## 55. Server-A 관점

```bash
chronyc clients
```

결과:

```text
192.168.111.150
```

의미:

```text
Client-L이 실제로 NTP 요청 전송
```

---

## 56. 양쪽 검증이 중요한 이유

Client에서만 보면:

```text
어떤 Server를 바라보는지
```

확인할 수 있다.

Server에서도 확인하면:

```text
실제 Client 요청이 Server까지 도착했는지
```

확인할 수 있다.

즉:

```text
Client Source 확인
        +
Server Client 확인
        =
End-to-End NTP 검증
```

이다.

---

# chronyc burst

## 57. burst란?

```bash
chronyc burst 4/4
```

는 NTP Source에 여러 측정을 빠르게 수행하도록 요청할 때 사용할 수 있다.

이번 실습에서는 Client-L의 Source를 변경한 뒤 초기 Sample을 빠르게 확보하는 데 사용하였다.

---

# makestep

## 58. makestep

기본 설정:

```conf
makestep 1.0 3
```

Chrony가 초기 동기화 과정에서 System Clock 차이가 큰 경우 점진적인 보정 대신 Clock을 빠르게 조정할 수 있도록 하는 설정이다.

여기서:

```text
1.0
→ 시간 차이 기준

3
→ chronyd 시작 후 초기 Update 횟수 범위
```

와 관련된다.

---

# 점진적 시간 보정과 Step

## 59. Slew

Clock 차이가 작을 때 시간을 조금씩 빠르게 또는 느리게 조절하여 자연스럽게 맞추는 방식이다.

```text
현재 시간
        ↓
조금씩 보정
        ↓
NTP 시간에 접근
```

---

## 60. Step

시간 차이가 클 경우 Clock 값을 한 번에 크게 변경하는 방식이다.

```text
현재 시간
        ↓
즉시 조정
        ↓
NTP 기준 시간
```

`makestep`은 초기 동기화 과정에서 이런 동작을 허용하는 설정이다.

---

# rtcsync

## 61. rtcsync

기본 설정:

```conf
rtcsync
```

System Clock이 동기화된 상태를 Kernel에 전달하고 RTC와의 시간 관리에 활용되도록 하는 설정이다.

---

# driftfile

## 62. driftfile

```conf
driftfile /var/lib/chrony/drift
```

Chrony가 System Clock의 주파수 오차 정보를 저장하는 File이다.

System 재시작 후에도 이전에 학습한 Clock 특성을 활용할 수 있도록 한다.

---

# logdir

## 63. logdir

```conf
logdir /var/log/chrony
```

Chrony 관련 Log File을 저장할 Directory를 지정한다.

---

# NTP와 DNS

## 64. DNS 이름 표시 주의

`chronyc sources` 출력은 Source를 IP가 아니라 DNS Name으로 표시할 수 있다.

이번 실습:

```text
192.168.111.100
```

이:

```text
www.bangkyu.com
```

으로 표시되었다.

따라서 Source 확인 시 Hostname만 보고 다른 Server라고 판단하면 안 된다.

필요하면:

```bash
getent hosts www.bangkyu.com
```

또는:

```bash
nslookup www.bangkyu.com
```

으로 IP를 확인할 수 있다.

---

# 시간 동기화가 중요한 이유

## 65. Log 분석

예:

```text
Server-A Log
17:10

Client-L Log
17:15
```

시간이 맞지 않으면 장애 발생 순서를 정확하게 판단하기 어렵다.

---

## 66. 인증

일부 인증 Protocol과 보안 System은 Server와 Client의 시간 차이가 너무 크면 인증에 문제가 발생할 수 있다.

---

## 67. 분산 환경

여러 Server가 함께 동작하는 환경에서는 모든 Server가 비슷한 시간을 유지하는 것이 중요하다.

예:

```text
Web Server
Database Server
Storage Server
Monitoring Server
```

---

## 68. Scheduled Job

Cron과 같은 예약 작업도 System Clock을 기준으로 실행되므로 정확한 시간이 중요하다.

---

# Troubleshooting

## 69. Client가 Server-A를 Source로 선택하지 않을 때

Client-L:

```bash
chronyc sources -v
```

확인.

다음처럼:

```text
^? 192.168.111.100
```

이라면 Source를 아직 사용할 수 없는 상태일 수 있다.

---

## 70. Network 확인

```bash
ping -c 3 192.168.111.100
```

Server-A와 기본 Network 통신이 가능한지 확인한다.

---

## 71. chronyd 확인

Server-A:

```bash
systemctl is-active chronyd
```

Client-L:

```bash
systemctl is-active chronyd
```

양쪽 모두 확인한다.

---

## 72. Server 설정 확인

```bash
grep -vE '^[[:space:]]*#|^[[:space:]]*$' /etc/chrony.conf
```

특히:

```conf
allow 192.168.111.0/24
```

가 존재하는지 확인한다.

---

## 73. Client 설정 확인

Client-L:

```bash
grep -vE '^[[:space:]]*#|^[[:space:]]*$' /etc/chrony.conf
```

확인:

```conf
server 192.168.111.100 iburst
```

---

## 74. UDP 123 확인

Server-A:

```bash
ss -lunp | grep ':123'
```

`chronyd`가 UDP 123을 사용할 수 있는 상태인지 확인한다.

---

## 75. Firewall 확인

```bash
firewall-cmd --list-services
```

확인:

```text
ntp
```

---

## 76. Source 확인

Client-L:

```bash
chronyc sources -v
```

정상 목표:

```text
^* Server-A
```

---

## 77. Tracking 확인

```bash
chronyc tracking
```

확인:

```text
Reference ID
Stratum
System time
Leap status
```

---

## 78. Server에서 Client 확인

```bash
chronyc clients
```

Client IP:

```text
192.168.111.150
```

가 나타나는지 확인한다.

---

## 79. timedatectl 확인

```bash
timedatectl
```

정상:

```text
System clock synchronized: yes
NTP service: active
```

---

# NTP 문제 확인 순서

## 80. 추천 Troubleshooting 순서

```text
1. Network 연결
        ↓
2. chronyd Service
        ↓
3. Server chrony.conf
        ↓
4. allow Network
        ↓
5. Firewall NTP
        ↓
6. UDP 123
        ↓
7. Client chrony.conf
        ↓
8. chronyc sources
        ↓
9. chronyc tracking
        ↓
10. chronyc clients
        ↓
11. timedatectl
```

---

# 주요 명령어

## 81. Chrony Package

```bash
rpm -q chrony
```

---

## 82. Service

```bash
systemctl is-active chronyd
systemctl is-enabled chronyd
```

---

## 83. Service 재시작

```bash
systemctl restart chronyd
```

---

## 84. 시간 상태

```bash
timedatectl
```

---

## 85. Source

```bash
chronyc sources -v
```

---

## 86. Tracking

```bash
chronyc tracking
```

---

## 87. Client 조회

```bash
chronyc clients
```

---

## 88. 빠른 Sample

```bash
chronyc burst 4/4
```

---

## 89. NTP Port

```bash
ss -lunp | grep ':123'
```

---

## 90. Firewall

```bash
firewall-cmd --list-services
```

---

# 핵심 설정

## 91. Server-A

```conf
allow 192.168.111.0/24
```

의미:

```text
내부 Network Client의 NTP 요청 허용
```

---

## 92. Client-L

```conf
server 192.168.111.100 iburst
```

의미:

```text
Server-A를 NTP Source로 사용
```

---

# 이번 실습 최종 검증

## 93. Server-A

```text
chronyd
→ active

Stratum
→ 3

UDP 123
→ Listen

Firewall
→ ntp 허용

chronyc clients
→ 192.168.111.150 확인
```

---

## 94. Client-L

```text
chronyd
→ active

Source
→ ^* www.bangkyu.com

Reference
→ 192.168.111.100

Stratum
→ 4

System clock synchronized
→ yes

NTP service
→ active
```

---

# 최종 구조

## 95. 전체 동작

```text
                    External NTP Source
                            |
                            | NTP
                            v
                        Server-A
                     192.168.111.100
                            |
                     chronyd Client
                            |
                        Stratum 3
                            |
                            |
              allow 192.168.111.0/24
                            |
                            |
                         UDP 123
                            |
                            | NTP
                            v
                        Client-L
                     192.168.111.150
                            |
                     chronyd Client
                            |
                        Stratum 4
                            |
                            v
                 System Clock Synchronized
```

---

# 핵심 정리

## 96. NTP

```text
NTP
=
Network를 통해
System 시간을 동기화하는 Protocol
```

---

## 97. Chrony

```text
Chrony
=
Linux에서 NTP 시간 동기화를 수행하는 Software
```

---

## 98. chronyd / chronyc

```text
chronyd
→ 실제 시간 동기화 Daemon

chronyc
→ chronyd 상태 조회 및 관리 Client
```

---

## 99. Server 역할

Server-A:

```text
외부 NTP를 기준으로 자신의 시간 동기화
+
내부 Client에게 시간 제공
```

---

## 100. Client 역할

Client-L:

```text
Server-A를 기준으로 시간 동기화
```

---

## 101. Stratum

```text
Server-A
Stratum 3

        ↓

Client-L
Stratum 4
```

NTP Server를 한 단계 더 거치면서 Stratum 값이 증가하였다.

---

## 102. Source 선택

```text
^*
```

의미:

```text
^
→ NTP Server

*
→ 현재 실제 선택된 시간 Source
```

---

## 103. UDP 123

```text
NTP
→ UDP 123
```

Server-A의 `chronyd`가 해당 Port에서 Client 요청을 받을 수 있음을 확인하였다.

---

## 104. 정상 동기화 판단

```text
chronyc sources -v
→ ^*

chronyc tracking
→ 올바른 Reference
→ 정상 Stratum
→ Leap status Normal

timedatectl
→ System clock synchronized: yes

chronyc clients
→ 실제 Client IP 확인
```

이 네 가지를 함께 확인하면 NTP Server / Client 구조를 보다 명확하게 검증할 수 있다.

---

# 최종 이해

## 105. 실습 핵심 흐름

```text
External NTP
        ↓
Server-A chronyd
        ↓
Server-A 시간 동기화
        ↓
allow 192.168.111.0/24
        ↓
UDP 123
        ↓
Client-L chronyd
        ↓
Server-A를 Source로 선택
        ↓
Client-L 시간 동기화
```

이번 실습에서는 Server-A가 외부 NTP Source와 시간을 동기화하면서 내부 Network의 NTP Server 역할도 수행하도록 구성하였다.

Client-L은 외부 NTP Server를 직접 사용하지 않고 Server-A만 NTP Source로 사용하도록 변경하였다.

Client-L의 `chronyc sources -v`에서:

```text
^* www.bangkyu.com
```

을 확인하여 Server-A가 현재 실제 동기화 Source로 선택된 것을 확인하였다.

또한 Client-L의 `chronyc tracking`에서:

```text
Reference ID
→ Server-A

Stratum
→ 4
```

를 확인하였고, Server-A에서는:

```bash
chronyc clients
```

를 통해:

```text
192.168.111.150
```

Client-L의 실제 NTP 요청을 확인하였다.

마지막으로:

```text
System clock synchronized: yes
NTP service: active
```

상태까지 확인하여 Server-A → Client-L 시간 동기화 전체 흐름을 검증하였다.
